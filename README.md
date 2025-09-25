# IntelFreq
A collection of MSR value heuristics for Intel Macs. 

### Brief
Intel Macs, especially newer ones with T1/T2 chips, have complex MSR values that differ from Intel's standard MSR for the rest of the market. This repository is a collection of heuristics gathered from personal experience playing around with these MSRs. I used [VoltageShift](https://github.com/sicreative/voltageshift) to modify my MSRs. 

### 1. Forcing a specific range of CPU frequencies
**Inspiration**: When Low Power Mode is turned on in an Intel Mac, one of its effects is changing the maximum CPU frequency to 2.6GHz (regardless of Mac model). This is hot-swappable, meaning the user does not have to restart their system for the action to take in place. 

Upon further research, this is achievable with the `0x774` module from the CPU's MSR that allows for efficient hot-swappable frequency commands. You can take a look at your current frequency by running this command using VoltageShift: 

```
./voltageshift read 0x774
```

The format of the output is in binary (0s and 1s), but translated into hexadecimal (there are many online), it roughly looks like: 

```
90 XX YY ZZ
```
- `90`: An arbitrary 2-digit number that does not seem to affect anything. 
- `XX`: The currently requested frequency by the system (constantly changing).
- `YY`: The maximum allowed frequency. In Low Power Mode, this is always `1A` (2600MHz). 
- `ZZ`: The minimum allowed frequency. For all Macs, this is 04 (400MHz). 

For the frequency values, it's important to note that they are hex multipliers. Intel CPUs typically have a base clock multiplier of `100MHz`. So to get the MHz frequency you are enforcing, multiply the value in decimal form by 100. 

So for a value like `1A`, it translates to `26` in decimal, and `26*100 = 2600MHz`. You will need to convert it to the hex multiplier when writing this data. 

The system is constantly enforcing this value. Based on past experience, it seems to enforce the current CPU frequency (`XX`) through this protocol, so it always writes to this slot. In some 2012 WWDC session, Apple mentioned the system adjusts this value once every 150ms. 

So to enforce your own custom frequencies and make sure it stays there, you have to put it in a while true loop like such: 

```
while true; do
    ./voltageshift write 0x774 0x90101A04
done
```

This is a bit more of a hack, because it takes up 100% of a core. But if we believe Apple's claim of the 150ms interval frequency change, the while loop has to execute more than 6 times per second in order to keep up and exceed the system's enforcment, which usually is the case for modern Intel Macs that have better single-core performance. 

### 2. More granular control over power limits
**Inspiration**: We all know that VoltageShift can change the PL1 and PL2 limits (by `./voltageshift power x y`). Underneath the hood, it uses the `0x610` module. However, that module also changes some other critical things not exposed by the VoltageShift API. 

If you take a look at the result by running `./voltageshift read 0x610` and convert it to hex, it roughly looks like this: 

```
438 3E8 00 DD 8 320
```

> - `438`: CPU/GPU throttle priority.
>     - Setting the middle value (3 in this case) to an odd number will prioritize the GPU's power. When a task needs the GPU and there isn't enough power budget (likely due to user configuration), the system will always prioritize throttling down the CPU to give the GPU more budget. This is the default on Macs. The CPU will only stop throttling once its minimum allowed frequency is reached (typically 0.8 by the SMC, or 0.4 which is a value overwritten by macOS). Only then will the GPU start throttling.
>     - Setting the middle value to an even number will prioritize the CPU a bit more. When there is a limited power budget, the CPU will throttle down first but only to the extent of the base frequency. Setting an even number here (like 2) feels more reasonable as it gives your CPU a bit more leeway instead of choosing to max out the GPU. Weird that Apple didn't choose this. 
> - `3E8`: PL1 (for all Intel Macs, it's set at 125W)
>     - To convert from a wattage of your preference, multiply by 8 and convert to hex.
>     - Take the default example: `125W * 8 = 1000`, and `1000` in hex is `3e8`. 
>     - This is always 3 digits, meaning you can't exceed an integer wattage limit of 4095. 
> - `00`: Some arbitrary number
> - `DD`: Tau value (for all Intel Macs)
> - `8`: Another arbitrary number
> - `320`: PL2 (for all Intel Macs, it's set at 100W)

Note that this is different from Intel's official 64-bit MSR specification, meaning the T2 is definitely acting as a proxy for enhanced security. Then again, writing an invalid value to this slot will crash the system here so it's unclear what the proxy is doing exactly. 

It's weird to see all Macs having the exact same power limits and same Tau values without any performance tuning, but the results from many MacBooks (Pro 16", Pro 13", Air) all point to the same 125/100W limit and DD. 

For those unfamiliar, the SMC will allow CPU to go to PL1 and stay there, wait for (Tau value) seconds, and then revert to PL2 for the rest of the current session. Most of the time, a session changes only when there is a shift in CPU utilization. 

The Tau value is the hardest to crack here. When presented with the Mac's `0x606` reference table (which tells us the unit for time, power, and energy measurement), the Tau value completely doesn't make sense even with the provided time unit. When setting this value to `29` or `99`, it appears to be able to boost at PL1 indefinitely; when setting to `49` or `90`, it doesn't go to PL1 at all. It's likely not intended to be reverse engineered. 


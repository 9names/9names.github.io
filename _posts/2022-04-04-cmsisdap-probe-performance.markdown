---
layout: post
title:  "CMSIS-DAP debug probe performance"
date:   2022-04-07 20:07:43 +1000
categories: embedded rust debug
---


---
## CMSIS-DAP debug probe performance
---
```
```

There are quite a few [CMSIS-DAP](https://os.mbed.com/handbook/CMSIS-DAP) USB debug probes in the wild, and a lot of other devices can be converted into CMSIS-DAP debug probes using various firmware projects.

Most of these derive a lot of the core debugger functionality from [ARM](https://github.com/ARM-software/CMSIS_5) [sources](https://github.com/ARMmbed/DAPLink), though there [are exceptions](https://github.com/probe-rs/hs-probe-firmware).

These probes typically do not make any performance claims as universal tooling support and easy OS compatibility is a higher priority, but that does make choosing or recommending one over the other more difficult than it needs to be.

I could not find any benchmarks of these anywhere, but I do have a small collection of probes and other embedded hardware that can run the open probe firmwares - so I spent a few days running some benchmarks and collecting data!

```
Note: This post has been edited since original posting
* 2022-04-18 - rustyprobe was slower than expected because the firmware tested
was modified to only enumerate as cmsisdap-v1
* added rust-dap and a non-turbo firmware build of hs-probe to the dataset
* added a link to the data in CSV form at the bottom of the page
```

---
### Hardware
---
```
```
#### Test system
All tests were performed on a Ryzen 3600 running Ubuntu 20.04 with Linux kernel 5.16

USB performance can vary greatly between Operating Systems - these results may not match yours.


#### Probes
The commercial probes I used for testing were:
- [LPC-Link2](https://www.embeddedartists.com/products/lpc-link2/)
- [MCU-Link](https://www.nxp.com/design/microcontrollers-developer-resources/mcu-link-debug-probe:MCU-LINK) (with both official firmare and the latest DAPLink release)
- [J-Link EDU Mini](https://www.segger.com/products/debug-probes/j-link/models/j-link-edu-mini/) (in J-Link mode for comparison to CMSIS-DAP)
- STLink v3 (onboard debugger for [Nucleo-H743ZI2](https://os.mbed.com/platforms/ST-Nucleo-H743ZI2/)) (STLink mode for comparison to CMSIS-DAP)

Dev boards I also tested for performance:
- [stm32f411 USB-C pill](https://therealprof.github.io/blog/usb-c-pill-part1/) (ARM WeAct CMSIS-DAP firmware)
- Raspberry Pi Pico ([DapperMime](https://github.com/majbthrd/DapperMime), [RustyProbe](https://github.com/probe-rs/rusty-probe) and [rust-dap](https://github.com/ciniml/rust-dap) firmware)
- STLink V2 clone (STLink mode, latest firmware, for comparison to CMSIS-DAP)
- stm32f103 blue pill [DAP42](https://github.com/devanlai/dap42)

I also tested the [BBC Microbit V2](https://microbit.org/new-microbit/) with its onboard debugger, as this board is one of the [recommended boards](https://docs.rust-embedded.org/discovery/microbit/index.html) for folks new to Embedded Rust.

---
### Targets
---
```
```
I tested each of the above external probes against the following Microcontrollers:
- RP2040
- STM32H743ZI
- ATSAMD51

The speeds achieved for each target were identical (as you might expect) so I will only show graphs using STM32H7 as a target, since that was supported by the most probes.

The MicroBit V2 debugger was tested against NRF52833 (the MCU on the Microbit V2), but is listed in the graphs next to everything else to keep things simple.

### Software
For testing I used the [benchmark example](https://github.com/probe-rs/probe-rs/blob/master/probe-rs/examples/benchmark.rs)  from the [probe-rs](https://github.com/probe-rs/probe-rs) repo.

The original version [only lets you choose a few SWD frequencies](https://github.com/probe-rs/probe-rs/blob/6140a0d05161fb4424e934c02beb78225ba6ff15/probe-rs/examples/benchmark.rs#L66) - I removed this check so I could get a few more data points.

The list of frequencies tested (in KHz):
```100, 200, 300, 600, 1200, 2400, 4800, 10000, 20000, 40000, 80000, 100000, 200000```

Note that probes are free to choose their own frequency based on these requests, so sometimes a probe will perform better/worse than you would expect at a particular frequency.

Sometimes instead of limiting the frequency the probe rejected it or could not connect at this frequency. At these points the graph will show no data.


The RTT tests were done with firmware that effectively boils down to
{% highlight rust %}
loop {
    rprint!("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\r\n");
}
{% endhighlight %}

This may or may not be representative of what is in your code.

RTT throughput is measured with a *very* basic program and should not be considered accurate, but it should be good enough to provide *relative* performance.
[https://github.com/9names/stdoutbytecount](https://github.com/9names/stdoutbytecount)

---
### Results
---
```
```

No probe achieved greater performance at a requested speed above 40MHz, so graphs are plotted up to 80MHz.

Each graph has either write or read performance, since they are measured seperately and are often different.

#### All probes
![stm32_write_perf.svg](/assets/2022-04-04/stm32_write_perf.svg)
![stm32_read_perf.svg](/assets/2022-04-04/stm32_read_perf.svg)

---
```
```
#### Full-speed probes

It's very hard to see the performance of the USB Full Speed probes when plotted against the high-speed ones.

Here are some graphs with the USB High Speed probes removed.

Write performance for USB-C pill and rustyprobe are identical, so they overlap perfectly. rustyprobe is slightly slower on read at lower frequencies.

dap42 *always* uses the same frequency (internally it's hardcoded to 10Mhz SWD) so performance for that probe is always the same regardless of requested frequency.

rust-dap also runs at a fixed frequency.

CMSIS-DAP V2 firmware is roughtly twice as fast as CMSIS-DAP V1 firmware on USB Full Speed

![stm32_write_perf_fullspeed.svg](/assets/2022-04-04/stm32_write_perf_fullspeed.svg)
![stm32_read_perf_fullspeed.svg](/assets/2022-04-04/stm32_read_perf_fullspeed.svg)

---
```
```
#### Comparison against non-CMSIS-DAP probes

Since hsprobe is clearly the fastest CMSIS-DAP probe in my testing, I thought it might be interested to compare it's performance vs the proprietary probes supported by probe-rs.

I did not expect either of the STLinks to perform this well, to be honest. It is nice to know you don't need an expensive probe to get good performance.

![stm32_write_hsprobe_vs_proprietary.svg](/assets/2022-04-04/stm32_write_hsprobe_vs_proprietary.svg)
![stm32_read_hsprobe_vs_proprietary.svg](/assets/2022-04-04/stm32_read_hsprobe_vs_proprietary.svg)

---
```
```
#### RTT performance
My assumption was that read/write throughput would have a direct correlation with RTT throughput, and that's mostly true.

One outlier here is MCU-Link running DAPLink firmware, which manages to acheive nearly twice the RTT throughput despite having lower read/write performance than the default MCU-Link firmware.

![rtt.svg](/assets/2022-04-04/rtt.svg)

---
```
```
### Conclusion

My main takeaways from this testing:
- CMSIS-DAP is not a very efficient protocol. The data rates achieved by the high-speed probes are below what could be achieved over USB Full Speed, and the Full Speed ones are even slower.  
  There are plenty of gains to be had by adding extensions like the hardware vendors do, while still preserving the universal compatibility of the standard.
- The performance of all USB Full Speed CMSIS-DAP V1 probes are basically equivalent, and the same seems to be true for V2 though my sample size is small here.
I would not recommend DAP42 or rust-dap if care about low-speed connections (long wires, etc) since it won't honour the requested speeds.
- HSProbe is the fastest CMSIS-DAP probe that I own
- STLink performance is better than expected.  
  The USB Full Speed STLink V2 being almost the same performance class as the cheaper USB High Speed probes was quite surprising.  
  The V3 being twice as fast as HSProbe is also not what I expected, but obviously the limitation is that you can only debug STM processors with this probe.
- JLink performance, even without using their DLL or proprietary protocol, is better than I expected. I was anticipating it being beaten by the Full Speed probes, and this is not the case.  
  It's still not good though. And if you want RTT, there are better CMSIS-DAP options.

#### Data
If you'd prefer to see the raw data or make your own graphs, [download the data in CSV from here](/assets/2022-04-04/data.csv)

---
```
```    

If you have enjoyed this content, please consider sponsoring probe-rs

I have no affiliation, but without their software I could not have done any of this testing.    

Plus, we all benefit when our tools improve.    

[https://github.com/sponsors/probe-rs](https://github.com/sponsors/probe-rs)
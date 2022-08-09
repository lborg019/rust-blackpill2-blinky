# rust-blackpill2-blinky
Rust blinky program for WeAct BlackPill v2.0 STM32F411CEU6

[STM32F411xC Data Sheet](https://www.st.com/resource/en/datasheet/stm32f411ce.pdf)
[STM32F411xC Reference Manual](https://www.st.com/resource/en/reference_manual/dm00119316-stm32f411xc-e-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf)

## Install / Run:

- On a separate terminal: `$ openocd -f interface/stlink.cfg -f target/stm32f4x.cfg`
- `$ git clone https://github.com/lborg019/- rust-blackpill2-blinky`
- `$ cargo build` 
- `$ cargo run`

If done correctly, the OpenOCD terminal should yield:
```bash
[loop] led.set_low()
[loop] led.set_high()
[loop] led.set_low()
[loop] led.set_high()
```
and the blue LED light on the BlackPill v2.0 should flash on/off every second.

## Debugging:

On the same terminal where the program was ran with `$ cargo run`, issue an interrupt sequence `(mac os) control+c`. This should halt the execution of the firmware and automatically invoke GDB. GDB commands issued here will run directly on the MCU. GDB's *continue* functionality `$ (gdb) c` should resume execution of the firmware program.

The BlackPill v2.0 STM32F411CEU6's blue LED light is wired to pin 13 on GPIOC. GPIO peripherals have an atomic Binary Set/Reset Register (BSRR) that when set correctly can drive pin 13 high or low. This can be done with GDB's *set* functionality `$ (gdb) set`.

STM32F411xC/xE Register boundary access:
| Bus  | Boundary Access           | Peripheral |
|------|---------------------------|------------|
| AHB1 | 0x4002 3800 - 0x4002 3BFF | RCC        |
| AHB1 | 0x4002 1C00 - 0x4002 1FFF | GPIOH      |
| AHB1 | 0x4002 1400 - 0x4002 1BFF | Reserved   |
| AHB1 | 0x4002 1000 - 0x4002 13FF | GPIOE      |
| AHB1 | 0x4002 0C00 - 0x4002 0FFF | GPIOD      |
| AHB1 | 0x4002 0800 - 0x4002 0BFF | GPIOC      |
| AHB1 | 0x4002 0400 - 0x4002 07FF | GPIOB      |
| AHB1 | 0x4002 0000 - 0x4002 03FF | GPIOA      |

8.4.6 GPIO port output data register (GPIOx_ODR) (x = A..E and H)
Address offset: `0x14`
Reset value: `0x0000 0000`

8.4.7 GPIO port bit set/reset register (GPIOx_BSRR) (x=A..E and H)
Address offset: `0x18`
Reset Value: `0x0000 0000`

To set the pin high/low on GPIOC, we start with the lowest address from the boundary access addresses, and add the BSRR offset:
`0x4002 0800` + BSRR offset `0x18` = `0x4002 0818`

set lights off by setting pin 13 (BS13):         
`$ (gdb) set *(0x40020818 as *mut i32) = 0b0000_0000_0000_0000_0010_0000_0000_0000`

set lights on by resetting pin 13 (BR13):
`$ (gdb) set *(0x40020818 as *mut i32) = 0b0010_0000_0000_0000_0000_0000_0000_0000`

GDB can print the 32 bit binary of the Output Data Register (ODR) by using the `examine` functionality the following way:
`$ (gdb) x /tw 0x40020814`
where t: binary , w: word (32 bit)
which should yield:
`0x40020814:	00000000000000000010000000000000` when LED is off
`0x40020814:	00000000000000000000000000000000` when LED is on

It is important to notice that setting the ODR register is possible, but would not necessarily work instantly since it is not an atomic instruction register. For the sake of debugging this blinky program, BSRR is the one that should be used.

## Stack:
#### Hardware:
- Mac OS machine with USB adapter
- [WeAct Black Pill V2.0 - STM32F411CEU6](https://stm32-base.org/boards/STM32F411CEU6-WeAct-Black-Pill-V2.0.html)
All pins must be soldered, including GND, SWSCK, SWDIO, and 3.3V. Working with a breadboard is not necessary, but it is usually better.
- ST-Link Programmer (tested and working with the Aideepen ST Link V2 Programming Unit [amazon.com](https://www.amazon.com/dp/B01J7N3RE6?psc=1&ref=ppx_yo2ov_dt_b_product_details)

#### Software:
- [OpenOCD](https://openocd.org/) to communicate with the chip
- [GCC](https://gcc.gnu.org/) to compile the code
- [GDB](https://sourceware.org/gdb/) to debug the firmware
- [Rust](https://www.rust-lang.org/) language to program the firmware
- [ST-LINK server](https://www.st.com/en/development-tools/st-link-server.html) for OpenOCD
- XCode and XCode command line tools (dependencies for GDB)

## References:
- https://github.com/stm32-rs/stm32f4xx-hal
- https://github.com/antoinevg/daisy_bsp
- https://forum.electro-smith.com/t/rust-starter-for-daisy-seed/684
- [Antoine van Gelder's inspiring ADC20 talk](https://www.youtube.com/watch?v=udlK1LQ3f3g)
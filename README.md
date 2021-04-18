# STM32-HAL

[![Crate](https://img.shields.io/crates/v/stm32-hal2.svg)](https://crates.io/crates/stm32-hal2)
[![Docs](https://docs.rs/stm32-hal2/badge.svg)](https://docs.rs/stm32-hal2)


This library provides high-level access to STM32 peripherals. It's based on the 
[STM32 Peripheral Access Crates](https://github.com/stm32-rs/stm32-rs) generated by 
[svd2rust](https://github.com/rust-embedded/svd2rust). It provides a consistent API across 
multiple STM32 families, with minimal code repetition. This makes it easy to switch MCUs 
within, or across families, for a given project.

**Family support**: F3, F4, L4, L5, G0, G4, and H7. U5 is planned once its SVD files and PAC
become available.

**Motivation**: Use STM32s in real-world hardware projects. Be able to switch MCUs with
minimal code change. 

**Design priority**: Get hardware working with a robust feature set.
As of this library's creation, this is not feasible with existing HAL crates due
to lack of maintainer time, and differing priorities. When the `stm32fyxx` ecosystem
is viewed as a whole, there's a lot of DRY.

## Getting started
Review the [syntax overview example](https://github.com/David-OConnor/stm32-hal/tree/main/examples/syntax_overview)
for example uses of many of this library's features. Copy and paste the whole example folder (It's set up
using [Knurling's app template](https://github.com/knurling-rs/app-template)), or copy parts of `Cargo.toml` 
and `main.rs` as required.

### Example highlights:
```rust
use cortex_m_rt::entry;
use stm32_hal2::{
    clocks::Clocks,
    gpio::{GpioB, PinNum, PinMode, OutputType, AltFn},
    i2c::{I2c, I2cDevice},
    low_power,
    pac,
    timer::{Event::TimeOut, Timer},
};

#[entry]
fn main() -> ! {
    let mut cp = cortex_m::Peripherals::take().unwrap();
    let mut dp = pac::Peripherals::take().unwrap();

    let clock_cfg = Clocks::default();
    clock_cfg.setup(&mut dp.RCC, &mut dp.FLASH).unwrap();

    let mut gpiob = GpioB::new(dp.GPIOB, &mut dp.RCC);
    let mut pb15 = gpiob.new_pin(PinNum::P15, PinMode::Output);
    pb15.set_high();

    let mut timer = Timer::new_tim3(dp.TIM3, 0.2, &clock_cfg, &mut dp.RCC);
    timer.listen(TimeOut);

    let mut scl = gpiob.new_pin(PinNum::P6, PinMode::Alt(AltFn::Af4));
    scl.output_type(OutputType::OpenDrain, &mut gpiob.regs);

    let mut sda = gpiob.new_pin(PinNum::P7, PinMode::Alt(AltFn::AF4));
    sda.output_type(OutputType::OpenDrain, &mut gpiob.regs);

    let i2c = I2c::new(dp.I2C1, I2cDevice::One, 100_000, &clock_cfg, &mut dp.RCC);

    loop {
        low_power::sleep_now(&mut cp.SCB);
    }
}
```

The library is influenced by the `stm32fyxx` HALs, and a number of the modules here are modified 
versions of those. There are some areas where design philosophy is different. For example: GPIO type-checking,
level-of-abstraction from registers/PAC, and clock config.
    
Most peripheral modules are independent: The only dependency they have within this library
is the `ClockCfg` trait, which we may move to a standalone crate later. This makes
it easy to interchange them with other projects.

PRs encouraged. Documenting each step using reference manuals is encouraged, but not required.

## Errata

- U[s]ART (serial) isn't implemented outside L4.
- DMA and CAN aren't implemented.
- H7 clocks are missing advanced features.
- PWM input unimplemented
- SPI unimplemented for H7
- CRC unimplemented for L5, F4, G0, and G4.
- Timer 15 can't set PSC on L5 due to a PAC error that's now fixed upstream on GH
- ADC is unimplemented on F4.
- ADC 3 and 4 are unimplemented on G4. ADC3 is unimplemented on H7.
- Some timer implementations are missing.
- Low power modes beyond sleep and cstop aren't implemented for H7.
- Waiting on U5 PAC before implementing
- USB clock (PLLQ) isn't set up for F4 - USB may not work.
- SPI3 unimplemented on L4x3 and L5. (PAC inconsistency of rcc `sp3en` instead of `spi3en`.)
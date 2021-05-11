# STM32-HAL

[![Crate](https://img.shields.io/crates/v/stm32-hal2.svg)](https://crates.io/crates/stm32-hal2)
[![Docs](https://docs.rs/stm32-hal2/badge.svg)](https://docs.rs/stm32-hal2)


This library provides high-level access to STM32 peripherals. It's based on the 
[STM32 Peripheral Access Crates](https://github.com/stm32-rs/stm32-rs) generated by 
[svd2rust](https://github.com/rust-embedded/svd2rust). It provides a consistent API across 
multiple STM32 families, with minimal code repetition. This makes it easy to switch MCUs 
within, or across families, for a given project.

**Family support**: F3, F4, L4, L5, G0, G4, and H7. U5 is planned once its SVD files and PAC
become available. WL and WB eventually.

**Motivation**: Use STM32s in real-world hardware projects. Be able to switch MCUs with
minimal code change. 

**Design priority**: Get hardware working with a robust feature set, aimed at
practical uses. The freedom to choose the right MCU for each project, without 
changing code bases.

## Getting started
Review the [syntax overview example](https://github.com/David-OConnor/stm32-hal/tree/main/examples/syntax_overview)
for example uses of many of this library's features. Copy and paste its whole folder (It's set up
using [Knurling's app template](https://github.com/knurling-rs/app-template)), or copy parts of `Cargo.toml` 
and `main.rs` as required.

### Example highlights:
```rust
use cortex_m;
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

The library is influenced by the `stm32fyxx` HALs, and several of the modules here are modified 
versions of those. There are some areas where design philosophy is different. For example: GPIO 
type-checking, level-of-abstraction from registers/PAC, role of EH traits in the API, 
feature parity among STM32 families, and clock config.
    
Most peripheral modules are independent: The only dependency they have within this library
is the `ClockCfg` trait, which we may move to a standalone crate later. This makes
it easy to interchange them with other projects.

The Rust docs page is built for STM32L4x3, and some aspects are not accurate for other
variants. We currently don't have a good solution to this problem, and may
self-host docs in the future.

## Contributing

PRs are encouraged. Documenting each step using reference manuals is encouraged, but not required.

Most peripheral modules use the following format:

- Enums for various config settings, that implement `#[repr(u8)]` for their associated register values
- A peripheral struct that has public fields for config. This struct also includes
a private `regs` field that is the appropriate reg block. Where possible, this is defined generically
in the implementation, eg:
`U: Deref<Target = pac::usart1::RegisterBlock>`. Reference the [stm32-rs-nightlies Github](https://github.com/stm32-rs/stm32-rs-nightlies)
to identify when we can take advantage of this.
- If config fields are complicated, we use a separate `PeriphConfig` struct owned by the peripheral struct.
This struct impls `Default`.
- Use raw pointers only when necessary. For example, pass `&mut dp.RCC` to methods when able.
- A constructor named `new` that performs setup code, including RCC peripheral enable and reset
- `enable_interrupt` and `clear_interrupt` functions, which accept an enum of interrupt type
- `embedded-hal` implementations as required, that call native methods. Note that
we design APIs based on STM32 capabilities, and apply EH traits as applicable.
- When available, base setup and usage steps on instructions provided in Reference Manuals.
These steps are copy+pasted in comments before the code that performs each one.
- Don't use PAC convenience field settings; they're implemented inconsistently across PACs.
(eg don't use something like `en.enabled()` - use `en.set_bit()`.)


### Example module structure:
```rust
#[derive(clone, copy)]
#[repr(u8)]
/// Select pulse repetiation frequency. Modifies `FCRDR_CR` register, `Prf` field.
enum Prf {
    Medium = 0,
    High = 1,
}

#[derive(clone, copy)]
/// Available interrupts. Enabled in `FCRDR_CR`, `...IE` fields. Cleared in `FCRDR_ICR`.
enum FcRadarInterrupt {
    TgtAcq,
    LostTrack,
}

/// Represents the Fire Control Radar peripheral.
pub struct FcRadar<F> {
    regs: F,
    prf: Prf,
}

impl<F> FcRadar<F>
where
    F: Deref<Target = pac::FCRDR::RegisterBlock>,
{
    pub fn new(regs: R, prf: Prf, rcc: &mut pac::RCC) -> Self {
        rcc_en_reset!(apb1, fcradar1, rcc);

        regs.cr.modify(|_, w| w.prf().bit(prf as u8 != 0));        

        Self { regs, prf }
    }

    /// Track a target. See H8 RM, section 3.3.5.
    pub fn track(&mut self, hit_num: u8) -> Self {
        // RM: "To begin tracking a target, perform the following steps:"

        // 1. Select the hit to track to setting the HIT bits in the FCRDR_TR register. 
        #[cfg(feature = "h8")]
        self.regs.tr.modify(|_, w| unsafe { w.HIT().bits(hit_num) });
        #[cfg(feature = "g5")]
        self.regs.tr.modify(|_, w| unsafe { w.HITN().bits(hit_num) });

        // 2. Begin tarcking by setting the TRKEN bit in the FCRDR_TR register.
        self.regs.tr.modify(|_, w| w.TRKEN().set_bit());

    }
    
    /// Enable an interrupt.
    pub fn enable_interrupt(&mut self, interrupt: FcRadarInterrupt) {
        match interrupt {
            FcRadarInterrupt::TgtAcq => self.regs.cr.modify(|_, w| w.taie().set_bit()),
            FcRadarInterrupt::LostTrack => self.regs.cr.modify(|_, w| w.ltie().set_bit()),
        }   
    }

    /// Clear an interrupt flag - run this in the interrupt's handler to prevent
    /// repeat firings.
    pub fn clear_interrupt(&mut self, interrupt: FcRadarInterrupt) {
        match interrupt {
            FcRadarInterrupt::TgtAcq => self.regs.icr.write(|w| w.tacf().set_bit()),
            FcRadarInterrupt::LostTrack => self.regs.icr.write(|w| w.ltcf().set_bit()),
        }   
    }
}
```

## Errata

- SAI, SDIO, ethernet unimplemented
- DMA only implemented for USART, and only on L4 and G4.
- Only bxCAN is implemented - the fdCAN used on newer families is unimplemented
- USART synchronous mode, and auto-baud-rate detection unimplemented
- USART interrupts unimplemented on F4
- H7 clocks are missing advanced features
- PWM input unimplemented
- SPI unimplemented for H7
- CRC unimplemented for L5, F4, G0, and G4
- Flash read/write unimplemented on H7
- Low power timers (LPTIM) unimplemented
- Timer 15 can't set PSC on L5 due to a PAC error that's now fixed upstream on GH
- ADC unimplemented on F4
- ADC3 unimplemented on H7
- Low power modes beyond sleep and cstop aren't implemented for H7
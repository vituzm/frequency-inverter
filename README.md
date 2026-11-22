# Frequency Inverter — STM32F446RE

Prototype firmware for controlling a frequency inverter using an STM32F446RE. The project generates complementary PWM references from a sinusoidal lookup table, monitors an analog input via ADC/DMA, and allows starting or stopping actuation over a UART serial interface.

> [!WARNING]
> This project deals with power drive concepts and may involve hazardous voltages. Do not directly connect high-voltage loads to the Nucleo board or MCU signals. Use proper isolation, gate drivers, protection circuitry, and other appropriate hardware before testing on a real bridge or inverter.

## Features

- Generation of two complementary PWM references using a 32-sample sine table.
- DMA control for `TIM2` channels `CH1` and `CH2`.
- Electrical frequency initialized at 60 Hz and adjusted to 50 Hz via the `S` serial command.
- 12-bit ADC sampling on input `PA0`, triggered by `TIM3`.
- Moving-average filtering over the last 32 ADC samples.
- Control of a relay output on `PC0` based on the average ADC reading.
- `UART2` interface at 115200 bps to start or stop PWM generation.
- Support for the Node-RED dashboard/flow provided in `flows.json`.

## Hardware

### Microcontroller

- STM32F446RET6, system clock configured at 84 MHz.
- Recommended IDE: STM32CubeIDE.

### Pinout & Connections

| Function | Pin | Hardware / Peripheral |
|---|---|---|
| Positive PWM | `PA15` | `TIM2 CH1` |
| Negative PWM | `PB9` | `TIM2 CH2` |
| ADC sampling trigger | `PA6` | `TIM3 CH1/TRGO` |
| Analog input | `PA0` | `ADC1 IN0` |
| Control / relay output | `PC0` | GPIO output |
| UART TX | `PA2` | `USART2 TX` |
| UART RX | `PA3` | `USART2 RX` |
| Nucleo user LED | `PA5` | GPIO output |
| Blue user button | `PC13` | GPIO falling interrupt |

## Firmware Architecture

### Clock and Timers

RCC is configured with the internal HSI oscillator and PLL to achieve an 84 MHz system clock.

- **`TIM2`:** Generates PWM signals on `CH1` and `CH2`. DMA transfers the sinusoidal buffers `pwmP` and `pwmN` in circular mode to the compare registers.
- **`TIM3`:** Generates an update event configured as `TRGO` at 100 Hz to trigger `ADC1` conversions.
- **`ADC1`:** Uses a single channel, 12-bit resolution, and DMA to fill the `medidas` array with 32 samples.

## PWM Generation

During initialization, the firmware builds two 32-entry buffers:

- `pwmP[N_AMOSTRAS]`: positive half-wave profile.
- `pwmN[N_AMOSTRAS]`: negative half-wave profile.

The buffers are transferred via DMA to both `TIM2` channels. When the `S` character is received over UART, the firmware starts DMA on both channels and sets the frequency to 50 Hz. The `P` character stops both PWM channels.

`TIM2` frequency is recalculated at the end of each ADC block using:

```text
PSC = TIM_INTERNO / (frequenciaMotor * N_AMOSTRAS * 100) - 1
```

Here, `TIM_INTERNO` is 84 MHz and the `TIM2` base period is configured to 100 ticks.

## Analog Control

Each time DMA completes a 32-sample ADC block, the firmware:

1. Computes the arithmetic mean of the readings.
2. Drives `PC0` high if the average is greater than `3400`.
3. Drives `PC0` low otherwise.
4. Restarts a new ADC DMA acquisition.

With 12-bit resolution, the theoretical maximum ADC value is 4095. The current switching threshold is therefore roughly:

```text
3400 / 4095 ≈ 83% of the ADC full scale
```

The actual voltage threshold depends on the ADC reference voltage used.

## UART Control

`USART2` operates with the following framing:

- Baud rate: `115200`
- Data bits: `8`
- Parity: `none`
- Stop bits: `1`

Supported commands:

| Command | Action |
|---|---|
| `S` | Sets 50 Hz and starts both PWM channels via DMA |
| `P` | Stops both PWM channels |

After initialization, the firmware sends `BEM VINDO(A)` and waits for incoming characters via interrupt. Any other character leaves the state set in the callback unchanged, while re-arming UART reception for the next command.

## Project Structure

```text
.
├── README.md
├── flows.json                 # Node-RED flow for UI / monitoring
└── Inversor_code/
    ├── Inversor.ioc           # Original STM32CubeMX configuration
    ├── Core/
    │   ├── Inc/               # Header files
    │   └── Src/               # main.c and HAL initialization
    ├── Drivers/               # CMSIS and STM32 HAL
    ├── Debug/                 # Existing build artifacts and files
    └── Inversor Debug.launch  # Eclipse / STM32CubeIDE debug configuration
```

The main application source file is [`Inversor_code/Core/Src/main.c`](Inversor_code/Core/Src/main.c).

## Building and Flashing

1. Install STM32CubeIDE.
2. Open the `Inversor_code` folder as an existing project.
3. Select the `Debug` build configuration.
4. Build the project.
5. Connect the ST-LINK / Nucleo board to your computer.
6. Flash `Debug/Inversor.elf` and run the program.

Existing build outputs in `Debug/` include `Inversor.elf`, `Inversor.list`, and intermediate object files.

## Quick Start

1. Power the board and open a serial terminal on `PA2`/`PA3` via `UART2`.
2. Configure the terminal for `115200 8-N-1`.
3. Send `S` to start 50 Hz PWM generation.
4. Send `P` to stop PWM generation.
5. Apply and monitor the analog signal on `PA0` within a safe voltage range.
6. Observe the `PC0` output state according to the average ADC value.

## Technical Notes

- `frequenciaMotor` starts at 60 Hz, but only the `S` command path explicitly changes its value to 50 Hz.
- `pwmP` and `pwmN` are generated once during the initial setup stage of `main()`.
- ADC acquisition uses `TIM3` as an external trigger and is manually restarted via DMA inside the conversion-complete callback.
- The LED and push-button defined in the project follow the standard Nucleo-64 board layout, though the button callback is not implemented in the current `main.c`.
- `flows.json` can be imported into Node-RED to visualize or control the project via a graphical interface, provided the ST-Link/serial environment is properly configured.

## Key Files

- `Inversor_code/Core/Src/main.c`: application entry point, timers, ADC, DMA, UART, and control logic.
- `Inversor_code/Core/Src/stm32f4xx_hal_msp.c`: HAL peripheral-to-pin MSP initialization.
- `Inversor_code/Core/Inc/main.h`: pin definitions and main function prototypes.
- `Inversor_code/Inversor.ioc`: STM32CubeMX / CubeIDE project configuration.
- `flows.json`: associated Node-RED flow.
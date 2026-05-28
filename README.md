# LPuart Two-Board Demo

This repository contains a two-board UART experiment based on `STM32U575VGTx`.
It includes two separate STM32CubeMX / Keil projects:

- `LPuart`: Board 1, used as a transparent bridge
- `LPuart2`: Board 2, used as a simple response node

The end-to-end behavior is:

1. The PC sends data to Board 1 through `USART1`.
2. Board 1 forwards the data to Board 2 through `LPUART1`.
3. Board 2 receives one frame, appends `*`, and sends it back.
4. Board 1 forwards the response back to the PC through `USART1`.

If the PC sends `hello`, the expected response is `hello*`.

## Overview

This project is a compact validation setup for:

- `USART1` communication between PC and STM32
- `LPUART1` communication between two STM32 boards
- polling-based serial forwarding
- simple frame detection based on idle timeout

Both projects use:

- MCU: `STM32U575VGTx`
- Framework: `STM32CubeMX + HAL`
- UART format: `115200 8N1`

## Project Layout

```text
lpuart_two/
|-- LPuart/      # Board 1: PC <-> Board 2 transparent bridge
|-- LPuart2/     # Board 2: receive, append '*', send back
`-- README.md
```

## Board 1: `LPuart`

Board 1 acts as a transparent bridge between the PC and Board 2.

Main roles:

- `USART1` connects to the PC serial tool
- `LPUART1` connects to Board 2
- the main loop polls both UART peripherals
- two ring buffers are used for bidirectional forwarding

Core data paths:

- `USART1 -> pc_ring -> LPUART1`
- `LPUART1 -> b2_ring -> USART1`

Implementation notes:

- polling is used instead of UART interrupts
- register-level access is used in the forwarding loop
- ring buffer size is `512` bytes
- UART error flags are cleared explicitly in the main loop
- Board 1 prints a startup banner to the PC:
  - `[Board1] Bridge Ready`

Key files:

- `LPuart/Core/Src/main.c`
- `LPuart/Core/Src/stm32u5xx_hal_msp.c`
- `LPuart/LPuart.ioc`

## Board 2: `LPuart2`

Board 2 receives a frame from Board 1, appends `*`, and sends it back.

Main roles:

- only `LPUART1` is used
- bytes are collected into a receive buffer
- a frame ends after a short idle period
- the response is the original frame plus one `*`

Implementation notes:

- receive buffer size is `256` bytes
- frame end condition is `30 ms` of silence
- receive path uses polling
- transmit path uses `HAL_UART_Transmit`
- Board 2 sends a startup banner:
  - `[Board2] Ready`

Key files:

- `LPuart2/Core/Src/main.c`
- `LPuart2/Core/Src/stm32u5xx_hal_msp.c`
- `LPuart2/LPuart2.ioc`

## Pin Mapping

### Board 1: `LPuart`

- `USART1_TX`: `PA9`
- `USART1_RX`: `PA10`
- `LPUART1_RX`: `PC0`
- `LPUART1_TX`: `PC1`

### Board 2: `LPuart2`

- `LPUART1_RX`: `PC0`
- `LPUART1_TX`: `PC1`

## Wiring

### Board 1 to PC

- Board 1 `PA9` -> USB-UART adapter `RX`
- Board 1 `PA10` -> USB-UART adapter `TX`
- common `GND`

### Board 1 to Board 2

- Board 1 `PC1 (TX)` -> Board 2 `PC0 (RX)`
- Board 1 `PC0 (RX)` -> Board 2 `PC1 (TX)`
- common `GND`

## UART Settings

- Baud rate: `115200`
- Data bits: `8`
- Stop bits: `1`
- Parity: `None`
- Flow control: `None`

## Software Flow

```text
PC
 |
 | USART1
 v
Board1 (LPuart)
 |
 | LPUART1
 v
Board2 (LPuart2)
 |
 | append '*'
 v
Board1
 |
 | USART1
 v
PC
```

## Build and Flash

The repository already contains Keil MDK-ARM project files:

- `LPuart/MDK-ARM/LPuart.uvprojx`
- `LPuart2/MDK-ARM/LPuart2.uvprojx`

Recommended steps:

1. Build both projects.
2. Flash `LPuart` to Board 1.
3. Flash `LPuart2` to Board 2.
4. Connect the UART lines and common ground.
5. Open a serial terminal on the PC side for Board 1 `USART1`.
6. Send any text and check whether the reply ends with `*`.

## Expected Runtime Behavior

Typical startup behavior:

- Board 1 sends `[Board1] Bridge Ready` to the PC.
- Board 2 sends `[Board2] Ready` on its `LPUART1`.

Example:

```text
PC sends:    STM32
PC receives: STM32*
```

## Code Analysis Summary

This repository is best understood as a UART link validation demo rather than a reusable protocol stack.

What it demonstrates well:

- basic `USART1` and `LPUART1` configuration on STM32U5
- serial forwarding between two boards
- polling-based non-blocking bridge logic on Board 1
- simple idle-time frame detection on Board 2

Current limitations:

- Board 1 drops bytes if a ring buffer becomes full
- Board 2 uses `30 ms` silence as frame termination, which is simple but not robust for all protocols
- Board 2 transmit uses blocking HAL transmit
- CubeMX-generated NVIC settings are present, but user code disables UART interrupts during runtime

## Suggested Improvements

- switch RX/TX paths to interrupt or DMA mode
- define an explicit frame protocol with header, length, or terminator
- add counters for dropped bytes and UART errors
- add a `.gitignore` for Keil user files and build outputs


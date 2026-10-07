# STM32F407 CAN Internal Loopback

A beginner-friendly practical demonstration of **Classical CAN communication using the STM32F407G-DISC1** in **internal Loopback Mode** with STM32CubeIDE.

The purpose of this project is to verify CAN transmission and reception completely inside the STM32 CAN peripheral without requiring a second CAN node or external CAN transceiver.

---

## Project Overview

In this experiment, the STM32F407 CAN1 peripheral is configured in **Loopback Mode**.

A CAN message is transmitted by CAN1 and internally fed back to its receive path.

```text
              STM32F407

        ┌─────────────────────┐
        │                     │
        │     CAN1 / bxCAN    │
        │                     │
        │   TX ──────────┐    │
        │                │    │
        │                ▼    │
        │          Internal   │
        │          Loopback   │
        │                │    │
        │                ▼    │
        │   RX ──────────┘    │
        │                     │
        └─────────────────────┘
```

According to ST's bxCAN documentation, Loopback Mode internally feeds the transmitted CAN data back to the receive path and stores it in a receive mailbox if it passes the acceptance filter.

---

# Hardware

### Required

* STM32F407G-DISC1
* USB cable
* Computer

### Not required

* CAN transceiver
* SN65HVD230
* CANH/CANL wiring
* 120 Ω termination resistor
* Second STM32 board
* USB-CAN analyzer

This is an **internal CAN controller test**, not a physical CAN bus test.

---

# Software

* STM32CubeIDE
* STM32CubeMX configuration integrated with CubeIDE
* STM32 HAL

---

# STM32 CAN Configuration

The CAN peripheral used in this project is:

```text
CAN Instance : CAN1
Mode         : Loopback
```

The project was generated using STM32CubeMX/CubeIDE.

### CAN Configuration

```text
CAN Instance          : CAN1
CAN Mode              : CAN_MODE_LOOPBACK

Prescaler             : 16
Synchronization Jump  : CAN_SJW_1TQ
Time Segment 1        : CAN_BS1_11TQ
Time Segment 2        : CAN_BS2_4TQ

Time Triggered Mode   : Disabled
Auto Bus-Off          : Disabled
Auto Wake-Up          : Disabled
Auto Retransmission   : Disabled
Receive FIFO Locked   : Disabled
Transmit FIFO Priority: Disabled
```

The project clock configuration gives a PCLK1 of approximately **6.25 MHz**. With the selected CAN timing parameters:

```text
CAN clock = 6.25 MHz
Prescaler = 16

Total time quanta =
1 + 11 + 4
= 16 TQ

Bit rate ≈
6.25 MHz / (16 × 16)
≈ 24.4 kbps
```

This bitrate is sufficient for this internal loopback demonstration because there is no external CAN node involved.

For a future physical CAN network, the timing parameters should be recalculated for the required bus bitrate.

---

# CAN Filter Configuration

A CAN acceptance filter is configured so that received messages can enter FIFO0.

```c
canFilter.FilterBank = 0;
canFilter.FilterMode = CAN_FILTERMODE_IDMASK;
canFilter.FilterScale = CAN_FILTERSCALE_32BIT;

canFilter.FilterIdHigh = 0x0000;
canFilter.FilterIdLow = 0x0000;

canFilter.FilterMaskIdHigh = 0x0000;
canFilter.FilterMaskIdLow = 0x0000;

canFilter.FilterFIFOAssignment = CAN_RX_FIFO0;
canFilter.FilterActivation = ENABLE;

canFilter.SlaveStartFilterBank = 14;
```

The zero mask allows the received identifier to pass the filter.

---

# CAN Message Used

The transmitted CAN message in this experiment is:

```text
CAN ID : 0x123
DLC    : 8

Data:
11 22 33 44 55 66 77 88
```

The transmit configuration is:

```c
TxHeader.StdId = 0x123;
TxHeader.IDE = CAN_ID_STD;
TxHeader.RTR = CAN_RTR_DATA;
TxHeader.DLC = 8;
```

Data:

```c
uint8_t TxData[8] =
{
    0x11,
    0x22,
    0x33,
    0x44,
    0x55,
    0x66,
    0x77,
    0x88
};
```

---

# Transmission

The message is transmitted using:

```c
HAL_CAN_AddTxMessage(
    &hcan1,
    &TxHeader,
    TxData,
    &TxMailbox
);
```

The CAN controller then internally loops the transmitted message back to its receive path.

---

# Reception

The program checks whether CAN FIFO0 contains a received message:

```c
HAL_CAN_GetRxFifoFillLevel(
    &hcan1,
    CAN_RX_FIFO0
);
```

If a message is available, it is read using:

```c
HAL_CAN_GetRxMessage(
    &hcan1,
    CAN_RX_FIFO0,
    &RxHeader,
    RxData
);
```

The received message is then checked against the expected identifier and data.

Expected result:

```text
RxHeader.StdId = 0x123
RxHeader.DLC    = 8

RxData[0] = 0x11
RxData[1] = 0x22
RxData[2] = 0x33
RxData[3] = 0x44
RxData[4] = 0x55
RxData[5] = 0x66
RxData[6] = 0x77
RxData[7] = 0x88
```

---

# Debugging in STM32CubeIDE

The project can be verified directly using the STM32CubeIDE debugger.

A breakpoint can be placed at:

```c
can_received = 1;
```

When the debugger stops there, the following variables can be inspected:

```text
RxHeader.StdId
RxHeader.DLC
RxData[0]
RxData[1]
RxData[2]
RxData[3]
RxData[4]
RxData[5]
RxData[6]
RxData[7]
```

Expected values:

```text
ID   = 0x123
DLC  = 8

DATA = 11 22 33 44 55 66 77 88
```

---

# Hardware Register Verification

The CAN peripheral registers can also be inspected through the STM32CubeIDE SFR/debugger view.

Important registers include:

```text
CAN_BTR
CAN_TSR
CAN_RF0R

CAN_RI0R
CAN_RDT0R
CAN_RDL0R
CAN_RDH0R
```

The receive registers provide a hardware-level view of the received CAN frame.

For example:

```text
CAN_RI0R
    ↓
Received Identifier

CAN_RDT0R
    ↓
Data Length Code

CAN_RDL0R
    ↓
First 4 data bytes

CAN_RDH0R
    ↓
Last 4 data bytes
```

This allows the experiment to be verified at both:

1. **Software level** – `RxHeader` and `RxData`
2. **Hardware peripheral level** – CAN receive registers

---

# Project Flow

```text
Start
  │
  ▼
HAL_Init()
  │
  ▼
System Clock Configuration
  │
  ▼
MX_GPIO_Init()
  │
  ▼
MX_CAN1_Init()
  │
  ▼
Configure CAN Filter
  │
  ▼
Start CAN1
  │
  ▼
Prepare CAN Frame
  │
  ▼
Transmit CAN Frame
  │
  ▼
Internal Loopback
  │
  ▼
CAN RX FIFO0
  │
  ▼
Read CAN Message
  │
  ▼
Compare ID + Data
  │
  ▼
PASS
```

---

# Result

The internal CAN loopback test successfully demonstrates:

* CAN1 initialization
* CAN Loopback Mode
* CAN acceptance filtering
* CAN frame transmission
* CAN receive FIFO operation
* CAN frame reception
* Standard 11-bit CAN identifier
* DLC = 8
* 8-byte CAN payload
* Debugging CAN registers and received data in STM32CubeIDE

### Expected Result

```text
CAN ID : 0x123
DLC    : 8

Received:
11 22 33 44 55 66 77 88

CAN LOOPBACK TEST : PASS
```

---

# Demonstration Video

A demonstration video showing the project running in STM32CubeIDE will be added here.

**Video:** `https://drive.google.com/file/d/1DKLFAjMTmk7rA5HURK9LhQ-03kVo-FjX/view?usp=sharing`

The video demonstrates:

1. STM32CubeIDE project configuration
2. CAN1 Loopback configuration
3. Program execution
4. CAN message transmission
5. CAN message reception
6. Debugger inspection of `RxHeader` and `RxData`
7. CAN peripheral register inspection

---

# Project Files

Recommended repository structure:

```text
STM32F407-CAN-LoopBack/
│
├── README.md
│
│
├── Core/
│   ├── Inc/
│   └── Src/
│       └── main.c
│
├── Drivers/
│
├── .ioc
```

---

# Important Note

This project demonstrates **internal CAN loopback**, not communication over a physical CAN bus.

No CAN transceiver is used.

For actual CAN bus communication, the next stage will be:

```text
STM32F407
    │
    │ CAN TX/RX
    ▼
CAN Transceiver
(SN65HVD230)
    │
    ├──────── CANH
    │
    └──────── CANL
              │
              ▼
        Another CAN Node
```

The next practical stage can use an STM32 and another low-cost CAN-capable controller/transceiver combination.

---

# References

* STMicroelectronics STM32F407 Reference Manual (RM0090)
* STMicroelectronics STM32F407 documentation
* STM32CubeIDE / STM32CubeMX documentation

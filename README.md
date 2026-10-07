# Controller Area Network (CAN) – Basics and Operating Modes

## 1. What is CAN?

**CAN (Controller Area Network)** is a robust serial communication protocol designed for reliable communication between multiple electronic control units (ECUs) without requiring a central host.

CAN is widely used in:

* Automotive systems
* Industrial automation
* Robotics
* Medical equipment
* Embedded systems
* Aerospace and transportation
* Battery Management Systems (BMS)
* Motor control systems

Unlike UART, where communication is generally between two devices using separate TX and RX lines, CAN is designed as a **multi-node shared bus**.

---

## 2. Why was CAN developed?

Traditional point-to-point communication can require many communication lines when multiple controllers need to communicate.

CAN allows multiple nodes to share the same two-wire differential bus:

```text
             CANH ──────────────────────────────
                  │        │        │
               Node 1   Node 2   Node 3
                  │        │        │
             CANL ──────────────────────────────
```

Each node can transmit or receive CAN frames on the same bus.

CAN does not use a device address in the same way as UART/I2C. Instead, messages contain an **identifier**, which also determines their priority during bus arbitration.

---

# 3. Basic CAN Network

A typical physical CAN network contains:

```text
             CAN Bus
       CANH ───────────────────────
             │       │       │
          Node 1   Node 2   Node 3
             │       │       │
       CANL ───────────────────────

       120 Ω                    120 Ω
        │                          │
        └──────────────────────────┘
          Termination resistors
```

Each CAN node generally contains:

```text
Application
     │
     ▼
CAN Controller
     │
     ▼
CAN Transceiver
     │
     ▼
CANH / CANL
```

### CAN Controller

The CAN controller handles:

* CAN frame generation
* Bit timing
* Arbitration
* Error detection
* Message filtering
* Transmission
* Reception

For example, the STM32F407 contains a **bxCAN controller**.

### CAN Transceiver

The transceiver converts the controller's logic-level signals into differential CANH/CANL signals and converts received differential signals back to logic levels.

Examples include:

* SN65HVD230
* SN65HVD232
* TJA1050

A transceiver is required for a normal physical CAN network.

---

# 4. CAN Bus Signals

CAN uses two differential signals:

* **CANH** – CAN High
* **CANL** – CAN Low

The differential voltage between CANH and CANL represents the CAN bus state.

CAN uses two logical bus states:

### Recessive

```text
CANH ≈ CANL
```

### Dominant

```text
CANH > CANL
```

The differential signaling provides good immunity against electrical noise.

---

# 5. CAN is a Multi-Master Protocol

CAN uses a **multi-master architecture**.

This means any node can attempt to transmit when the bus is available.

For example:

```text
Node A ──┐
Node B ──┼── CAN Bus
Node C ──┘
```

There is no permanent master that controls the communication.

If multiple nodes attempt to transmit simultaneously, CAN uses **arbitration** to determine which message gets access to the bus.

---

# 6. CAN Message Identifier

Every CAN message contains an identifier.

For example:

```text
ID = 0x123
```

The identifier is not simply the address of a device.

It generally represents the **meaning or priority of the message**.

For example:

```text
0x100 → Engine speed
0x200 → Vehicle speed
0x300 → Temperature
0x400 → Battery status
```

Lower numerical CAN identifiers have higher arbitration priority.

---

# 7. CAN Arbitration

Suppose two nodes start transmitting simultaneously:

```text
Node A → ID 0x100
Node B → ID 0x200
```

CAN performs bit-by-bit arbitration.

A dominant bit overrides a recessive bit.

The node transmitting the lower-priority identifier detects that it has lost arbitration and stops transmitting.

The higher-priority message continues without being corrupted.

This is one of the major advantages of CAN.

---

# 8. CAN Frame

A Classical CAN data frame contains fields such as:

```text
┌─────┬─────┬─────┬─────┬─────┬──────┬─────┐
│ SOF │ ID  │Ctrl │ DLC │Data │ CRC   │ ACK │
└─────┴─────┴─────┴─────┴─────┴──────┴─────┘
```

Important fields include:

### SOF – Start of Frame

Marks the beginning of the frame.

### Identifier

Contains the message identifier.

Classical CAN supports:

* Standard identifier → 11 bits
* Extended identifier → 29 bits

### DLC – Data Length Code

Specifies the number of data bytes.

For Classical CAN:

```text
DLC = 0 → 0 bytes
DLC = 1 → 1 byte
...
DLC = 8 → 8 bytes
```

### Data

Contains the actual payload.

Classical CAN supports up to **8 data bytes per frame**.

### CRC

Used for error detection.

### ACK

Used by receiving nodes to acknowledge successful frame reception.

---

# 9. Classical CAN vs CAN FD

## Classical CAN

```text
Maximum payload = 8 bytes
```

## CAN FD

CAN FD extends CAN by allowing:

```text
Maximum payload = 64 bytes
```

CAN FD can also use a higher data-phase bit rate.

This project uses **Classical CAN / bxCAN**, not CAN FD.

---

# 10. CAN Bit Rate

CAN communication requires nodes on the same bus to use compatible bit timing.

Common CAN bit rates include:

```text
125 kbps
250 kbps
500 kbps
1 Mbps
```

The actual timing depends on:

* CAN peripheral clock
* Prescaler
* Time Segment 1
* Time Segment 2
* Synchronization Jump Width

For STM32 bxCAN, these parameters are configured through the CAN bit timing registers.

---

# 11. CAN Error Detection

CAN contains several mechanisms for detecting communication errors.

Important mechanisms include:

* Bit monitoring
* Bit stuffing
* CRC checking
* Frame checking
* ACK checking

CAN can detect errors and initiate error handling automatically.

This makes CAN suitable for electrically noisy environments.

---

# 12. CAN Acceptance Filtering

CAN controllers can filter received messages.

For example, suppose a node is interested only in:

```text
ID = 0x123
```

Instead of processing every message on the bus, hardware filtering can be configured so that only selected identifiers are placed into the receive FIFO.

STM32 bxCAN provides hardware acceptance filters.

---

# 13. CAN Operating Modes

STM32 bxCAN provides several important operating modes.

## 13.1 Normal Mode

This is the normal operating mode for communication on a physical CAN bus.

```text
STM32
  │
CAN Controller
  │
Transceiver
  │
CANH / CANL
  │
Other CAN nodes
```

In this mode, the controller participates normally in:

* Transmission
* Reception
* Arbitration
* ACK
* Error handling

A physical CAN network and transceiver are normally required.

---

## 13.2 Loopback Mode

Loopback mode is primarily used for testing and self-test.

The transmitted message is internally fed back into the receive path.

```text
        STM32 CAN Controller

             TX
              │
              ▼
        ┌───────────┐
        │  CAN Core │
        └─────┬─────┘
              │
          Internal
           Loopback
              │
              ▼
             RX
```

The STM32 can therefore transmit a CAN message and receive its own message without another CAN node.

This is the mode used in the practical project in this repository.

ST documents that bxCAN loopback internally feeds the Tx output back to the Rx input and stores the message in a receive mailbox if it passes the acceptance filter.

### Hardware required

For this internal loopback experiment:

* STM32F407G-DISC1
* USB cable

No external:

* CAN transceiver
* CANH/CANL wiring
* second STM32
* USB-CAN analyzer

is required.

---

## 13.3 Silent Mode

Silent mode is also known as **listen-only mode**.

The CAN controller can monitor traffic on an existing CAN bus without actively participating in transmission.

It is useful for:

* CAN bus monitoring
* Debugging
* Traffic analysis

The controller does not disturb the existing bus by transmitting dominant bits.

---

## 13.4 Loopback + Silent Mode

STM32 bxCAN can combine loopback and silent modes.

This is useful for self-testing while avoiding interference with an active CAN system.

ST refers to this as a "Hot Selftest" configuration.

---

# 14. CAN Controller vs CAN Transceiver

These two components should not be confused.

### CAN Controller

Handles the CAN protocol.

```text
Frame
Arbitration
CRC
Bit timing
Filtering
TX/RX
```

### CAN Transceiver

Handles the physical electrical interface.

```text
Logic TX/RX
      ↕
CANH/CANL
```

For example:

```text
STM32F407
┌──────────────────┐
│                  │
│   CAN Controller │
│                  │
└────────┬─────────┘
         │ TX/RX
         ▼
┌──────────────────┐
│ CAN Transceiver  │
│ SN65HVD230       │
└────────┬─────────┘
         │
      CANH/CANL
```

The internal loopback experiment can bypass the external transceiver because the test occurs inside the CAN controller.

---

# 15. STM32F407 bxCAN

The STM32F407 contains the **bxCAN** peripheral.

Important CAN concepts when working with STM32 include:

* CAN controller
* CAN initialization
* Bit timing
* CAN filters
* Transmit mailboxes
* Receive FIFOs
* CAN identifiers
* CAN data
* CAN status/error registers
* CAN operating modes

The STM32F407 reference manual documents bxCAN, including normal, silent, loopback and combined test modes.

---

# 16. Typical CAN Communication Flow

A typical transmission is:

```text
Application
     │
     ▼
Prepare CAN message
     │
     ▼
CAN Identifier
     │
     ▼
DLC + Data
     │
     ▼
CAN Controller
     │
     ▼
Arbitration
     │
     ▼
CAN Transceiver
     │
     ▼
CANH / CANL
     │
     ▼
Other CAN nodes
     │
     ▼
Receive Filter
     │
     ▼
Receive FIFO
     │
     ▼
Application
```

---

# 17. What I am learning in this repository

This repository starts with CAN at the controller level before moving to physical CAN communication.

### Stage 1

```text
STM32F407
   ↓
Internal CAN Loopback
```

### Stage 2

```text
STM32F407
   ↓
CAN Transceiver
   ↓
CANH/CANL
```

### Stage 3

```text
STM32F407 ←→ CAN Bus ←→ ESP32 / Other CAN Node
```

This approach makes it possible to understand the CAN controller and frame handling before introducing physical bus hardware.

## Relationship Between LTSSM and Receiver Detection

The LTSSM decides **when receiver detection must happen**, while the PIPE/PHY performs the actual electrical receiver-detection operation.

### LTSSM flow

A simplified LTSSM sequence is:

```text
Detect.Quiet
    ↓
Detect.Active
    ↓
Receiver Detection
    ↓
Receiver found?
    ├── Yes → Polling
    └── No  → Detect.Quiet
```

When the LTSSM enters `Detect.Active`, it requests the PHY to determine whether a valid receiver is connected on the PCIe lane.

The LTSSM itself does not perform the analog receiver-detection operation.

Instead:

```text
LTSSM
    ↓
MAC / PIPE control logic
    ↓
PHY
    ↓
Electrical receiver detection
```

### LTSSM to PIPE relationship

In a PIPE-based implementation, the flow is approximately:

```text
        PCIe MAC / LTSSM
               |
               | Detect.Active
               v
        PIPE Control Logic
               |
               | PowerDown = P1
               | TxElecIdle = 1
               | TxDetectRx = 1
               v
              PHY
               |
               | Performs electrical
               | receiver detection
               v
       PhyStatus + RxStatus
               |
               v
        PIPE Control Logic
               |
               | detect_done
               | detect_rx_present
               v
             LTSSM
```

### Receiver-detection request

In our design, the LTSSM-side request is represented by:

```verilog
mac_detect_req
```

The PIPE wrapper first checks whether receiver detection is currently allowed:

```verilog
wire detect_allowed;

assign detect_allowed =
    (PowerDown == P1) &&
    TxElecIdle;
```

This means receiver detection can start only when:

```text
PowerDown = P1
TxElecIdle = 1
```

### Starting the operation

The complete launch condition is:

```verilog
assign start_detect_operation =
    (state == ST_IDLE) &&
    !start_rate_operation &&
    !start_power_operation &&
    detect_pending &&
    mac_detect_req &&
    detect_allowed;
```

Therefore receiver detection starts only when:

```text
PIPE controller is idle
AND
no higher-priority operation is starting
AND
a receiver-detect request is pending
AND
the LTSSM still wants receiver detection
AND
the PIPE receiver-detection conditions are satisfied
```

### Driving the PHY

When receiver detection starts, the wrapper asserts:

```verilog
TxDetectRx <= 1'b1;

active_operation <= OP_DETECT;
state            <= ST_WAIT;
```

The relevant PIPE combination becomes:

```text
PowerDown  = P1
TxElecIdle = 1
TxDetectRx = 1
```

The PHY interprets this as:

```text
Perform Receiver Detection
```

### PHY completion

After performing the electrical detection, the PHY asserts:

```text
PhyStatus = 1
```

The receiver-detection result is returned through `RxStatus`.

For our supported PIPE implementation:

```text
RxStatus = 3'b011
    → Receiver detected

RxStatus = 3'b000
    → No receiver detected
```

The wrapper captures the result:

```verilog
detect_rx_present <=
    (RxStatus == RXST_RX_DETECTED);

detect_done <= 1'b1;
```

### Returning the result to the LTSSM

The wrapper therefore converts the PIPE result into simpler MAC-side signals:

```text
detect_done
detect_rx_present
```

The LTSSM can then decide its next state.

Conceptually:

```text
LTSSM Detect.Active
        |
        v
mac_detect_req
        |
        v
PIPE Wrapper
        |
        | PowerDown  = P1
        | TxElecIdle = 1
        | TxDetectRx = 1
        v
PHY Receiver Detection
        |
        | PhyStatus
        | RxStatus
        v
PIPE Wrapper
        |
        | detect_done
        | detect_rx_present
        v
LTSSM
        |
        +---- Receiver detected ----> Polling
        |
        +---- No receiver ----------> Detect.Quiet
```

### Important distinction

`Detect.Active` and `P1` are not the same type of state.

```text
Detect.Active
    = PCIe LTSSM state

P1
    = PIPE PHY power state
```

The LTSSM determines that receiver detection is required.

The MAC/PIPE control logic then places the PHY in the required PIPE condition and requests receiver detection.

Therefore:

```text
LTSSM decides WHEN receiver detection is required.

PIPE wrapper generates the required PIPE controls.

PHY performs the actual electrical detection.

PIPE wrapper returns the result to the LTSSM.
```

### Key takeaway

The overall relationship is:

```text
LTSSM
  ↓
Requests Receiver Detection
  ↓
PIPE Wrapper
  ↓
P1 + TxElecIdle=1 + TxDetectRx=1
  ↓
PHY performs Receiver Detection
  ↓
PhyStatus + RxStatus
  ↓
PIPE Wrapper
  ↓
detect_done + detect_rx_present
  ↓
LTSSM selects the next state
```

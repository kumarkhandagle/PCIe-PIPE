## PIPE `TxDetectRx/Loopback` Interpretation

In PCIe PIPE mode, `TxDetectRx/Loopback` is a shared control signal. The PHY determines its meaning from the combination of:

```text
PowerDown
TxDetectRx/Loopback
TxElecIdle
```

Therefore, `TxDetectRx` must not be interpreted independently.

### Important PCIe-mode combinations

| `PowerDown`  | `TxDetectRx` | `TxElecIdle` | PHY interpretation  |
| ------------ | -----------: | -----------: | ------------------- |
| P0 (`2'b00`) |            0 |            0 | Normal transmission |
| P0 (`2'b00`) |            0 |            1 | Electrical Idle     |
| P0 (`2'b00`) |            1 |            0 | Loopback            |
| P0 (`2'b00`) |            1 |            1 | Illegal combination |
| P1 (`2'b10`) |            0 |            1 | PHY idle in P1      |
| P1 (`2'b10`) |            1 |            1 | Receiver Detection  |

### Loopback operation

For PHY loopback:

```text
PowerDown  = P0
TxDetectRx = 1
TxElecIdle = 0
```

The PHY interprets this combination as:

```text
Enable internal PHY loopback
```

Conceptually:

```text
MAC TxData
    |
    v
+---------+
|   PHY   |
|         |
|  TX --> RX
|   internal
|   loopback
+---------+
    |
    v
MAC RxData
```

This is why the MAC-side wrapper accepts loopback only when the PHY is in P0 and the transmitter is not electrically idle.

Example RTL condition:

```verilog
assign loopback_request_valid =
    (state == ST_IDLE) &&
    !starting_operation &&
    mac_loopback &&
    (PowerDown == P0) &&
    !mac_tx_elecidle;
```

### Receiver Detection

For receiver detection:

```text
PowerDown  = P1
TxDetectRx = 1
TxElecIdle = 1
```

The PHY interprets this combination as:

```text
Perform receiver detection
```

The PHY completes the operation using `PhyStatus`, while `RxStatus` provides the detection result.

Typical result interpretation:

```text
RxStatus = 3'b011 -> Receiver detected
RxStatus = 3'b000 -> No receiver detected
```

Example RTL condition:

```verilog
assign detect_allowed =
    (PowerDown == P1) &&
    TxElecIdle &&
    !loopback_mode;
```

### Why `loopback_mode` is required

`TxDetectRx = 1` alone does not indicate which operation is active.

For example:

```text
P0 + TxDetectRx=1 + TxElecIdle=0
    -> Loopback

P1 + TxDetectRx=1 + TxElecIdle=1
    -> Receiver Detection
```

Therefore the wrapper uses an internal flag:

```verilog
reg loopback_mode;
```

`loopback_mode` explicitly records that `TxDetectRx` currently belongs to the loopback function.

For loopback:

```verilog
TxDetectRx    <= 1'b1;
loopback_mode <= 1'b1;
```

For receiver detection:

```verilog
TxDetectRx    <= 1'b1;
loopback_mode <= 1'b0;
```

This prevents the wrapper from incorrectly assuming that every assertion of `TxDetectRx` represents loopback.

### Loopback active indication

The wrapper generates:

```verilog
assign mac_loopback_active =
    loopback_mode &&
    !TxElecIdle;
```

Therefore:

```text
mac_loopback_active = 1
```

means:

```text
TxDetectRx is currently assigned to the loopback function
AND
the PIPE transmitter is not electrically idle.
```


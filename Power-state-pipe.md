# PCIe PIPE Power Down Modes

## 1. Purpose

```text
PCIe LTSSM
    decides the link-level state transition

        ↓

MAC / LTSSM logic
    selects the required PIPE PHY power state

        ↓

PIPE PowerDown
    commands the PHY power state
```

PIPE power states are PHY operating states. They are not PCIe LTSSM states.

---

## 2. PIPE PHY Power States

For this wrapper:

```verilog
localparam [1:0] P0  = 2'b00;
localparam [1:0] P0S = 2'b01;
localparam [1:0] P1  = 2'b10;
localparam [1:0] P2  = 2'b11;
```

Conceptually:

| PIPE state | Meaning                                         |
| ---------- | ----------------------------------------------- |
| P0         | PHY fully operational                           |
| P0s        | Light PHY power-saving state                    |
| P1         | Deeper PHY power-saving state                   |
| P2         | Deeper power state; unsupported by this wrapper |

---

## 3. PIPE Legal Transition Topology

For PCIe PIPE operation, P0 is the common transition point.

Conceptually:

```text
                 P0s
                  ▲
                  │
                  ▼
                  P0
                /    \
               ▼      ▼
              P1      P2
```

The legal transition pairs are conceptually:

```text
P0  <-> P0s
P0  <-> P1
P0  <-> P2
```

Therefore, the following direct low-power-to-low-power transitions are not part of the normal legal transition set:

```text
P0s -> P1
P1  -> P0s

P0s -> P2
P2  -> P0s

P1  -> P2
P2  -> P1
```

The required method is to return through P0.

Examples:

```text
P0s -> P0 -> P1

P1  -> P0 -> P0s

P1  -> P0 -> P2

P2  -> P0 -> P1
```

---

## 4. Why P0 Is the Common Transition State

P0 is the normal operational PHY state.

Low-power states such as P0s, P1, and P2 may have different internal PHY resources disabled, reduced, gated, or placed into special operating conditions.

Allowing every low-power state to transition directly to every other low-power state would require the PHY to support many extra transition paths.

For example:

```text
P0s -> P1
```

would require the PHY to convert directly from one reduced-power configuration into another reduced-power configuration.

Instead, PIPE uses a simpler topology:

```text
P0s -> P0 -> P1
```

This gives the PHY a known operational baseline before entering the next low-power configuration.

A useful mental model is:

```text
P0 = rendezvous / baseline state

P0s = branch from P0
P1  = branch from P0
P2  = branch from P0
```

## 5. Why P0s -> P1 Is Not a Direct Jump

Assume:

```text
LTSSM condition corresponds to L0s
PHY state = P0s
```

Now the link eventually needs to enter an L1-related condition.

The conceptual link-level progression is:

```text
L0s
 │
 │ exit the L0s condition
 ▼
L0
 │
 │ perform L1 entry sequence
 ▼
L1
```

The associated PIPE PHY sequence becomes:

```text
P0s
 │
 │ PowerDown = P0
 ▼
P0
 │
 │ PowerDown = P1
 ▼
P1
```

Therefore:

```text
P0s -> P1
```

is not needed as a direct PIPE transition.

The correct sequence is:

```text
P0s -> P0 -> P1
```

---

## 6. Why P1 -> P0s Is Not a Direct Jump

The same principle applies in the reverse direction.

If the PHY is currently in P1 and the link later needs an L0s-related operating condition, the sequence returns through normal operation first.

Conceptually:

```text
L1
 │
 ▼
L0
 │
 ▼
L0s
```

Corresponding PIPE sequence:

```text
P1
 │
 ▼
P0
 │
 ▼
P0s
```

Therefore:

```text
P1 -> P0s
```

is not a direct legal jump.

The MAC requests:

```text
P1 -> P0
```

waits for completion, and later requests:

```text
P0 -> P0s
```

## 7. Legal Transition vs Allowed-Right-Now

These are two different concepts.

### 7.1 Legal transition

This asks:

```text
Is this state-to-state movement supported?
```

Examples:

```text
P0 -> P1      LEGAL

P0 -> P0s     LEGAL

P1 -> P0      LEGAL

P0s -> P0     LEGAL
```

### 7.2 Allowed right now

This asks:

```text
Even if the transition is legal,
are all current operating conditions safe?
```

For example:

```text
P0 -> P1
```

may be legal, but cannot start if:

```text
TxElecIdle = 0
```

or:

```text
loopback_mode = 1
```

Therefore:

```text
legal transition != immediately executable transition
```

---

## 8. Wrapper Legal-Transition Function

The current wrapper implements:

```verilog
function legal_power_transition;

    input [1:0] current_power;
    input [1:0] requested_power;

    begin
        case (current_power)

            P0: begin
                legal_power_transition =
                    (requested_power == P0S) ||
                    (requested_power == P1);
            end

            P0S: begin
                legal_power_transition =
                    (requested_power == P0);
            end

            P1: begin
                legal_power_transition =
                    (requested_power == P0);
            end

            default: begin
                legal_power_transition = 1'b0;
            end

        endcase
    end

endfunction
```

So the implemented wrapper transition table is:

| Current             | Requested | Result                  |
| ------------------- | --------- | ----------------------- |
| P0                  | P0s       | Legal                   |
| P0                  | P1        | Legal                   |
| P0s                 | P0        | Legal                   |
| P1                  | P0        | Legal                   |
| P0s                 | P1        | Illegal                 |
| P1                  | P0s       | Illegal                 |
| Any supported state | P2        | Illegal in this wrapper |
| P2                  | Any state | Illegal in this wrapper |

---

## 9. Example: P0s -> P1 Correct Sequence

### 9.1 Step 1

Current PHY state:

```text
P0s
```

MAC requests:

```text
mac_powerdown = P0
```

Transition:

```text
P0s -> P0
```

This is legal.

### 9.2 Step 2

Wrapper drives:

```text
PowerDown = P0
```

PHY completes the transition and asserts:

```text
PhyStatus = 1
```

Wrapper returns:

```text
power_done = 1
```

### 9.3 Step 3

The LTSSM/MAC performs whatever higher-level link sequencing is required before entering the next low-power condition.

### 9.4 Step 4

MAC ensures TX is electrically idle and requests:

```text
mac_powerdown = P1
```

### 9.5 Step 5

Wrapper performs:

```text
P0 -> P1
```

which is legal.

Overall:

```text
P0s -> P0 -> P1
```

---

## 10. Example: P1 -> P0s Correct Sequence

The same rule applies:

```text
P1 -> P0 -> P0s
```

not:

```text
P1 -> P0s
```

Sequence:

```text
P1
 │
 │ request P0
 ▼
P0
 │
 │ complete required LTSSM/link behavior
 │
 │ enter TX electrical idle if required
 ▼
P0s
```

---

## 11. Electrical-Idle Requirement

### 11.1 Entering P0s or P1

Before the wrapper accepts a transition from P0 into a low-power state, the transmitter shall already be in electrical idle.

The MAC shall first assert:

```verilog
mac_tx_elecidle = 1'b1;
```

The wrapper shall then cause:

```verilog
TxElecIdle = 1'b1;
```

Only after `TxElecIdle` is high may a transition into P0s or P1 be accepted.

Required sequence:

```text
1. MAC asserts mac_tx_elecidle.
2. Wrapper asserts TxElecIdle.
3. PHY transmitter enters electrical idle.
4. MAC requests mac_powerdown = P0s or P1.
5. Wrapper validates the request.
6. Wrapper drives PowerDown = requested state.
7. PHY performs the power transition.
```

If `mac_powerdown` requests P0s or P1 while `TxElecIdle == 0`, the request shall be blocked.

### 11.2 Returning to P0

A transition to P0 does not require `TxElecIdle` to be asserted as an explicit launch prerequisite.

The wrapper shall keep the transmitter idle while the PHY is in a non-P0 state or while a power operation is pending.

---

## 12. Loopback Restriction

Power-state changes shall not be started while loopback mode is active.

The wrapper shall require:

```verilog
loopback_mode == 1'b0
```

before accepting a power-state transition.

Reason:

Loopback configures the PHY into a special operating mode that depends on stable PHY datapath and control behavior. Changing the PHY power state while loopback is active can interfere with loopback operation and with shared PHY control behavior such as `TxDetectRx`.

Therefore:

```text
loopback_mode = 0 -> power change may proceed

loopback_mode = 1 -> power change shall be blocked
```

The MAC shall disable loopback before requesting a power-state change.

---

## 13. Power-Change Qualification Rule

A power-state change is allowed only when all required conditions are true.

```verilog
assign power_change_allowed =
    legal_power_transition(PowerDown, mac_powerdown) &&
    ((mac_powerdown == P0) || TxElecIdle) &&
    !loopback_mode;
```

This expression has three independent requirements.

### 13.1 Legal Transition

```verilog
legal_power_transition(PowerDown, mac_powerdown)
```

The requested current-state-to-target-state transition shall be supported by the wrapper.

### 13.2 Electrical Idle

```verilog
(mac_powerdown == P0) || TxElecIdle
```

Interpretation:

```text
Target = P0:
    TxElecIdle is not a launch prerequisite.

Target = P0s or P1:
    TxElecIdle must be 1.
```

### 13.3 Loopback Disabled

```verilog
!loopback_mode
```

Interpretation:

```text
loopback_mode = 0 -> condition passes
loopback_mode = 1 -> condition fails
```

---

```mermaid
stateDiagram-v2

    [*] --> ST_RESET

    ST_RESET --> ST_RESET : PHY Initialization
    ST_RESET --> ST_IDLE : PhyStatus = 0

    ST_IDLE --> ST_WAIT : start_power_operation

    ST_WAIT --> ST_IDLE : PhyStatus = 1
    ST_WAIT --> ST_WAIT : Waiting for PHY

    ST_WAIT --> ST_FAULT : Timeout

    ST_FAULT --> ST_FAULT : Reset required

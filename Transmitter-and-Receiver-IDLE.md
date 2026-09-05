# PCIe PIPE Electrical Idle: TxElecIdle, RxElecIdle, and Recovery.Speed


> **TxElecIdle controls the local transmit direction. RxElecIdle reports the electrical condition of the local receive direction.**

PCIe is full-duplex, so the two directions are independent.

---

## 1. The Two Directions of a PCIe Link

Consider two PCIe devices, Device A and Device B.

```text
Device A                                      Device B

TX_A  ------------------------------------->  RX_B

RX_A  <-------------------------------------  TX_B
```

There are therefore two independent electrical paths:

```text
A -> B

B -> A
```

<img width="1232" height="355" alt="image" src="https://github.com/user-attachments/assets/e09fbd83-eba8-4c1b-8e53-e8f2ae86d155" />


Device A can stop transmitting while Device B is still transmitting.

Therefore, Device A needs two different pieces of information:

```text
TxElecIdle_A
    "What am I asking my own transmitter to do?"

RxElecIdle_A
    "What electrical condition do I see on the signal coming from B?"
```

This is why PIPE has both signals.

---

# 2. TxElecIdle

## 2.1 Direction

`TxElecIdle` is driven from the MAC toward the PHY.

```text
MAC
 |
 | TxElecIdle
 v
PHY transmitter
 |
 v
TX+ / TX-
```

It is a **command/control signal**.

---

## 2.2 Meaning

When:

```text
TxElecIdle = 1
```

the MAC is telling the PHY:

```text
"Place my transmitter into Electrical Idle."
```

The PHY stops normal high-speed differential transmission and drives the transmitter into the electrical-idle condition defined by the PHY/PCIe electrical requirements.

When:

```text
TxElecIdle = 0
```

and the PHY is in an appropriate active state such as P0, the transmitter can send the parallel data presented by the MAC.

A useful memory rule is:

```text
TxElecIdle = 1
    "I am making MY transmitter electrically idle."
```

---

# 3. RxElecIdle

## 3.1 Direction

`RxElecIdle` travels in the opposite direction.

```text
RX+ / RX-
   |
   v
PHY receiver
   |
   | RxElecIdle
   v
MAC
```

It is a **status indication** produced by the PHY.

---

## 3.2 Meaning

When:

```text
RxElecIdle = 1
```

the PHY is reporting:

```text
"Electrical Idle has been detected on my receive lane."
```

It does **not** mean:

```text
"My receiver circuit is powered off."
```

The receiver may still be powered, active, and continuously monitoring the RX pins.

A better conceptual name would be:

```text
rx_electrical_idle_detected
```

rather than:

```text
receiver_is_idle
```

---

## 3.3 What the PHY Is Detecting

The PHY receiver monitors the differential signal on:

```text
RX+
RX-
```

Conceptually, the receive front end contains an electrical-idle or squelch detector.

```text
RX+ / RX-
    |
    v
Analog receive front end
    |
    +--------------------------+
    |                          |
    v                          v
CDR / deserializer       Electrical-idle detector
                               |
                               v
                          RxElecIdle
```

When sufficient high-speed differential activity is detected:

```text
RxElecIdle = 0
```

When the signal satisfies the PHY's electrical-idle detection condition:

```text
RxElecIdle = 1
```

The exact analog detection implementation is PHY-dependent.

---

# 4. RxElecIdle Does Not Mean "Valid Data"

A very important distinction is:

```text
RxElecIdle
```

and:

```text
RxValid
```

do not mean the same thing.

`RxElecIdle` asks:

```text
"Is the incoming lane electrically idle?"
```

`RxValid` asks:

```text
"Is valid receive data/symbol information available?"
```

Therefore this condition can occur:

```text
RxElecIdle = 0
RxValid    = 0
```

Meaning:

```text
Electrical activity is present,
but valid aligned receive data is not yet available.
```

A possible sequence is:

```text
Remote transmitter starts
        |
        v
Electrical activity appears
        |
        v
RxElecIdle -> 0
        |
        v
CDR locks / receive processing stabilizes
        |
        v
RxValid -> 1
```

---

# 5. TxElecIdle and RxElecIdle Observe Different Directions

For Device A:

```text
Device A                                    Device B

          TxElecIdle_A
MAC_A -----------> PHY_TX_A
                       |
                       | A -> B
                       +----------------------> PHY_RX_B
                                                  |
                                                  v
                                            RxElecIdle_B
```

If Device A asserts:

```text
TxElecIdle_A = 1
```

then eventually Device B may detect:

```text
RxElecIdle_B = 1
```

Therefore:

```text
A.TxElecIdle
```

and:

```text
B.RxElecIdle
```

can describe the same physical A-to-B electrical-idle event from opposite ends.

But:

```text
A.TxElecIdle
```

and:

```text
A.RxElecIdle
```

describe two different directions.

For Device A:

```text
TxElecIdle_A
    controls A -> B

RxElecIdle_A
    observes B -> A
```

---

# 6. Why RxElecIdle Is Needed If TxElecIdle Already Exists

Because PCIe is full-duplex.

Suppose Device A does:

```text
TxElecIdle_A = 1
```

Then:

```text
A -> B becomes electrically idle.
```

But Device B may still be transmitting toward A:

```text
B -> A remains active.
```

Therefore Device A could have:

```text
TxElecIdle_A = 1
RxElecIdle_A = 0
```

This means:

```text
A has stopped transmitting,
but A still sees B transmitting.
```

Later B may also enter Electrical Idle.

Then A may see:

```text
RxElecIdle_A = 1
```

Now both directions are electrically quiet from A's point of view.

```text
A -> B : local TX is idle

B -> A : local RX detects Electrical Idle
```

---

# 7. RxElecIdle Is Not Receiver Detection

PCIe receiver detection in the LTSSM `Detect` state is a different mechanism.

## Receiver Detection

Receiver detection asks:

```text
"Is a receiver termination physically present at the far end?"
```

Typical PIPE mechanism:

```text
LTSSM Detect.Active
        |
        v
TxDetectRx request
        |
        v
PHY performs receiver detection
        |
        v
PhyStatus / RxStatus
        |
        v
Receiver present or not present
```

## RxElecIdle

RxElecIdle asks:

```text
"Is the incoming receive lane currently detected as electrically idle?"
```

Comparison:

| Concept | Main Question |
|---|---|
| Receiver Detection | Is a receiver termination present at the far end? |
| RxElecIdle | Is the incoming lane electrically idle? |
| RxValid | Is valid receive data available? |
| PowerDown | What operating/power state is the PHY being requested to use? |

---

# 8. RxElecIdle Is Not the Receiver Power State

Another important distinction is between:

```text
RxElecIdle
```

and:

```text
PowerDown
```

If the PHY is in P1, the receiver may be unavailable for normal receive processing.

The MAC already knows this from the PHY power-state control/handshake.

Conceptually:

```text
PowerDown = P1
        |
        v
PHY transitions to P1
        |
        v
PhyStatus indicates completion
        |
        v
MAC treats normal receiver operation as inactive
```

You should not use:

```text
RxElecIdle = 1
```

to mean:

```text
"The receiver is powered down."
```

A useful implementation is:

```verilog
wire rx_receiver_active;

assign rx_receiver_active =
    (PowerDown == P0) ||
    (PowerDown == P0S);
```

Then normal receive status can be qualified with receiver availability.

---

# 9. Synchronizing RxElecIdle

In many PIPE implementations, `RxElecIdle` is asynchronous to `pclk`.

Therefore a MAC-side wrapper may synchronize it.

```text
RxElecIdle
    |
    v
rx_ei_meta
    |
    v
rx_ei_sync
    |
    v
optional stability filter
    |
    v
mac_rx_elecidle
```

Example:

```verilog
reg       rx_ei_meta;
reg       rx_ei_sync;
reg [7:0] rx_ei_counter;

always @(posedge pclk or negedge reset_n) begin

    if (!reset_n) begin

        rx_ei_meta      <= 1'b1;
        rx_ei_sync      <= 1'b1;
        rx_ei_counter   <= 8'd0;
        mac_rx_elecidle <= 1'b1;

    end else begin

        rx_ei_meta <= RxElecIdle;
        rx_ei_sync <= rx_ei_meta;

        if (rx_ei_sync == mac_rx_elecidle) begin

            rx_ei_counter <= 8'd0;

        end
        else if (RX_EI_FILTER_CYCLES == 0) begin

            mac_rx_elecidle <= rx_ei_sync;
            rx_ei_counter   <= 8'd0;

        end
        else if (rx_ei_counter >=
                 (RX_EI_FILTER_CYCLES - 1)) begin

            mac_rx_elecidle <= rx_ei_sync;
            rx_ei_counter   <= 8'd0;

        end
        else begin

            rx_ei_counter <= rx_ei_counter + 8'd1;

        end
    end
end
```

---

# 10. Purpose of the Two Synchronizer Flops

The first stage:

```verilog
rx_ei_meta <= RxElecIdle;
```

is the metastability-catching stage.

The second stage:

```verilog
rx_ei_sync <= rx_ei_meta;
```

provides a synchronized value for functional logic.

Functional logic should use:

```text
rx_ei_sync
```

not:

```text
RxElecIdle
```

and not:

```text
rx_ei_meta
```

Conceptually:

```text
asynchronous domain
       |
       v
RxElecIdle
       |
       v
+-------------+
| rx_ei_meta  |   first CDC stage
+-------------+
       |
       v
+-------------+
| rx_ei_sync  |   functional synchronized value
+-------------+
       |
       v
MAC logic
```

---

# 11. Purpose of the Optional Filter

The synchronizer solves a CDC/metastability problem.

The counter solves a different problem:

```text
short changes / chatter / glitches
```

For example:

```text
Expected accepted value = 0

rx_ei_sync:

0 0 0 1 1 0 0 0
      -----
      short excursion
```

If three stable samples are required, the short excursion can be rejected.

```text
rx_ei_sync        0  1  1  0
counter            0  1  2  0
mac_rx_elecidle   0  0  0  0
```

The counter effectively measures:

> How many consecutive samples has `rx_ei_sync` remained different from the currently accepted `mac_rx_elecidle` value?

---

# 12. Meaning of RX_EI_FILTER_CYCLES

A clean convention is:

```text
RX_EI_FILTER_CYCLES = 0
    no additional filtering

RX_EI_FILTER_CYCLES = 1
    one synchronized observation is sufficient

RX_EI_FILTER_CYCLES = N
    require N consecutive synchronized observations
```

This filter value is an implementation choice.

It is **not** a PCIe rule saying that Electrical Idle must remain present for 3, 4, 32, or any other fixed number of `pclk` cycles.

The selected value should be justified by:

```text
target PHY behavior
+
CDC architecture
+
desired glitch rejection
+
LTSSM timing budget
```

---

# 13. Where the MAC Uses RxElecIdle

The MAC does not normally use `RxElecIdle` as packet data.

Instead it provides physical-layer state information to receive-control and LTSSM logic.

Conceptually:

```text
RxElecIdle
    |
    v
synchronizer/filter
    |
    v
mac_rx_elecidle
    |
    +----------------------+
                           |
                           v
                       MAC/LTSSM
                           ^
                           |
          +----------------+----------------+
          |                                 |
       RxValid                       ordered-set logic
          |                                 |
       RxData                        TS1 / TS2 / EIOS /
       RxDataK                       EIEOS recognition
```

The LTSSM interprets these observations according to the current PCIe state.

---

# 14. Electrical Idle in Recovery.Speed

`Recovery.Speed` is an important example because the link intentionally stops old-rate signaling before changing signaling rate.

For a simplified Gen1-to-Gen2 speed transition:

```text
Recovery.RcvrCfg
      |
      | speed change negotiated
      v
Recovery.Speed
      |
      v
send required Electrical Idle Ordered Set (EIOS)
      |
      v
command local transmitter to Electrical Idle
      |
      v
establish/recognize required receive-side Electrical Idle condition
      |
      v
request PHY rate change
      |
      v
wait for PHY rate-change completion
      |
      v
resume Recovery operation at the new rate
```

The exact transition criteria and timers are defined by the applicable PCIe Base Specification.

---


EIOS is a PCIe protocol object sent over the serial link. TxElecIdle is a PIPE control signal that tells the local PHY to physically stop differential transmission.

A useful sequence is:

MAC / LTSSM
    |
    | 1. Send EIOS as normal transmit symbols
    v
PIPE TxData / TxDataK
    |
    v
PHY serializes EIOS
    |
    v
Remote device receives EIOS

then

MAC / LTSSM
    |
    | 2. Assert TxElecIdle
    v
PIPE TxElecIdle = 1
    |
    v
PHY actually drives TX+/TX- into Electrical Idle

So EIOS means roughly:

"I am about to enter Electrical Idle."

Whereas TxElecIdle = 1 means:

"PHY, now actually place the transmitter into the electrical-idle condition."

The PHY does not normally inspect the outgoing byte stream and say, “I saw EIOS, therefore I should shut down my analog transmitter.” PIPE specifically separates those responsibilities. The PIPE specification notes that EIOS transmission is part of the normal transmit pipeline and should not require the PHY itself to detect the EIOS pattern; afterward TxElecIdle controls the physical transition.

AMD likewise defines phy_txelecidle as the command that forces TX+/TX− to Electrical Idle; when asserted, the PHY drives the differential outputs to the electrical-idle/common-mode condition.

So imagine Device A:

Device A MAC                 Device A PHY               Device B

Send EIOS
    |
    +---- TxData/TxDataK -----> serialize EIOS ---------> receives EIOS
                                      |
                                      |
Assert TxElecIdle --------------------+
                                      |
                                      v
                               TX physically idle ------> detects idle

If you sent EIOS but never asserted TxElecIdle, the remote side could receive EIOS, but your PHY could continue electrically transmitting whatever subsequently appears on TxData.

In other words:

EIOS alone
    = protocol notification

TxElecIdle alone
    = physical action

A correct sequence needs both:

EIOS
  ↓
"Tell partner what is about to happen"

TxElecIdle = 1
  ↓
"Actually make it happen electrically"

This separation is useful because the responsibilities are clean:

LTSSM / MAC
    knows PCIe protocol
    generates EIOS
    decides WHEN to enter idle

PHY
    knows analog transmitter implementation
    responds to TxElecIdle
    physically removes differential signaling


    EIOS
= protocol-level indication sent to the remote device
  that this transmit direction is transitioning into Electrical Idle.

TxElecIdle
= local PIPE command telling your own PHY:
  "Now physically place the transmitter into Electrical Idle."

So the sequence is:

MAC/LTSSM
   |
   | transmit EIOS through TxData/TxDataK
   v
Local PHY
   |
   | serializes EIOS
   v
Remote receiver
   |
   | understands the protocol transition
   v

then

MAC/LTSSM
   |
   | TxElecIdle = 1
   v
Local PHY
   |
   | physically stops normal differential signaling
   v
TX lane enters Electrical Idle


If we removed EIOS, the remote side would see only:

normal traffic
    ↓
electrical activity disappears
    ↓
RxElecIdle = 1

But from that observation alone, the receiver cannot immediately distinguish among cases such as:

1. Partner intentionally entered Electrical Idle
2. Signal was lost because of noise/channel problem
3. CDR temporarily lost lock
4. Partner reset or disappeared
5. A protocol-defined idle transition is occurring

EIOS solves that ambiguity.

The intended sequence is:

Remote receives valid PCIe symbols
        ↓
Remote receives EIOS
        ↓
Remote MAC understands:
"an intentional Electrical Idle transition is beginning"
        ↓
transmitter actually stops signaling
        ↓
RxElecIdle / inferred-idle indication follows

So EIOS is analogous to an orderly protocol notification, while disappearance of electrical activity is the physical event.



# 15. Recovery.Speed Viewed from Device A

Consider Device A.

First, Device A handles its transmit direction.

```text
Device A LTSSM
      |
      v
send EIOS
      |
      v
mac_tx_elecidle = 1
      |
      v
TxElecIdle_A = 1
      |
      v
A's PHY transmitter enters Electrical Idle
```

Now consider Device A's receive direction.

```text
Device B enters Electrical Idle
      |
      v
B stops normal signaling toward A
      |
      v
A PHY detects the receive electrical-idle condition
      |
      v
RxElecIdle_A = 1
      |
      v
mac_rx_elecidle_A = 1
```

Therefore Device A has two different observations:

```text
Tx side:
I commanded my transmitter idle.

Rx side:
I detect/infer the incoming direction is idle.
```

---

# 16. Recovery.Speed Viewed from Both Devices

A simplified conceptual picture is:

```text
Device A                                      Device B

send EIOS                                    send EIOS
    |                                            |
    v                                            v
TxElecIdle_A = 1                           TxElecIdle_B = 1
    |                                            |
    |         A -> B goes idle                   |
    +------------------------------------------->|
                                                 |
                                      RxElecIdle_B / idle inference


                                                 |
    |         B -> A goes idle                   |
    |<-------------------------------------------+
    |
RxElecIdle_A / idle inference
```

Each side controls its own TX direction and observes the opposite RX direction.

This is the key reason both TX and RX electrical-idle information exist.

---

# 17. Does RxElecIdle Prove That EIOS Was Received?

No.

`RxElecIdle = 1` means:

```text
"The receive lane is detected as electrically idle."
```

It does not by itself mean:

```text
"I successfully decoded EIOS immediately before the lane went idle."
```

These are different observations.

A MAC may have:

```text
rx_eios_detected
```

from receive ordered-set decoding, and separately:

```text
mac_rx_elecidle
```

from the PHY electrical-idle indication.

Conceptually:

```text
RxData / RxDataK
      |
      v
EIOS recognition
      |
      v
rx_eios_detected


RX electrical condition
      |
      v
RxElecIdle
      |
      v
mac_rx_elecidle
```

Together, and interpreted in the correct LTSSM state, they provide stronger protocol information.

---
# 18. Better Recovery.Speed Architecture

Conceptually:

```text
               Recovery.Speed
                     |
                     v
          required protocol conditions
                     |
                     v
                 send EIOS
                     |
                     v
         assert local TxElecIdle
                     |
                     v
     determine receive Electrical Idle
      using applicable PCIe rules
                     |
                     v
       idle conditions satisfied
                     |
                     v
             request new Rate
                     |
                     v
             wait for PhyStatus
                     |
                     v
        continue Recovery at new rate
```

This keeps the responsibilities clean:

```text
LTSSM
    decides WHEN

MAC receive logic
    interprets WHAT WAS RECEIVED

PIPE wrapper
    transports/conditions controls and status

PHY
    performs electrical transmission/reception
    and physical rate/power operations
```

---

# 19. Relationship to Power States

Electrical Idle and PHY power state are related but are not identical.

For example:

```text
TxElecIdle = 1
```

means the transmitter is commanded electrically idle.

It does not automatically mean:

```text
PowerDown = P1
```

Similarly:

```text
RxElecIdle = 1
```

does not mean the receiver is powered down.

A useful separation is:

```text
TxElecIdle
    condition of local TX signaling

RxElecIdle
    detected condition of incoming RX signaling

PowerDown
    requested operating/power state of PHY

PhyStatus
    completion indication for PHY operations
```

---

# 20. Complete Mental Model

For one PCIe device:

```text
                         LTSSM
                           |
              +------------+-------------+
              |                          |
              v                          |
       mac_tx_elecidle                   |
              |                          |
              v                          |
         PIPE wrapper                    |
              |                          |
              v                          |
         TxElecIdle                      |
              |                          |
              v                          |
          Local PHY TX                   |
              |                          |
==============|====== PCIe Link =========|==============
              |                          |
          Local PHY RX                   |
              |                          |
              v                          |
         RxElecIdle                      |
              |                          |
              v                          |
        CDC synchronizer                 |
              |                          |
              v                          |
       optional filter                   |
              |                          |
              v                          |
      mac_rx_elecidle -------------------+
```

In one sentence:

> **The LTSSM uses TxElecIdle to control whether its own transmitter is electrically active, while it uses RxElecIdle as physical-layer information about whether the opposite transmit direction is electrically idle.**

---

# 21. Quick Comparison Table

| Signal / Function | Direction | Meaning |
|---|---|---|
| `TxElecIdle` | MAC -> PHY | Command local TX into Electrical Idle |
| `RxElecIdle` | PHY -> MAC | RX Electrical Idle detected on incoming lane |
| `RxValid` | PHY -> MAC | Valid receive data/symbol information is available |
| `TxDetectRx` | MAC -> PHY | Request receiver-termination detection |
| `RxStatus` | PHY -> MAC | Receive status/errors; also carries receiver-detect result during detection |
| `PowerDown` | MAC -> PHY | Request PHY power/operating state |
| `PhyStatus` | PHY -> MAC | Completion of certain PHY operations such as rate/power/detect |

---


# 22. Reference Basis

This chapter is written as an implementation-oriented teaching reference for a Gen1/Gen2 PIPE-style MAC/PHY interface. Signal semantics were cross-checked against:

- AMD PCI Express PHY Product Guide **PG345**, Status Signals Interface Ports (`phy_rxelecidle`, `phy_rxvalid`, `phy_phystatus`).
- AMD PCI Express documentation **PG343**, Command Signals (`phy_txelecidle`, `phy_txdetectrx`, `phy_powerdown`, `phy_rate`).
- AMD PCIe PIPE per-lane interface documentation **PG054**, including `PIPERXnELECIDLE` and `PIPETXnELECIDLE`.
- PCI Express PIPE Architecture Specification for the general MAC/PHY interface model and Gen1/Gen2 electrical-idle behavior.
- The PCI Express Base Specification for exact LTSSM `Recovery.Speed` entry/exit criteria, Electrical Idle inference rules, and protocol timing.

For implementation, always use the PIPE revision and PCIe Base Specification revision applicable to the selected PHY/IP.

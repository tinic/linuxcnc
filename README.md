# linuxcnc
Personal LinuxCNC configurations

## G0759: homing on the glass scales' reference marks

`configs/G0759/G0759-coffeecnc.ini` homes X, Y and Z on the reference (R) mark of each
glass scale. The scales are read by an [rp2040-encoder](https://github.com/tinic/rp2040-encoder)
board (firmware and HAL component v3.0.0 or later), and the panel is
[CoffeeCNC](https://github.com/tinic/coffeecnc). `configs/G0759/G0759.ini` (gmoccapy) still homes
in place.

Each scale has one mark in its travel and the machine has no home or limit switches, so at
power-up nothing says on which side of its mark an axis stands. `src/refmark.comp` remembers
where a mark is once the encoder board has seen it and then acts as a home switch that is closed
on one whole side of the mark. LinuxCNC's switch homing with an index finds the mark from either
side and creeps the last 3 mm onto it, always in the + direction.

### Install

```sh
sudo halcompile --install src/refmark.comp
```

### Using it

1. After the controller has been powered up, move each axis across its mark once: jog it, or
   turn the handwheel with the machine off. `refmark.N.ready` becomes true; CoffeeCNC's Home
   button changes from "find mark" to "not homed", and its letter is red while the axis is
   within 2 mm of the mark.
2. Switch the machine on and home the axis, or all axes (Z, then X, then Y).
3. The axis ends on its mark and reads 0.000.

A mark stays known across restarts of LinuxCNC for as long as the encoder board is powered.
An axis that stands on its mark when the board starts must leave the mark and cross it again:
the board counts the rising edge of the pulse.

Z's mark is 172 mm below the upper stop: homing Z from above lowers the head to about 3 mm
below the mark before it creeps up onto it.

### `refmark` pins

One instance per axis: `loadrt refmark names=refmark.x,refmark.y,refmark.z`, each added to the
servo thread ahead of the motion functions. `configs/G0759/refmark_homing.hal` has the wiring.

| Pin | Type | Default | Connected to | Meaning |
|-----|------|---------|--------------|---------|
| `position` | float in | | the joint's position feedback (`x-pos-fb`) | Encoder position, in the units and sign of `joint.N.motor-pos-fb` |
| `ref-count` | u32 in | | `rp2040-encoder.0.ref-count-N` | Reference pulses the encoder board has seen |
| `ref-position` | float in | | `rp2040-encoder.0.ref-position-N` | Encoder position latched at the latest pulse |
| `connected` | bit in | | `rp2040-encoder.0.connected` | The encoder board is connected and counting |
| `homing` | bit in | | `joint.N.homing` | The joint is homing |
| `index-enable` | bit io | | `joint.N.index-enable` (and `pid.N.index-enable`) | Index handshake: cleared here in the servo period in which `ref-count` changes while the joint waits for the index |
| `home-sw` | bit out | | `joint.N.home-sw-in` | The derived home switch: true when the mark is known and `position > mark - approach` (`position < mark + approach` with `reverse`) |
| `ready` | bit out | | `coffeecnc.reference-mark-N` | The mark is known and the board is connected: homing can find it from either side |
| `at-mark` | bit out | | `coffeecnc.at-reference-mark-N` | `ready`, and the axis is within `window` of the mark |
| `fault` | bit out | | `joint.N.amp-fault-in` | Homing was started while the mark is not known, or the index search has passed the mark by `approach` without a pulse |
| `mark` | float out | | | Encoder position of the mark in use |
| `approach` | float in | 3.0 | | Distance before the mark at which the home switch closes, in machine units |
| `window` | float in | 2.0 | | Half width of the zone in which `at-mark` is true |
| `tolerance` | float in | 3.0 | | Largest believable distance between a latched mark and the axis position at the moment its pulse is seen |
| `reverse` | bit in | 0 | | Search and final approach in the − direction |

When a mark is taken over:

- At a pulse seen while LinuxCNC runs, the latched position must lie within `tolerance` of
  where the axis is at that moment; the component waits up to 2 s for a latched position that
  fits. A mark that does not fit makes the axis not ready until it crosses the mark again.
- A count that is already there when the encoder board connects (LinuxCNC was restarted, the
  board kept counting) is taken over with its latched position, unless that position is
  exactly 0, which means the board has not delivered it.
- A count that goes back to 0 means the board has restarted: the mark is forgotten.

### INI and HAL settings

`[JOINT_n]` of `G0759-coffeecnc.ini`:

| Key | Value | Why |
|-----|-------|-----|
| `HOME_SEARCH_VEL` | 5.0 | Search for the edge of the derived switch, in +; from the + side LinuxCNC first backs off in − |
| `HOME_LATCH_VEL` | 0.5 | Same sign as the search: the last approach, onto the switch edge and on to the mark, is in + |
| `HOME_USE_INDEX` | YES | The mark is the index |
| `HOME_INDEX_NO_ENCODER_RESET` | YES | The scale count is not reset at the mark |
| `HOME_OFFSET`, `HOME` | 0.0 | The mark is machine zero, and the axis stays there after homing |
| `HOME_SEQUENCE` | Z 0, X 1, Y 2 | Order of "home all" |
| `MIN_LIMIT`, `MAX_LIMIT` | X −222.11 / 177.855, Y −33.375 / 102.77, Z −87.15 / 170.37 | 2 mm inside the hard stops (also in `[AXIS_*]`) |

`[COFFEECNC] REFERENCE_MARKS = yes` tells CoffeeCNC that `coffeecnc_g0759_postgui.hal` connects
`ready` and `at-mark`: it refuses Home for an axis whose mark is not known and shows the axis
that stands at its mark.

`custom.hal`: `rp2040-encoder.0.sign-N` is −1 on all three axes (v3.0.0 replaced the
component's fixed inversion with this pin), and the encoder count is no longer zeroed when
LinuxCNC starts. Do not use `rp2040-encoder.0.reset-N` on this machine: the latched mark
positions are in that count.

### Measured on 22 November 2026

Each axis was driven to both hard stops by hand; the scales count 0.005 mm.

| Axis | Travel between hard stops | Mark, approached in + | Hard stops in machine coordinates |
|------|---------------------------|-----------------------|-----------------------------------|
| X | 403.965 mm | 224.110 mm from the negative stop | −224.110 and +179.855 |
| Y | 140.145 mm | 35.375 mm from the negative stop | −35.375 and +104.770 |
| Z | 261.520 mm | 172.370 mm below the upper stop | −89.150 and +172.370 |

The latched position of a mark differs by 0.02 to 0.03 mm between the two directions of
travel. All three axes were homed on the machine and stood on their latched marks to the
count afterwards. A first homing after a restart of the encoder board also went through:
the marks were found again by moving each axis across its mark, and all three axes homed.

One crossing of a mark is not always one pulse: while a program moved X slowly over its
mark, `rp2040-encoder.0.ref-count-0` rose by about 250 within 0.1 mm. `refmark` only looks
for a change of the count, clears the index at the first one, and keeps the position of the
last edge as the mark, so that the mark it uses can lie up to about 0.07 mm from the edge
homing stopped on. That does not matter for the derived switch, which closes 3 mm before
the mark.

Not exercised: the fault for an index search that passes its mark, and a stall during a
home move (the scales are the position feedback, so that should end in a following error).

## VFD

Notes on the SVD-PS or SVD-P VFD (LAPOND, Shenzhen Soyanpower New Energy Co.,Ltd)

Docs: http://www.soyan-tech.com/uploads/20189675/PDF/SVD-p-User-Manual.pdf

Only the following settings for Modbus seem to work (for instance changing the default baud rate will break communication)

```
PD-00 6005
PD-01 0
PD-02 1
PD-03 2
PD-04 0
PD-05 1 <---- Note the change from the default settings
PD-06 0
PD-08 0
```

- Serial settings: 9600 baud, 8N2
- Slave address is 1
- All modbus registers are 16-bit.

Modbus read/write registers:

```
0x1000 { 0-10000: 0-100% speed in %/100}
```


Modbus read registers:

```
0x1001 { 0-12000: running frequency in Hz/100 }
0x1002 { 0-800: DC BUS voltage V/10 }
0x1003 { 0-800: Output voltage V }
0x1004 { 0-800: Output current A/100 }
0x1005 { 0-10000: 0-100% running speed in %/10 }
0x1006 { 0-10000: output power }
0x1009 { 0-65535: input terminal }
0x100a { 0-1000: AI1 }
0x100b { 0-1000: AI2? }
```

Modbus write registers:

```
0x2000 { 1: Forward run
         2: Reverse run
         3: Forward JOG
         4: Reverse JOG
         5: Stop
       }
```

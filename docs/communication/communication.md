# Communication

The Hyper-K Trigger Timing Distributor (HKTTD) uses two communication mechanisms.
[SiTCP](https://www.sitcp.net/) connects a computer to the Main and Sub modules, while the MIKUMARI data-interface multi-channel interface (MMCI) carries data between the Main and Sub modules.

The HKTTD communication configuration is currently under development.
Many of the current register and channel assignments are intended for system testing and may be removed or simplified in future revisions.

## SiTCP

SiTCP provides data transfer and register access between each module and the computer.

Transmission Control Protocol (TCP) uses a first-in, first-out (FIFO)-like interface.
The FPGA writes data to the transmit FIFO, and SiTCP transfers the data to the computer.

Remote Bus Control Protocol (RBCP) allows user software to read and write registers in the FPGA.
These registers are used to configure the FPGA and monitor its status.

HKTTD uses a 16-bit address composed of a 4-bit module ID and a 12-bit local address:

| Bits | Description |
| --- | --- |
| 15–12 | Module ID |
| 11–0 | Local address |

The register maps below list only the modules specific to HKTTD.
Registers provided by the AMANEQ platform are outside the scope of this guide.

### Main module

#### TCP FIFO configuration

The Main firmware implements one SiTCP connection.
Its TCP transmit FIFO sends the measured phase-shift value for MIKUMARI link 0 when the value changes.
Each transmitted value is a signed two's-complement 32-bit fixed-point value with 25 fractional bits and can be positive or negative.
Multiply the signed value by `8 ns / 2^25` to obtain the measured phase shift.
The value is transmitted in big-endian byte order, with the most-significant byte first.

#### HKTTD register map

The Main `BusController` uses the following module ID assignments:
The assignments are defined in `HkttdMain/hdl/defBusAddressMap.vhd`.

| Module ID | Constant | Destination |
| --- | --- | --- |
| `0x0` | `kHPP` | `HkttdPulseEventPublisher` |
| `0x1`–`0x8` | `kHDL1`–`kHDL8` | `HkttdDeliverLink` 1–8 |

`kHDL1` through `kHDL8` correspond to CDD-OPT optical ports 0 through 7, respectively.

##### HkttdPulseEventPublisher registers

The `HkttdPulseEventPublisher` registers are assigned to Main module ID `0x0`.
Their local addresses are defined in `hkttd-hdl/hkttdCore/defHkttdPulseEventPublisher.vhd`.

| Local address | Access | Size | Description |
| --- | --- | --- | --- |
| `0x000` | Read/write | 1 byte | Enable the external signal pulse and dummy pulse |
| `0x100` | Read/write | 4 bytes | Dummy-pulse period |

The control register at local address `0x000` uses the following bit assignments:

| Bit | Description |
| --- | --- |
| 0 | Enable the external signal pulse (`0`: disabled, `1`: enabled) |
| 1 | Enable the dummy pulse (`0`: disabled, `1`: enabled) |
| 7–2 | Unused |

The dummy-pulse period at local address `0x100` is an unsigned 32-bit value with a unit of 1 ns.
Its default value is `0x000186A0`, corresponding to 100 µs.

##### HkttdDeliverLink registers

Each link uses the same local address map.
The local addresses are defined in `hkttd-hdl/hkttdCore/defHkttdDeliverLink.vhd`.

| Local address | Access | Size | Description |
| --- | --- | --- | --- |
| **Pulse counters** |  |  |  |
| `0x000` | Write | 1 byte | Reset the signal-pulse counter |
| `0x010` | Read | 4 bytes | Signal-pulse counter |
| `0x100` | Write | 1 byte | Reset the dummy-pulse counter |
| `0x110` | Read | 4 bytes | Dummy-pulse counter |
| **Round-trip transmission latency** |  |  |  |
| `0x200` | Read | 2 bytes | Transmission latency |
| **MIKUMARI status monitoring** |  |  |  |
| `0x300` | Write | 1 byte | Reset the MIKUMARI status and phase-shift counters |
| `0x310` | Read | 4 bytes | Lane-up counter (`counter_cbt_lane_up`) |
| `0x320` | Read | 4 bytes | MIKUMARI link-up counter |
| `0x330` | Read | 4 bytes | Pattern-error counter |
| `0x340` | Read | 4 bytes | Checksum-error counter |
| `0x350` | Read | 4 bytes | Shut-off counter |
| **Phase monitoring** |  |  |  |
| `0x360` | Read | 4 bytes | Measured phase shift |
| `0x370` | Read | 4 bytes | Positive phase-shift counter |
| `0x380` | Read | 4 bytes | Negative phase-shift counter |

**Pulse counters.**
Each `HkttdDeliverLink` maintains separate counters for signal and dummy pulses.
Writing a counter-reset register transmits the value captured before the reset to the connected Sub module through the corresponding MMCI channel and then clears the counter.

**Round-trip transmission latency.**
The Main module periodically sends a timing pulse over the MIKUMARI link.
The returned pulse captures the timing-counter value at local address `0x200`.
This register is an unsigned 16-bit value with a unit of 8 ns.
Multiply the register value by 8 ns to obtain the round-trip transmission latency.

**MIKUMARI status monitoring.**
These registers count falling edges of the CBT lane-up and MIKUMARI link-up signals, and rising edges of the pattern-error, checksum-error, and shut-off-request signals.
Writing local address `0x300` clears these counters and the positive and negative phase-shift counters.

**Phase monitoring.**
The measured phase shift at local address `0x360` is a signed two's-complement 32-bit fixed-point value with 25 fractional bits and can be positive or negative.
Multiply the signed register value by `8 ns / 2^25` to obtain the measured phase shift.
The positive and negative phase-shift counters count phase-adjustment events in the corresponding direction.
Each event corresponds to a 78 ps adjustment step in the current Main firmware.

### Sub module

#### TCP FIFO configuration

The Sub firmware implements one SiTCP connection.
Its TCP transmit FIFO sends the measured latency difference when a dummy pulse is generated.
Each transmitted value is an unsigned 16-bit value with a unit of 8 ns.
Multiply the value by 8 ns to obtain the measured latency difference.
The value is transmitted in big-endian byte order, with the most-significant byte first.

#### HKTTD register map

The Sub `BusController` uses the following module ID assignments:
The assignments are defined in `HkttdSub/hdl/defBusAddressMap.vhd`.

| Module ID | Constant | Destination |
| --- | --- | --- |
| `0x0` | `kHPS` | `HkttdPulseScheduler` |

##### HkttdPulseScheduler registers

The `HkttdPulseScheduler` registers are assigned to Sub module ID `0x0`.
Their local addresses are defined in `hkttd-hdl/hkttdCore/defHkttdPulseScheduler.vhd`.

| Local address | Access | Size | Description |
| --- | --- | --- | --- |
| **Pulse-generation parameters** |  |  |  |
| `0x000` | Read/write | 2 bytes | Pulse-generation delay |
| `0x010` | Read/write | 2 bytes | Pulse width |
| **Pulse scalers** |  |  |  |
| `0x100` | Read | 8 bytes | Signal-pulse scaler |
| `0x110` | Read | 8 bytes | Dummy-pulse scaler |
| **MIKUMARI status monitoring** |  |  |  |
| `0x200` | Read | 4 bytes | Pattern-error counter |
| `0x210` | Read | 4 bytes | Checksum-error counter |
| `0x220` | Read | 4 bytes | Lane-up counter (`lane_up_bus`) |
| `0x230` | Read | 4 bytes | MIKUMARI link-up counter |
| **Debug control** |  |  |  |
| `0x300` | Write | 1 byte | Assert the development-only `flag_pc` strobe |

**Pulse-generation parameters.**
The pulse-generation delay and pulse width are unsigned 16-bit values with a unit of 1 ns.
Their default values are `0x1388` and `0x000A`, corresponding to 5 µs and 10 ns, respectively.
The same parameters are applied to both Sub-module pulse outputs.

**Pulse scalers.**
Each 64-bit scaler compares the pulse counts recorded by the Main and Sub modules.
Bits 63–32 contain the counter value received from the Main module through MMCI, while bits 31–0 contain the corresponding counter value captured by the Sub module.
The Sub-module counter is cleared when the Main-module counter value is received.
These values can be used to evaluate the pulse-transmission efficiency by comparing the Sub-module count with the Main-module count.
When the Main-module count is nonzero, calculate the efficiency as the Sub-module count divided by the Main-module count.

**MIKUMARI status monitoring.**
These registers contain the MIKUMARI monitoring counters captured when the dummy-pulse counter value is received through MMCI.
The pattern-error and checksum-error counters count rising edges, while the lane-up and link-up counters count falling edges.
The internal counters are cleared after their values are captured.

**Debug control.**
Writing local address `0x300` from the computer asserts the internal `flag_pc` strobe for one 8 ns system-clock cycle.
The strobe is intended for PC-controlled debugging, similar to a trigger signal.
It is currently not connected to other logic and is retained for development.
This register is expected to be removed in a future revision.

The addresses in these tables are base local addresses.
Multi-byte values are accessed one byte at a time, beginning with the least-significant byte at the base address.

## MMCI

MMCI multiplexes multiple data channels over each MIKUMARI link between the Main and Sub modules.
Each channel is identified by an 8-bit address and has an assigned priority and data width.

The current channel assignments are defined in `hkttd-hdl/hkttdCore/defMmciAddress.vhd`:

| Address | Direction | Data | Transmission condition | Priority |
| --- | --- | --- | --- | --- |
| `0x00` | Main → Sub | Phase-compensated pulse timing | When a pulse is sampled and phase compensation is ready | 0 |
| `0x01` | Main → Sub | Uncompensated pulse timing | When a pulse is sampled | 0 |
| `0x02` | Main → Sub | Signal-pulse counter | When local address `0x000` of the corresponding `HkttdDeliverLink` module (`0x1`–`0x8`) in the Main firmware is written | 1 |
| `0x03` | Main → Sub | Dummy-pulse counter | When local address `0x100` of the corresponding `HkttdDeliverLink` module (`0x1`–`0x8`) in the Main firmware is written | 1 |

The counter channels transmit the counter value captured before the corresponding counter is cleared.
The selected `HkttdDeliverLink` module in the Main firmware determines which Sub module receives the counter value.
The phase-compensated timing data is used for `NIM_OUT 1`, while the uncompensated timing data is used for `NIM_OUT 2`.

The MMCI implementation supports channels in both directions.
All HKTTD channels currently defined in `defMmciAddress.vhd` transmit data from the Main module to the Sub module.

# Software

Two programs are provided for HKTTD measurements: `efficiency_measurement` measures dummy-pulse transmission efficiency between the Main and Sub modules, and `avg_phase_measurement` records phase measurements from the Main module.
They also serve as examples for learning SiTCP communication: `efficiency_measurement` demonstrates slow control through RBCP register access, while `avg_phase_measurement` demonstrates TCP FIFO data transfer.

The software currently targets Linux.
`efficiency_measurement` depends on HulCore, provided by hul-common-lib, for RBCP communication.
Both programs rely on POSIX networking interfaces, either through HulCore or directly, and require adaptation for native Windows support.

## Register map

The destination IP address identifies the Main or Sub board, while the RBCP address identifies the register to access on that board.
RBCP addresses are defined as named constants in `include/RegisterMap.hh`.
See [Communication](../communication/communication.md) for the corresponding Main and Sub register maps.

The 32-bit RBCP address places the Module ID in bits 31–28 and the local address in bits 27–16.
In `include/RegisterMap.hh`, each register address is defined as `(Module ID << 28) | (Local address << 16)`, with the lower 16 bits set to zero.
For example, `HDL1::kAddrDummyCounter` is `(0x1 << 28) | (0x110 << 16) = 0x11100000`.

The table below lists the main registers used for efficiency measurement, rather than the complete contents of `include/RegisterMap.hh`.
The first three columns describe the firmware address assignments; the last two show the corresponding register names and RBCP addresses used by the software.

| Main/Sub module | Module ID | Local address | Register name | RBCP address |
| --- | --- | --- | --- | --- |
| Main (`HkttdPulseEventPublisher`) | `0x0` | `0x000` | `HPP::kAddrSwitch` | `0x00000000` |
| Main (`HkttdPulseEventPublisher`) | `0x0` | `0x100` | `HPP::kAddrDummyPeriod` | `0x01000000` |
| Main (`HkttdDeliverLink`, link 0) | `0x1` | `0x110` | `HDL1::kAddrDummyCounter` | `0x11100000` |
| Main (`HkttdDeliverLink`, link 0) | `0x1` | `0x100` | `HDL1::kAddrDummyCounterRst` | `0x11000000` |
| Sub (`HkttdPulseScheduler`) | `0x0` | `0x110` | `HPS::kAddrScalerDummy` | `0x01100000` |

## `efficiency_measurement`

`efficiency_measurement` measures dummy-pulse transmission efficiency between the Main module and the Sub module connected to MIKUMARI link 0.
The software uses SiTCP slow control through RBCP to configure pulse generation and read the measurement registers defined in the [Register map](#register-map) section.

At the start of each measurement, the software disables external signal pulses and dummy pulses, resets the counters, and sets the dummy-pulse period (`HPP::kAddrDummyPeriod`) to 2001 ns.
Pulse generation is controlled by `HPP::kAddrSwitch`: bit 0 enables external signal pulses, and bit 1 enables dummy pulses.
The software writes `0x00` to disable both pulse types during initialization, then writes `0x02` to enable dummy pulses while keeping external signal pulses disabled.
It stops generation by writing `0x00` after the Main counter (`HDL1::kAddrDummyCounter`) exceeds 100,000.
The software waits for the Main counter to stabilize before writing its reset register (`HDL1::kAddrDummyCounterRst`).

Writing to `HDL1::kAddrDummyCounterRst` causes the Main `HkttdDeliverLink` to send its counter value to the Sub through MMCI before clearing the counter.
On receiving this value, the Sub `HkttdPulseScheduler` captures both counts in the 64-bit scaler register `HPS::kAddrScalerDummy`: bits 63–32 contain the Main transmitted count, and bits 31–0 contain the Sub received count.
The Sub then clears its internal dummy-pulse counter for the next measurement.
The Sub also captures the MIKUMARI pattern-error, checksum-error, lane-down, and link-down counts and clears their internal counters.

The software reads the scaler through RBCP, extracts the two 32-bit counts, and calculates the transmission efficiency as the Sub received count divided by the Main transmitted count.
If the transmitted count is zero, the current implementation uses 1 as the denominator.
The software repeats the measurement and saves the pulse period, counts, monitoring results, and efficiency to a CSV file.

## `avg_phase_measurement`

`avg_phase_measurement` records the accumulated round-trip phase shift between the Main module and the Sub module connected to MIKUMARI link 0.
The Main firmware averages the detected phase and accumulates its changes.
When the accumulated round-trip phase shift changes and the output FIFO has space, the firmware queues the value for transmission through SiTCP TCP.

The program decodes each measurement as a signed 32-bit little-endian value and divides it by 2^22 to obtain the round-trip phase shift in nanoseconds.
The software saves both the raw hexadecimal value and the converted round-trip phase shift to a CSV file.

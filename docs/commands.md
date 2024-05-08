## Common Amateur Radio Interface (CARI)

### Introduction
CARI protocol is supposed to mimic professionally used *Common Public Radio Interface* (CPRI)
and offer a part of its functionality.
The protocol allows for basic data, control and supervision plane access, letting the user directly control
radio devices over a given medium. [ZeroMQ](https://zeromq.org/) is used for all data transfers.<br>

**Note:** Multi-oscillator (n>2) and MIMO device support is still work-in-progress.

### Basic topology
The interface assumes there are at least two devices: master and controlled device (slave), connected over some physical medium.
At the minimum, there are 4 paths for the signal flow:
- baseband uplink
- baseband downlink
- control
- supervision

Baseband uplink and downlink connections can only be used when required. Baseband streams are managed by ZMQ PUB-SUB pairs.

Control plane is used for setting radio equipment's parameters, such as oscillators' frequencies and RF signals' power levels.
This path uses a ZMQ REP-REQ pair.<br>
Supervision plane gives access to basic telemetry data: chassis temperature, reflected RF power, etc. The slave device
offers telemetry data over its ZMQ PUB instance.<br>
For multi-oscillator devices, there can be more than one pair of baseband streams.

**Note:** only two devices can exchange CARI command-reply pairs at the same time. In other words - only
one slave device can be addressed within a single transaction. To set a desired parameter over multiple devices,
it is required to issue several commands.

In the example below, the Baseband Unit (BBU) acts as a master over the Remote Radio Unit (RRU).

![CARI example](../gfx/CARI.png)

### Command/reply format
A *command* is a sequence of bytes sent by the master device, executing a particular action
at the controlled device.<br>
A *reply* is a similar sequence sent back by the controlled device as an acknowledgement.
Replies may contain return values for error signalling.<br>
Since there is no device addressing, the physical layer has to provide it.
Oscillators within a single device can be accessed independently.

| Command ID | Byte count | Params         |
|------------|------------|----------------|
| 1 byte     | 2 bytes    | 0..65532 bytes |

Table 1 - Command/reply format

The reply for a command has to use the same value in its ID field.
Byte count includes **all** bytes in the sequence.

### Command list
Note : the table below is still under development and therefore is subject to change.

| Command | Byte count | Action                                | Parameters                   | Return value               | Total length |
|---------|------------|---------------------------------------|------------------------------|----------------------------|--------------|
| 0x00    | 3          | Ping/pong                             | -                            | 32-bit value (error flags) | 7            |
| 0x01    | 11         | Set RX frequency                      | 64-bit frequency in Hz       | 0/1                        | 4            |
| 0x02    | 11         | Set TX frequency                      | 64-bit frequency in Hz       | 0/1                        | 4            |
| 0x03    | 4          | Set TX power                          | 8-bit value, unsigned[1]     | 0/1                        | 4            |
| 0x04    | -          | Reserved                              | -                            | -                          | -            |
| 0x05    | 7          | Set RX frequency correction           | float[2]                     | 0/1                        | 4            |
| 0x06    | 7          | Set TX frequency correction           | float[2]                     | 0/1                        | 4            |
| 0x07    | 4          | Set Automatic Frequency Control (AFC) | 0/1                          | 0/1                        | 4            |
| 0x08    | -          | Reserved                              | -                            | -                          | -            |
| 0x09    | 4          | Reception stop/start                  | 0/1                          | 0 at RX end only           | 4            |
| 0x0A    | variable   | ZMQ-SUB connect to publisher          | BBU address as string[3]     | 0/1                        | 4            |
| 0x0B    | variable   | Stream chunk                          | array of bytes, floats, etc. | 0/1                        | 4            |
| ...     | ...        | ...                                   | ...                          | ...                        | ...          |
| 0x80    | 3          | Get IDENT string                      | -                            | IDENT string               | variable     |
| 0x81    | 3          | Get device's capabilities             | -                            | 8-bit value                | 4            |
| 0x82    | 3          | Get RX frequency                      | -                            | 64-bit frequency in Hz     | 11           |
| 0x83    | 3          | Get TX frequency                      | -                            | 64-bit frequency in Hz     | 11           |
| 0x84    | 3          | Get TX power                          | -                            | 8-bit value, unsigned[1]   | 4            |
| 0x85    | 3          | Get RX frequency correction           | -                            | float[2]                   | 7            |
| 0x86    | 3          | Get TX frequency correction           | -                            | float[2]                   | 7            |
| ...     | ...        | ...                                   | ...                          | ...                        | ...          |

All values are little-endian. Return value of 0 means success, any other value is an error code.
Parameter of 0 disables the function, 1 enables it.

[1] 0.25dBm steps, starting at 0.0dBm: P[dBm]=value*0.25<br>
[2] The unit for this setting is *ppm*<br>
[3] The string does not have to be null-terminated<br>

### Device's capabilities flags

| Flag       | Meaning                                     |
|------------|---------------------------------------------|
| 0x01       | Amplitude modulation (incl. CW)             |
| 0x02       | Frequency modulation (incl. M17 and AFSK)   |
| 0x04       | Single sideband (incl. FreeDV)              |
| 0x08       | Phase shift keying (incl. pi/4-DQPSK)       |
| 0x10       | Raw I/Q                                     |
| 0x20..0x40 | Reserved                                    |
| 0x80       | Simplex=0/duplex=1                          |

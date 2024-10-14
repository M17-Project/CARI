## Common Amateur Radio Interface (CARI)

### Protocol revision
The protocol described in this document is **CARI 1.1**.

### Introduction
CARI protocol is supposed to mimic professionally used *Common Public Radio Interface* (CPRI)
and offer a part of its functionality.
The protocol allows for basic data, control and supervision plane access, letting the user directly control
radio devices over a given medium. [ZeroMQ](https://zeromq.org/) is used for all data transfers.<br>

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
Transmitters and receivers within a single device are called *subdevices*.
Subdevices can be accessed independently and are addressed using a single byte.
There can be a maximum of 64 subdevices.

| Command ID | Byte count | Params           |
|------------|------------|------------------|
| 1 byte     | 2 bytes    | 0 .. 65532 bytes |

| Command ID | Byte count | Address* | Params           |
|------------|------------|----------|------------------|
| 1 byte     | 2 bytes    | 1 byte   | 0 .. 65531 bytes |

**Table 1** - Command/reply format

\*Address of either a register or subdevice.

The reply for a command uses the same value in its ID field.
Byte count includes **all** bytes in the sequence.

### Command list
| Command | Byte count | Action                                | Address |Parameters                      | Return value               | Total length |
|---------|------------|---------------------------------------|---------|--------------------------------|----------------------------|--------------|
| 0x00    | 3          | Ping/pong                             | -       | -                              | 32-bit value (error flags) | 7            |
| 0x01    | 5          | Set register value                    | req'd   | 8-bit value                    | -                          | -            |
| 0x02    | 12         | Set subdevice's frequency             | req'd   | 64-bit frequency in Hz         | 0/1                        | 4            |
| 0x03    | 5          | Set subdevice's power                 | req'd   | 8-bit value, unsigned[1]       | 0/1                        | 4            |
| 0x04    | 8          | Set subdevice's frequency correction  | req'd   | float[2]                       | 0/1                        | 4            |
| 0x05    | 5          | Reception stop/start                  | req'd   | 0/1                            | 0 at RX end only           | 4            |
| 0x06    | variable   | ZMQ-SUB connect to publisher          | -       | BBU address as string[3]       | 0/1                        | 4            |
| 0x07    | variable   | Stream chunk                          | -       | array of bytes, floats, etc.   | 0/1                        | 4            |

**Table 2** - *WRITE* command list

| Command | Byte count | Action                                | Address | Return value               | Total length |
|---------|------------|---------------------------------------|---------|----------------------------|--------------|
| 0x80    | 3          | Get *IDENT* string                    | -       | *IDENT* string             | variable     |
| 0x81    | 4          | Get register value                    | req'd   | 8-bit value                | 5            |
| 0x82    | 4          | Get frequency                         | req'd   | 64-bit frequency in Hz     | 12           |
| 0x83    | 4          | Get power                             | req'd   | 8-bit value, unsigned[1]   | 5            |
| 0x84    | 4          | Get frequency correction              | req'd   | float[2]                   | 12           |

**Table 3** - *READ* command list

All values are little-endian. Return value of 0 means success, any other value is an error code.
Parameter of 0 disables the function, 1 enables it.

[1] 0.25dBm steps, starting at 0.0dBm: P[dBm]=value*0.25<br>
[2] The unit for this setting is *ppm*<br>
[3] The string does not have to be null-terminated<br>

### Error flags
| Bit      | Meaning                                        |
|----------|------------------------------------------------|
| 0 (LSB)  | PLL lock error                                 |
| 1        | Subdevice communication error                  |
| 2        | Device overheat                                |
| 3        | Frequency reference unlocked                   |
| 4 .. 30  | Reserved                                       |
| 31 (MSB) | Reserved                                       |

### Config/info register access
Some registers are write-only. Subdevices' capabilities are located at addresses `0x80 + address`
and `0x80 + address + 1`.

| Address         | Description                           | Access |
|-----------------|---------------------------------------|--------|
| 0x00            | CARI version support                  | R      |
| 0x01            | Number of subdevices                  | R      |
| 0x02 .. 0x7F    | User-defined                          | R/W    |
| 0x80            | Subdevice's capabilities, byte 0      | R      |
| 0x81            | Subdevice's capabilities, byte 1      | R      |
| 0x82 .. 0xFF    | Remaining subdevice's capabilities    | R      |

**Table 4** - Register address map

### CARI version support
This field holds supported CARI protocol version as `(major<<4)|minor`.

### Subdevice's capabilities
| Bit        | Meaning                                     |
|------------|---------------------------------------------|
| 0 (LSB)    | Amplitude modulation (incl. CW)             |
| 1          | Frequency modulation (incl. M17 and AFSK)   |
| 2          | Single sideband (incl. FreeDV)              |
| 3          | Phase shift keying (incl. pi/4-DQPSK)       |
| 4          | I/Q modulation available                    |
| 5          | Baseband stream compression available       |
| 6          | Supervision channel available               |
| 7 (MSB)    | Full duplex available                       |

**Table 5** - Capabilities - byte 0 (low)

| Bit        | Meaning                                     |
|------------|---------------------------------------------|
| 0 (LSB)    | Automatic Frequency Control available       |
| 1          | RX                                          |
| 2          | TX                                          |
| 3          | Reserved                                    |
| 4          | Reserved                                    |
| 5          | Reserved                                    |
| 6          | Reserved                                    |
| 7 (MSB)    | Reserved                                    |

**Table 6** - Capabilities - byte 1 (high)

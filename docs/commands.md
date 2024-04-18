## Common Amateur Radio Interface (CARI) Command Set

### Caveat
CARI protocol along with the table below is still under development and therefore is subject to change.

### Command format

| Command ID | Byte count | Params         |
|------------|------------|----------------|
| 1 byte     | 2 bytes    | 0..65532 bytes |

### Command list

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

### Device's capabilities

| Flag       | Meaning                                     |
|------------|---------------------------------------------|
| 0x01       | Amplitude modulation (incl. CW)             |
| 0x02       | Frequency modulation (incl. M17 and AFSK)   |
| 0x04       | Single sideband (incl. FreeDV)              |
| 0x08       | Phase shift keying (incl. pi/4-DQPSK)       |
| 0x10       | Raw I/Q                                     |
| 0x20..0x40 | Reserved                                    |
| 0x80       | Simplex=0/duplex=1                          |

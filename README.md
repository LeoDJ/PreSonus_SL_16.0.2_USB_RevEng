# PreSonus StudioLive 16.0.2 USB Reverse Engineering

## USB Protocol
Through looking at the USB traffic captured by Wireshark, while fiddling with the mixer controls / Universal Control PC software, I made the following conclusions:

### Request
The Universal Control application sends commands to the mixer via vendor defined USB control requests on endpoint 3.  
They follow the pattern `F0 <command> F7`.  

### Response  
- Starts with `04` or `07` depending on the length of the following payload (see "chunking" below)
- Always ends in `F7` (sometimes there are a few null bytes after the last byte though).  
- Appears to have some kind of chunking.  
  - The chunks are 4 bytes in length.
  - The first byte is the chunk flags (?) followed by 3 data bytes.
  - Chunk flags (bitfield): `04`: always set, `02`: last chunk, `01`: single chunk?
- (It seems to repeat the same message without chunking on endpoint 4?)

### Commands

#### `38 03` General Status Request?
![](doc/wireshark_3803.png)  
This request gets sent periodically roughly every 40ms.

Response:
 | Offset | Example       | Description                                                                                                                                                                        |
 | ------ | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
 | 00     | `39 03`       | Command (but 0x39)                                                                                                                                                                 |
 | ...    |               |                                                                                                                                                                                    |
 | 10     | `00 08 01 00` | [Channel](#channels) State Changed Notification <br>Bitfield:<br>`00 00 00 01`: Ch1<br>`00 00 00 02`: Ch2<br>`00 00 00 04`: Ch3<br>`00 00 00 08`: Ch4<br>`00 00 01 00`: Ch5<br>... |
 | ...    |               |                                                                                                                                                                                    |
 | 16     | `00`          | `01`: GEQ State Changed                                                                                                                                                            |
 | 17     | `00`          | `01`: Fader Positions Changed                                                                                                                                                      |
 | ...    |               |                                                                                                                                                                                    |
 | 21-33  | `01`          | **Channel Level Meters**<br>Min: `01`, Max: `15`(?)                                                                                                                                |
 | 34-41  | `01`          | More Level Meters?                                                                                                                                                                 |
 | 42     | `01 01`       | Main Level Meter (L/R)                                                                                                                                                             |
 | 44     | `1E`          | ?                                                                                                                                                                                  |

#### `6B xx` Channel State Read
![](doc/wireshark_6B00.png)
Reads the state of channel xx (see [Channels](#channels) below for the mapping).  
The software only requests this after detecting a "Channel State Changed" in the "General Status Request".  
It also follows the "chunking" mentioned above.  
All two-byte values are actually just one byte (`00`-`FF`), with the nibbles spread across the two bytes, big-endian.  

Response: 
 | Offset | Example | Description                                                                                            |
 | ------ | ------- | ------------------------------------------------------------------------------------------------------ |
 | 00     | `6B`    | Command                                                                                                |
 | 01     | `00`    | [Channel](#channels)                                                                                   |
 | 02     | `03 08` | Main Level (fader)                                                                                     |
 | 04     | `07 02` | Pan                                                                                                    |
 | 06     | `0F 0F` | ?                                                                                                      |
 | 08     | `00 00` | Aux1 Level                                                                                             |
 | 10     | `00 00` | Aux2 Level                                                                                             |
 | 12     | `00 00` | Aux3 Level                                                                                             |
 | 14     | `00 00` | Aux4 Level                                                                                             |
 | ...    |         |                                                                                                        |
 | 28     | `00 00` | FXA Level                                                                                              |
 | 30     | `00 00` | FXB Level                                                                                              |
 | ...    |         |                                                                                                        |
 | 44     | `09 0C` | HPF Frequency                                                                                          |
 | 46     | `08 03` | EQ Low Frequency                                                                                       |
 | 48     | `00 00` | ?                                                                                                      |
 | 50     | `09 04` | EQ Mid Frequency                                                                                       |
 | ...    |         |                                                                                                        |
 | 62     | `04 06` | EQ Low Gain                                                                                            |
 | 64     | `00 00` | ?                                                                                                      |
 | 66     | `00 00` | EQ Mid Gain                                                                                            |
 | 68     | `08 00` | ?                                                                                                      |
 | 70     | `0F 0F` | Compressor                                                                                             |
 | 72     | `08 00` | Compressor Ratio                                                                                       |
 | 74     | `08 00` | Compressor Response                                                                                    |
 | 76     | `00 00` | ?                                                                                                      |
 | 78     | `00 00` | Compressor Gain                                                                                        |
 | ...    |         |                                                                                                        |
 | 84     | `00 00` | Gate                                                                                                   |
 | ...    |         |                                                                                                        |
 | 106    | `00`    | Bitfield:<br>`01`: HPF On                                                                              |
 | 107    | `00`    | Bitfield:<br>`08`: Gate On                                                                             |
 | 108    | `0A`    | Bitfield:<br>`02`: Compressor On<br>`04`: Compressor Auto<br>`08`: EQ High On                          |
 | 109    | `00`    | Bitfield:<br>`02`: EQ Low On<br>`08`: EQ High On                                                       |
 | 110    | `00`    | Bitfield:<br>`01`: EQ Low Shelf On<br>`08:` EQ High Shelf On                                           |
 | 111    | `05`    | Bitfield:<br>`01`: Phantom Power On<br>`02`: USB Audio Input On<br>`04`: Polarity<br>`08`: Digital Out |
 | 112    | `06`    | Bitfield:<br>`01`: Mute<br>`02`: Solo<br>`04`: Link                                                    |
 | ...    |         |                                                                                                        |
 | 121    | `00`    | ?                                                                                                      |

#### `6A xx` Channel State Write
![](doc/wireshark_6A00.png)  
On writes, no "chunking" is applied, it's simply the same data as in the Channel State Read command.

#### `6E` Fader Position Read
![](doc/wireshark_6E.png)  
Reads all fader positions.  
The software only requests this after detecting a "Fader Positions Changed" in the "General Status Request".  
Fader position is one byte `04`-`FF`, but the nibbles spread across two bytes, big endian.

Response:
 | Offset | Example | Description                                          |
 | ------ | ------- | ---------------------------------------------------- |
 | 00     | `6E`    | Command                                              |
 | 01-31  | `00 04` | Fader Position Ch1-Aux4 (see [channels](#channels)]) |
 | 33     | `0F 0F` | Fader Position Main                                  |
 | 35     | `00 05` | (FXA??)                                              |
 | 37     | `00 05` | (FXB??)                                              |
 | 39     | `00 00` | ?                                                    |
 | 41     | `06 01` | ?                                                    |

### Channels
- `00-07`: Channels 1 - 8
- `08-0B`: Channels 9/10 - 15/16 (stereo channels)
- `0C-0F`: Aux 1-4
- `10`: Main
- `11-12`: FXA/FXB
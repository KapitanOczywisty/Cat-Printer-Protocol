
# Packet format

<code>0x51 0x78 COMMAND DIRECTION LENGTH<sup>lo</sup> LENGTH<sup>hi</sup> [PAYLOAD<sub>...</sub>] CRC 0xFF</code>

> Protocol uses Little Endian byte order:
> - 16-bit values are written as `value & 0xFF, value >> 8`
> - 1-bit pixels are stored in reverse order (less significant bit first)

- `0x51 0x78` - packet start[^packet-start]
- `COMMAND` - Command byte, see [Commands](#commands) for details
- `DIRECTION` - Direction byte: `0x00` command, `0x01` response from printer
- `LENGTH` - Payload length, as 16-bit value: <code>LENGTH<sup>lo</sup> = LENGTH & 0xFF ; LENGTH<sup>hi</sup> = LENGTH >> 8</code>
- `PAYLOAD` - Every commands have at least one byte of payload
- `CRC` - 8-bit CRC checksum of the payload
- `0xFF` - packet end

Long packets can be sent in parts, though whole packet cannot be longer than 4104 bytes[^length]

[^packet-start]: On the new devices, packed supposed to start with `0x18 0x51 0x78`, though I couldn't see any difference in the packet handling. It's advised to use the old packet start.
[^length]: Packet length was tested on P7H, it accepted packets up to 4104 bytes, split into chunks of up to 509 bytes (512 MTU).

# Commands

| | 0xA0 | 0xB0 | 0xC0 |
| :--- | --- | --- | --- |
0x0 | [Retract](#0xa0-160--96-retract)
0x1 | [Feed](#0xa1-161--95-feed)
0x2 | [Bitmap](#0xa2-162--94-bitmap)
0x3 | [GetDeviceState](#0xa3-163--93-request-device-state)
0x4 | [SetDpi](#0xa4-164--92-setdpi)
0x5
0x6 | [Lattice](#0xa6-166--90-lattice)
0x7
0x8 | [GetDeviceInfo](#0xa8-168--88-request-device-info)
0x9 | [UpdateDevice](#0xa9-169--87-updatedevice)

## Information commands

These commands will trigger notification with the same command byte.

### 0xA3 *<sup>(163 / -93)</sup>* Request device state

`PAYLOAD = 0x00`

> iprint name: GetDeviceState

### 0xA8 *<sup>(168 / -88)</sup>* Request device info

`PAYLOAD = 0x00`

> iprint name: GetDeviceInfo

### 0xBB *<sup>(187 / -69)</sup>* Request device ID

`PAYLOAD = 0x01`

> iprint name: GetDevId

## Control commands

### 0xBE *<sup>(190 / -66)</sup>* Set drawing mode

`PAYLOAD = MODE [GRAYSCALE]`

Doesn't seem to have any noticable effect on the printer, even though there are some values defined in the iprint. Could be dependent on the printer model and version. It's advised to send `0x00` payload.

- `MODE` - Drawing mode: `0x00` - image, `0x01` - text
- `GRAYSCALE` - Optional grayscale mode: `0x00` - 8, `0x01` - 16

> iprint naming: ApplyEnergy

### 0xA9 *<sup>(169 / -87)</sup>* UpdateDevice

`PAYLOAD = 0x00`

TODO: Some settings are applied here? Not sure what exactly is being updated.

### 0xA4 *<sup>(164 / -92)</sup>* SetDpi

It's set to 50 in most cases.

TODO: research

> iprint naming: quality

### 0xA6 *<sup>(166 / -90)</sup>* Lattice

`START_PAYLOAD = 0xAA 0x55 0x17 0x38 0x44 0x5F 0x5F 0x5F 0x44 0x38 0x2C`<br/>
`END_PAYLOAD   = 0xAA 0x55 0x17 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x17`

Some control boundaries - unknown purpose.

### 0xBD *<sup>(189 / -67)</sup>* Set feeder speed

`PAYLOAD = DIVIDER`

$\texttt{Speed} \left[\frac{\texttt{lines}}{\texttt{s}}\right] = \frac{2304}{\texttt{DIVIDER}}$

Argument is a divider for the feeder speed, it affects both feeding and printing. Waiting for printing data or too high energy setting can cause the feeder to stutter, and this is undesirable for good quality prints.

#### Example speeds

Divider | Speed $\left[\frac{\texttt{lines}}{\texttt{s}}\right]$ | Time $\left[\frac{\texttt{µs}}{\texttt{line}}\right]$
---: | ---: | ---:
192 | 12 l/s | 83 333 µs
128 | 18 l/s | 55 556 µs
64 | 36 l/s | 27 778 µs
32 | 72 l/s | 13 889 µs
16 | 144 l/s | 6 944 µs
8 | 288 l/s | 3 472 µs

Feeder is not able to handle values bellow 8 and above 192.

### 0xAF *<sup>(175 / -81)</sup>* Exposure

<code>PAYLOAD = EXP<sup>lo</sup> EXP<sup>hi</sup></code>

Sets how intense black dots should be, 0 is the lightest, 65535 is the darkest.

> It looks like it could be the time of exposure of each line in microseconds. - unconfirmed

iprint naming: `SetEnergy`

## Feeder commands

### 0xA0 *<sup>(160 / -96)</sup>* Retract

<code>Payload = LIN<sup>lo</sup> LIN<sup>hi</sup></code>

Number of lines to retract, 0 - 65535.

### 0xA1 *<sup>(161 / -95)</sup>* Feed

<code>Payload = LIN<sup>lo</sup> LIN<sup>hi</sup></code>

Number of lines to feed, 0 - 65535.

## Drawing commands

### 0xA2 *<sup>(162 / -94)</sup>* Bitmap

### 0xBF *<sup>(191 / -65)</sup>* Draw continuous bits

Format can compress lines with continous lines of the same color. Payload consists of 1-bit of color and 7-bits of length, which can be represented as `color_bit << 7 | count`, each byte can represent at most 127 single-color pixels. White line of 384 pixels would be represented by `0x7f 0x7f 0x7f 0x03` (3x127 + 3).

> [!IMPORTANT]
> Each 8 pixel group is printed in reverse order, just like in the bitmap command.

#### Example
```text
0000 1111 | 1110 0000 | 1111 1111 | ...
 <-----<  |  <-----<  |  <-----<  | ...

0x84 = 1 << 7 | 4		// sends 4 "1" pixels
0x09 = 0 << 7 | 9		// sends 9 "0" pixels
0x8b = 1 << 7 | 11		// sends 11 "1" pixels
...
```
> [!WARNING]
> Sum of bits have to be equal to line length, otherwise payload is ignored or treated as bitmap data.

### 0xCF *<sup>(162 / -94)</sup>* Draw grayscale

This mode allows drawing 16-color grayscale images. Each byte represents 2 pixels, which is then compressed using LZO1x algorithm. Also allows drawing multiple lines at once, this combined with the LZO compression, decreases the amount of data sent to the printer, though adds some processing overhead, especially on the printer side.

Color values: F - black, 0 - white, E through 1 - grayscale

Pixel data are packed line-by-line as follows: `pixel[1] << 4 | pixel[0], pixel[3] << 4 | pixel[2], ...`, then compressed using LZO1x_1 algorithm.

Payload: <code>DATA_LEN<sup>lo</sup> DATA_LEN<sup>hi</sup> LZO_LEN<sup>lo</sup> LZO_LEN<sup>hi</sup> [LZO_DATA<sub>...</sub>]</code>
- `DATA_LEN` - Data length  before compression
- `LZO_LEN` - LZO compressed data length
- `LZO_DATA` - Compressed data

Maximum of 4095 uncompressed bytes can be sent in a single packet, and since only complete lines are used, the effective limit is 21 lines[^lzo1x-limit]. LZO1x limit can be calculated as: $\frac{4095\left[\texttt{byte}\right] * 2\left[\frac{\texttt{pixel}}{\texttt{byte}}\right]}{384\left[\frac{\texttt{pixel}}{\texttt{line}}\right]} = 21.328125 \left[\texttt{line}\right]$

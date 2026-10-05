# Audio Stream Serial Protocol

Source: `Source/hardware/audio.c` (`audio_stream_*` functions).

## Link settings

| Item | Value |
|---|---|
| Port | UART4 (PC10 TX, PC11 RX) |
| Baud rate while streaming | 230400 |
| Format | 8 data bits, no parity, 1 stop bit |
| Baud rate after stream stops | 38400 (system default) |
| Direction | Radio to host only |
| Multi-byte values | Little-endian |

The radio switches to 230400 baud at the start of the stream (via `uart_set_baud()`) and returns to 38400 after the stop packet.

## Packet framing

Every packet starts with `0xAA` and ends with `0x55`. The second byte is the packet identifier.

```
0xAA | ID | ...payload... | 0x55
```

| ID | Packet |
|---|---|
| `0x60` | Audio data |
| `0x61` | Stream start (metadata) |
| `0x62` | Stream stop |

## Stream start packet (`0x61`)

Sent once, before any audio data.

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xAA` |
| 1 | 1 | `0x61` |
| 2 | 4 | Frequency, u32 (units as passed to `audio_stream_start()`) |
| 6 | 2 | Channel number, u16 |
| 8 | 32 | Description, ASCII |
| 40 | 1 | `0x55` |

Total length is 41 bytes.

Description rules:
- Shorter strings are padded with `0x00` after the terminator.
- Longer strings are truncated to 32 characters and are not null-terminated.

After this packet, the sample timer (shared with the APRS Bell 202 timer) is set to 9600 Hz and sampling begins.

## Audio data packet (`0x60`)

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xAA` |
| 1 | 1 | `0x60` |
| 2 | 2 | Length field, `0x00 0x08` |
| 4 | 2048 | 1024 samples, signed 16-bit, little-endian |
| 2052 | 1 | `0x55` |

Total length is 2053 bytes.

- The length field is sent as bytes `00 08`, which is `0x0800` = 2048 when read little-endian. It is the payload size in bytes, not the sample count.
- Samples are the ADC3 reading minus a DC bias. The bias is the average of 2048 ADC readings taken in `audio_stream_init()`.
- The sample rate is 9600 Hz. A 1024-sample packet covers about 106.7 ms of audio. Sending 2053 bytes at 230400 baud takes about 89 ms, so the link keeps up.
- The radio double-buffers: a 2048-sample buffer is split into two 1024-sample halves. The sampling ISR fills one half while the main loop sends the other, so sampling is never interrupted.
- When a half fills, the ISR sets `stream_index_trigger` to the start index of the full half (`0` or `1024`). `0xFFFF` means no packet is ready. `audio_stream_tick()` sends that half and then resets the trigger to `0xFFFF`.
- If the main loop is too slow to send a half before the other fills, data can be overwritten or a packet lost.

## Stream stop packet (`0x62`)

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xAA` |
| 1 | 1 | `0x62` |
| 2 | 1 | `0x55` |

The sample timer is disabled before this packet is sent. The baud rate returns to 38400 right after it.

## Host receive notes

1. Open the port at 230400 8N1 and wait for `0xAA`.
2. Read the ID byte and handle the packet by ID, using the sizes above. Do not scan for `0x55` to find the end, because audio data can contain that byte value.
3. Check that the last byte is `0x55`. If it is not, discard the packet and resync on the next `0xAA`.
4. On `0x62`, reopen or reconfigure the port at 38400 baud.

# Audio Stream Serial Protocol for nicFW950 (RT-950 Pro)


## Link settings

| Item | Value |
|---|---|
| Port | Serial Motorola Connection |
| Baud rate while streaming | 230400 |
| Format | 8 data bits, no parity, 1 stop bit |
| Direction | Radio to host only |
| Multi-byte values | Little-endian |

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
| 2 | 4 | Frequency, (32 bit little endian in 10Hz units) |
| 6 | 2 | Channel number, (16 bit little endian - Channel number is 0 for frequency mode VFO events |
| 8 | 32 | Description, ASCII (null termination only applied to strings less than 32 characters |
| 40 | 1 | `0x55` |

Total length is 41 bytes.

Description rules:
- Shorter strings are padded with `0x00` after the terminator.
- Longer strings are truncated to 32 characters and are not null-terminated.

## Audio data packet (`0x60`)

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xAA` |
| 1 | 1 | `0x60` |
| 2 | 2 | Length field, `0x00 0x08` (Always 2048) |
| 4 | 2048 | 1024 samples, signed 16-bit, little-endian |
| 2052 | 1 | `0x55` |

Total length is 2053 bytes.

- The length field is the payload size in bytes, not the sample count.
- The sample rate is 9600 Hz. A 1024-sample packet covers about 106.7 ms of audio. Sending 2053 bytes at 230400 baud takes about 89 ms, so the link keeps up.

## Stream stop packet (`0x62`)

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `0xAA` |
| 1 | 1 | `0x62` |
| 2 | 1 | `0x55` |

## Host receive notes

1. Open the port at 230400 8N1 and wait for `0xAA`.
2. Read the ID byte and handle the packet by ID, using the sizes above. Do not scan for `0x55` to find the end, because audio data can contain that byte value.
3. Check that the last byte is `0x55`. If it is not, discard the packet and resync on the next `0xAA`.
4. Audio is not continually sent. A squelch opening event will trigger the stream to begin, starting with the stream start packet and ending with the stream stop packet.

### Suggested Process Loop

- Scan for byte 0xAA
  * Once 0xAA is receieved switch on the next byte.
    - For a 0x61 (start stream) packet, extract and store the stream metadata in state variable and/or display them in the UI.
      * At this point initialize any live audio system or construct your audio file header, preferably including the meta data into the header.
    - For a 0x60 (audio data) packet, extract the 1024 16 bit samples.
      * Add/process the audio data into your audio file and/or route it to your live audio system for instant playback.
    - For a 0x62 (stop stream) packet. Finalize your audio file and save/close the file and/or shut down your live audio system/place it into an idle state

 ### Strategy for Live Playback

 As audio data packets are received, place the data blacks into a queue for processing.
 Do not begin playback until you have at least two blocks queued
 If when removing a block from the queue, the queue is empty, process the 1024 sample block and repeat the last sample making 1025 in total.
 If when removing a block from the queue, the queue contains two or more waiting blocks, process the 1024 sample block but do not send the last sample making 1023 in total
 (Note that you may introduce a more sophisticated zero crossing policy for the sample addition or removal rather than using the last sample)

 This keeps the audio synced and prevents latency drift. There will be small differences in the sample rates on both sides.

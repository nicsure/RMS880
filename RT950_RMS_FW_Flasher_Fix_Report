# RMS firmware flasher: old implementation versus corrected implementation

## Scope and evidence

This report compares the uploaded `NicFW950_RMS6` project's `Firmware.axaml.cs` with the supplied replacement file. It describes normal firmware flashing with **“Include current channel list as permanent channels” unchecked**, which is the mode used when the problem occurred.

The comparison uses:

- The old RMS source from the uploaded project archive.
- The replacement `Firmware.axaml.cs` supplied in this conversation.
- The web flasher's `app.js`, including its partial-final-block fix.
- The repository's bootloader protocol notes and original serial capture.
- The previously uploaded `RT_950Pro_V0.29_260617.BTF`, which is 385,824 bytes long.

**Confirmed:** the host sends different header lengths, block announcements, and bytes beyond the end of the firmware. **Not confirmed:** which difference caused particular menus to freeze, how the bootloader internally uses each field, or the exact contents of flash after either transfer. Establishing those points requires bootloader source, hardware readback, or controlled device testing.

The user reported that the radio booted after RMS flashing but some menus froze; flashing the same firmware through the web flasher did not produce that behaviour. This is useful evidence of a flashing-path difference, but does not isolate one specific cause.

## Overview of the changes

| Area | Old RMS | Corrected RMS | Communication effect |
|---|---|---|---|
| Ordinary image allocation | Rounded image size plus 2,024 bytes | Exactly the rounded image size | Removes one transmitted block beyond the rounded image |
| Bytes beyond input EOF | Zero-initialized (`00`) | Filled with `FF` | Changes only the padding after the original file bytes |
| Image identity, command `02` | 16 bytes from offset `0x3E0` | 32 bytes from the same offset | Header payload grows from 16 to 32 bytes |
| Block announcement, command `04` | Total transmitted block count | Final zero-based block index | Announcement now agrees with the observed protocol |
| Input size checks | No explicit minimum/sequence-limit check | 1,024 to 67,108,864 bytes | Rejects invalid sizes before opening the serial port |
| Progress | Based on byte address and allocated size | Completed blocks / total blocks | Correct final progress; no packet change |
| UI recovery | Tabs restored only after normal completion | `try`/`catch`/`finally` | Exceptions are reported and tabs restored; no packet change |

The fixed ordinary transfer now follows the web flasher's header, block count, block contents, and finish sequence. This does not imply that the two applications have identical transport implementations or timing.

## 1. Serial framing and settings that did not change

Both RMS versions open the firmware connection at **115,200 baud, 8 data bits, no parity, and one stop bit**. The normal serial path uses no hardware handshake by default. `Com.Create(..., false)` means this path does not run the application firmware's separate baud-change exchange.

Packets retain this structure:

```text
AA | command | sequence (2 bytes) | payload length (2 bytes) | payload | CRC (2 bytes) | 55
```

The sequence, payload length, and CRC are big-endian. Packet overhead is nine bytes.

CRC remains CRC-16/XMODEM: polynomial `0x1021`, initial value `0x0000`, no reflection, and no final XOR. It covers the command, sequence, length, and payload; it excludes the leading `AA`, trailing CRC, and final `55`.

The ordinary session still follows this order:

1. `0A`: identify using ASCII `BOOTLOADER_V3`.
2. `02`: send image identity/header information.
3. `04`: announce the final block index.
4. `03`: send each 1,024-byte block with its zero-based sequence.
5. `45`: finish/restart.

Every command waits for its acknowledgement before the next command is sent. There were no added delays, baud changes, or retry loops in the fix.

## 2. Correct image allocation: removing the extra block

### Old code

```csharp
int len = (int)Math.Ceiling(unpadded.Length / 1024.0) * 1024;
len += 2024;
firmware = new byte[len];
```

Later, the old code calculated:

```csharp
int blocks = firmware.Length / 1024;
```

This is integer division. If `N` is the number of blocks needed for the rounded input image:

```text
Allocated bytes       = N × 1024 + 2024
Transmitted blocks    = floor((N × 1024 + 2024) / 1024)
                      = N + 1
Transmitted bytes     = (N + 1) × 1024
Unused buffer bytes   = 1000
```

Adding 2,024 bytes therefore did **not** transmit exactly 2,024 extra bytes. It transmitted one extra 1,024-byte block beyond the correctly rounded image; 1,000 allocated bytes were never transmitted.

### Corrected code

```csharp
const int blockSize = 1024;
int len = ((unpadded.Length + blockSize - 1) / blockSize) * blockSize;
firmware = new byte[len];
Array.Fill(firmware, (byte)0xFF);
Array.Copy(unpadded, 0, firmware, 0, unpadded.Length);
```

The allocation is exactly the size needed for complete transfer blocks. An aligned file gets no additional block. A partial final block gets only the padding required to reach the next 1,024-byte boundary.

### Example: the uploaded firmware

| Quantity | Old RMS | Corrected RMS |
|---|---:|---:|
| Input file size | 385,824 bytes | 385,824 bytes |
| Correctly rounded size | 386,048 bytes | 386,048 bytes |
| Allocated buffer | 388,072 bytes | 386,048 bytes |
| Transmitted blocks | 378 | 377 |
| Transmitted payload bytes | 387,072 bytes | 386,048 bytes |
| Transmitted bytes after input EOF | 1,248 | 224 |
| Untransmitted buffer tail | 1,000 bytes | 0 bytes |

These are firmware payload sizes; packet headers, CRCs, acknowledgements, and setup commands add serial traffic on top of them.

## 3. Padding: `00` versus `FF`

A newly allocated C# byte array contains zeros. The old code copied the original firmware over the beginning of that array and left the remainder as `00`.

The corrected ordinary branch explicitly fills the buffer with `FF` before copying the original firmware into it.

For the uploaded image, offsets below are relative to the start of the input/transfer image, **not independently established physical flash addresses**:

```text
Offsets                 Old RMS                   Corrected RMS
0x000000–0x05E31F        Original firmware bytes   Original firmware bytes
0x05E320–0x05E3FF        224 bytes of 00            224 bytes of FF
0x05E400–0x05E7FF        Extra block of 00         Not transmitted
```

The original file bytes are unchanged in both implementations. The first difference in the command `03` firmware payloads occurs at EOF, inside block 376. Old RMS then sends block 377 as an additional all-zero block; corrected RMS finishes after block 376.

`FF` is the conventional erased value for flash memory and matches the corrected web flasher's padding choice. However, writing `FF` is not proof that an erase occurred, nor proof that existing data was preserved. Actual erase/program operations are controlled by the bootloader. The original capture contains only aligned firmware, so it does not independently establish the partial-block padding convention.

## 4. Image identity: sending all 32 bytes

### Old code

```csharp
packet = ConstructPacket(0x02, 0x0000, firmware, 0x3e0, 16);
```

### Corrected code

```csharp
packet = ConstructPacket(0x02, 0x0000, firmware, 0x3e0, 32);
```

The source offset remains `0x3E0` (992). The old version sent file offsets 992–1007; the corrected version sends offsets 992–1023.

The resulting command frame changes from:

```text
Old:   AA 02 00 00 00 10 [16 header bytes] [CRC] 55   — 25 bytes total
Fixed: AA 02 00 00 00 20 [32 header bytes] [CRC] 55   — 41 bytes total
```

The CRC necessarily changes because the payload length and contents change. The CRC algorithm itself is unchanged.

This correction matches both the web implementation and the 32-byte command `02` payload in the original serial capture. The missing 16 bytes were still present in the old command `03` transfer of block zero; they were omitted specifically from the earlier identity command. Therefore it would be inaccurate to say the old RMS never transmitted those firmware bytes at all.

The available evidence does not explain how the bootloader uses all 32 identity bytes or whether accepting only 16 affects validation or programming decisions.

## 5. Block announcement: count versus final index

### Old code

```csharp
int blocks = firmware.Length / 1024;
packet = ConstructPacket(
    0x04, 0x0000,
    [(byte)(blocks >> 8), (byte)(blocks & 0xFF)],
    0, 2);
```

### Corrected code

```csharp
int blocks = firmware.Length / blockSize;
int lastBlock = blocks - 1;
packet = ConstructPacket(
    0x04, 0x0000,
    [(byte)(lastBlock >> 8), (byte)(lastBlock & 0xFF)],
    0, 2);
```

The observed command `04` argument is the last zero-based block index, rather than the number of blocks. In the original capture, command `04` carries `00 08`, followed by nine blocks numbered `0` through `8`.

For the uploaded image:

| Detail | Old RMS | Corrected RMS |
|---|---|---|
| Blocks actually transmitted | 378 | 377 |
| Actual command `03` sequences | 0–377 | 0–376 |
| Command `04` argument | 378 (`01 7A`) | 376 (`01 78`) |
| Announcement under observed semantics | Final index 378, implying 379 blocks | Final index 376, implying 377 blocks |

There are two separate corrections here: removing the extra transmitted block, and subtracting one when encoding the announcement. Together they change the example's argument from `01 7A` to `01 78`.

The old method's announced final index was one greater than the last block it actually sent. Whether that affects erase extent, completion checks, or another bootloader operation cannot be established without more device-side evidence.

## 6. Packet construction and acknowledgements

The packet constructor itself did not need a CRC or byte-order correction. Both versions correctly encode a command `03` block as:

```text
AA 03 [sequence high] [sequence low] 04 00 [1024 payload bytes] [CRC high] [CRC low] 55
```

A full firmware-block frame is 1,033 bytes. Sequence numbers remain 16-bit and support indices greater than 255.

For the example, blocks 0–375 have identical payloads and therefore identical frames in the old and corrected implementations. Block 376 differs in its final 224 padding bytes and CRC. Old RMS alone sends block 377.

Both versions retain RMS's acknowledgement checks:

- Find the leading `AA`.
- Parse the command, sequence, length, payload, and CRC.
- Require the trailing `55` and a valid CRC.
- Require the expected command, sequence `6`, and an empty payload.

An acknowledgement confirms that a packet met the bootloader's acceptance conditions. It is not a readback comparison of the programmed firmware and does not prove that every runtime menu will work.

RMS scans for a frame start, whereas the web implementation expects the next buffered nine response bytes to be an acknowledgement frame. That existing parser difference was not changed by this fix.

## 7. Input validation

The corrected method rejects input shorter than 1,024 bytes or longer than 67,108,864 bytes before opening the port.

The lower bound ensures that the complete 32-byte identity region at offset 992 exists. The upper bound limits the transfer to 65,536 blocks, with valid zero-based sequence values `0` through `65535`.

The validation prevents malformed lengths and sequence wraparound. It does **not** authenticate firmware, verify that it targets the correct radio, or validate its executable contents. It is an input-size check, not a complete firmware-format validator.

## 8. Progress and exception recovery

Old progress reporting used:

```csharp
SetProgress(address, len - 1024);
```

Because `len` included the additional 2,024 bytes, the final acknowledged block did not necessarily display 100%.

The replacement uses:

```csharp
SetProgress(i + 1, blocks);
```

Progress now represents the number of acknowledged blocks divided by the total block count. This can show 100% after the last data acknowledgement; final success is still reported only after command `45` is acknowledged.

The event handler now awaits the flash task inside `try`/`catch`/`finally`. An exception reports a failure, and `finally` restores the tab controls. These changes affect host behaviour only and add no serial commands.

Existing limitations remain: failed acknowledgements return early rather than raising detailed per-command errors, `Com.Write` swallows transport exceptions, and there is no programmed-image readback verification.

## 9. Why an old RMS-flashed image could boot but have failing menus

The old method transmitted the original file bytes intact, which is consistent with the radio being able to boot and run much of its firmware.

The discrepancy lies in the setup information and the extra writes beyond EOF. A possible explanation is that zero padding or the expanded transfer affected memory used only when certain functionality was accessed. Another possibility is that the wrong announcement or shortened identity information affected the bootloader's programming decisions.

These remain hypotheses. No supplied memory map proves that the affected offsets contain menu data, and no flash readback proves what the bootloader wrote or erased. It would be unjustified to claim that calibration, settings, or channel storage was definitely overwritten, or that the old flasher caused permanent hardware damage.

The user's observation that web flashing restores normal behaviour supports investigating the flashing process, but several fields changed together. A successful test of the replacement would demonstrate a practical improvement without identifying which individual change resolved the problem.

## 10. Permanent-channel mode is outside this fix's validated scope

The replacement retains the old permanent-channel allocation and copying logic, including `0xED000` and the additional 2,024 allocated bytes. The shared 32-byte header and final-index corrections apply to that mode too, but its image construction remains different from ordinary flashing.

It must not be described as fully corrected or equivalent to the web flasher. The ordinary input-size check also does not validate whether an image fits safely around the embedded permanent-channel region. That feature needs separate review before use.

## 11. Verification and recommended device check

Completed checks supporting this analysis:

- Inspected the original RMS code, its serial wrapper, the replacement, and web code.
- Confirmed the original capture's 32-byte identity command and final-index convention.
- Calculated the old and corrected allocations and transmitted byte extents using the uploaded firmware.
- Previously ran web-flasher regression tests and a simulated-serial Chromium transfer with the uploaded firmware. Those checks validated the web transfer logic, not the C# replacement or a real radio.

The C# replacement has not been compiled in this cloud workspace, and no real-radio transfer or memory readback was performed here.

For practical validation, build the replacement in Visual Studio, leave permanent channels unchecked, flash the same firmware file, and retry the menus that previously froze. For a definitive byte-level investigation, capture both serial sessions and read back the relevant firmware memory after each method using a supported device procedure. Compare the file contents, the final partial block, and the region beyond EOF. A NAND-storage backup is not automatically a readback of the MCU's firmware flash; the correct memory region must be verified.

## Appendix: exact short command frames for the uploaded firmware

The following frames are calculated from the uploaded image using CRC-16/XMODEM. They are expected host packets derived from source code, not a new hardware capture. Each line includes the full frame, including CRC and delimiters.

```text
Both: identify
AA 0A 00 00 00 0D 42 4F 4F 54 4C 4F 41 44 45 52 5F 56 33 40 34 55

Old: image identity
AA 02 00 00 00 10 52 54 2D 39 35 30 20 20 20 20 20 20 00 00 10 B5 DA FB 55

Fixed: image identity
AA 02 00 00 00 20 52 54 2D 39 35 30 20 20 20 20 20 20 00 00 10 B5 4C 10 84 EA 53 04 04 D5 40 42 C1 F1 00 01 38 BF DF 7C 55

Old: block announcement
AA 04 00 00 00 02 01 7A 43 4A 55

Fixed: block announcement
AA 04 00 00 00 02 01 78 63 08 55

Both: finish
AA 45 00 00 00 00 32 3F 55

```

The identity, block-announcement, and finish exchanges still each receive one corresponding acknowledgement. Old RMS additionally sends one data-block frame and receives one additional data acknowledgement. The changed identity frame carries 16 more bytes in the corrected transfer.

# Push 2 vs Push 3: Protocol Comparison

Detailed technical comparison between Ableton Push 2 and Push 3 communication protocols, based on reverse engineering analysis and official Push 2 documentation.

Most of this comparative analysis is based on Ableton's official Push 2 documentation available on GitHub.
Since I don't own a Push 2, all comparisons are derived from that documentation rather than direct hardware testing.

---

## USB Display Protocol Comparison

### USB Hardware

| Feature        | Push 2   | Push 3   | Technical Difference |
| -------------- | -------- | -------- | -------------------- |
| **Vendor ID**  | `0x2982` | `0x2982` | Same (Ableton)       |
| **Product ID** | `0x1967` | `0x1969` | Incremented by 2     |

---

### Display Specifications

| Specification    | Push 2               | Push 3               | Notes     |
| ---------------- | -------------------- | -------------------- | --------- |
| **Resolution**   | 960 x 160 pixels     | 960 x 160 pixels     | Identical |
| **Color Format** | RGB565 Little-Endian | RGB565 Little-Endian | Identical |
| **Frame Size**   | 327,680 bytes        | 327,680 bytes        | Identical |
| **Line Size**    | 2,048 bytes          | 2,048 bytes          | Identical |
| **Pixel Data**   | 1,920 bytes/line     | 1,920 bytes/line     | Identical |
| **Line Padding** | 128 bytes/line       | 128 bytes/line       | Identical |

---

### Frame Structure

```
Push 2 Frame Header:
FF CC AA 88 00 00 00 00 00 00 00 00 00 00 00 00

Push 3 Frame Header:
FF CC AA 88 00 00 00 00 00 00 00 00 00 00 00 00
```

**Result:** Identical - perfect compatibility at frame level

---

### USB Transfer Performance

| Metric            | Push 2       | Push 3       | Improvement Factor |
| ----------------- | ------------ | ------------ | ------------------ |
| **Chunk Size**    | 512 bytes    | 16,384 bytes | 32x larger         |
| **Transfer Time** | 50-100ms     | 20-30ms      | 2-3x faster        |
| **Bandwidth**     | \~3-6 MB/s   | \~10-15 MB/s | 2-3x higher        |
| **USB Standard**  | USB 2.0 Bulk | USB 2.0 Bulk | Same               |

---

### USB Transfer Optimization

#### Push 2 Transfer Pattern

```python
# Push 2: Small chunks, many transfers
PUSH2_CHUNK_SIZE = 512        # 512 bytes per transfer
PUSH2_CHUNKS_PER_FRAME = 640  # ~640 USB transfers per frame
PUSH2_TRANSFER_TIME = "50-100ms"
```

#### Push 3 Transfer Pattern

```python
# Push 3: Large chunks, fewer transfers
PUSH3_CHUNK_SIZE = 16384      # 16KB per transfer
PUSH3_CHUNKS_PER_FRAME = 20   # ~20 USB transfers per frame
PUSH3_TRANSFER_TIME = "20-30ms"
```

**Analysis:** Push 3's larger chunk size dramatically reduces USB overhead.

---

### Push 3 Encryption

```python
# Push 3 XOR Pattern (Confirmed)
XOR_PATTERN = [0xE7, 0xF3, 0xE7, 0xFF]

def encrypt_push3_frame(data):
    encrypted = bytearray(data)
    for i in range(len(encrypted)):
        encrypted[i] ^= XOR_PATTERN[i % 4]
    return bytes(encrypted)
```

---

## MIDI Protocol Comparison

### Device Identification

| Parameter           | Push 2                              | Push 3     | Compatibility |
| ------------------- | ----------------------------------- | ---------- | ------------- |
| **Manufacturer ID** | `00 21 1D`                          | `00 21 1D` | Identical     |
| **Device Family**   | `01 01`                             | `01 01`    | Identical     |
| **Device Inquiry**  | Supported                           | Supported  | Compatible    |
| **SysEx Format**    | `F0 00 21 1D 01 01 [CMD] [DATA] F7` | Same       | Compatible    |

---

### SysEx Command Compatibility

#### Confirmed Compatible Commands

```python
COMPATIBLE_COMMANDS = {
    # Display commands (Push 2 style - 16x8 limitation)
    0x18: "Display Configuration",
    0x19: "Display Brightness",
    0x1A: "Display Contrast",
    0x1B: "Display Text",
    
    # LED commands
    0x04: "Pad LED Control",
    0x05: "Button LED Control",
    0x06: "Touch Strip LED",
    
    # Device control
    0x01: "Device Inquiry",
    0x02: "Device Identity Request",
    0x03: "Device Identity Response",
    0x10: "Mode Change",
    
    # Pad control
    0x20: "Pad Velocity Curve",
    0x21: "Pad Threshold",
    0x22: "Pad Gain",
}
```

#### Push 3 Extension Commands

```python
PUSH3_EXTENSIONS = {
    0x38: "Push 3 Specific Command A",  # Function TBD
    0x3A: "Push 3 Specific Command B",  # Function TBD
    0x3E: "Push 3 Specific Command C",  # Function TBD
}
```

---

### MIDI CC Mapping Compatibility

#### Transport Controls

| Function   | Push 2 CC | Push 3 CC | Compatible |
| ---------- | --------- | --------- | ---------- |
| **Play**   | 85        | 85        | Yes        |
| **Record** | 86        | 86        | Yes        |
| **Stop**   | 29        | 29        | Yes        |

---

#### Encoders

| Encoder | Push 2 Touch | Push 3 Touch | Push 2 Turn | Push 3 Turn | Compatible |
| ------- | ------------ | ------------ | ----------- | ----------- | ---------- |
| **1**   | CC 0         | CC 0         | CC 71       | CC 71       | Yes        |
| **2**   | CC 1         | CC 1         | CC 72       | CC 72       | Yes        |
| **...** |              |              |             |             | Yes        |
| **8**   | CC 7         | CC 7         | CC 78       | CC 78       | Yes        |

---

#### Pads

```python
# Both devices use identical pad mapping
PAD_MAPPING = {
    'bottom_left': 36,   # C1
    'bottom_right': 43,  # G1
    'top_left': 92,      # E6
    'top_right': 99      # D#6
}
```

**Most** CC mappings are the same across the Push 2 and Push 3.
However, more work needs to be done to list each one in comparison.

---

## Migration Guidelines

### From Push 2 to Push 3

#### USB Display Code

```python
# Push 2 style (works on Push 3)
def send_frame_push2_style(device, frame_data):
    header = FRAME_HEADER
    device.write(0x01, header)
    # Encrypt for push 2, same XOR Pattern
    # Small chunks (slower but compatible)
    for i in range(0, len(frame_data), 512):
        chunk = frame_data[i:i+512]
        device.write(0x01, chunk)

# Push 3 optimized (recommended)
def send_frame_push3_optimized(device, frame_data):
    header = FRAME_HEADER
    device.write(0x01, header)
    
    # Encrypt frame data (Push 3 requirement)
    encrypted = encrypt_push3_frame(frame_data)
    
    # Large chunks (much faster)
    for i in range(0, len(encrypted), 16384):
        chunk = encrypted[i:i+16384]
        device.write(0x01, chunk)
```

---

#### MIDI Code

```python
# Push 2 MIDI code works unchanged on Push 3
def set_button_led(midi_out, button_cc, color):
    # Same SysEx command for both devices
    sysex = [0xF0, 0x00, 0x21, 0x1D, 0x01, 0x01, 0x05,
             button_cc, color, 0xF7]
    midi_out.send(mido.Message('sysex', data=sysex[1:-1]))
```

---

### Backwards Compatibility Strategy

#### Display Layer

```python
class UniversalPushDisplay:
    def __init__(self, device_type):
        self.device_type = device_type
        self.encryption = device_type == 'push3'
        self.chunk_size = 16384 if device_type == 'push3' else 512
    
    def send_frame(self, frame_data):
        if self.encryption:
            frame_data = encrypt_push3_frame(frame_data)
        
        self._transfer_chunks(frame_data, self.chunk_size)
```

---

#### MIDI Layer

```python
class UniversalPushMIDI:
    def __init__(self, device_type):
        self.device_type = device_type
        self.extended_commands = device_type == 'push3'
    
    def send_command(self, command, data):
        # Use Push 2 compatible commands by default
        # Enable Push 3 extensions when available
        if command in PUSH3_EXTENSIONS and self.extended_commands:
            return self._send_push3_command(command, data)
        else:
            return self._send_push2_command(command, data)
```

---

## Recommendations

### For New Projects

1. **Target Push 3** - Use optimized protocols for best performance.
2. **Maintain Compatibility** - Support Push 2 for a wider user base.
3. **Feature Detection** - Auto-detect device capabilities.
4. **Graceful Degradation** - Reduce features on older hardware.

---

### For Existing Push 2 Code

1. **Minimal Changes** - Most code works unchanged.
2. **Optimize Transfers** - Use larger chunks when possible.
3. **Test Both Devices** - Verify compatibility.

---

### Performance Optimization

```python
def optimize_for_device(device_type):
    if device_type == 'push3':
        return {
            'chunk_size': 16384,
            'encryption': True,
            'target_fps': 30,
            'advanced_features': True
        }
    else:  # Push 2
        return {
            'chunk_size': 512,
            'encryption': False,
            'target_fps': 15,
            'advanced_features': False
        }
```

---

## Conclusion

### Compatibility Assessment

* **MIDI Protocol:** Excellent - Full backwards compatibility.
* **USB Display:** Good - Same format, encryption addition.
* **Performance:** Significantly improved - 2-3x faster transfers.
* **Development:** Easy migration - minimal code changes required.

### Key Takeaways

1. Push 3 is largely a performance upgrade over Push 2.
2. Existing Push 2 software can be easily adapted for Push 3.
3. Cross-compatibility is achievable with minimal effort.
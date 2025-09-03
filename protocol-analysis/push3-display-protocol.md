# Push 3 USB Display Protocol Analysis

Comprehensive technical analysis of the Push 3 USB display communication protocol, including frame structure, encryption, and performance characteristics.

## Protocol Overview

The Push 3 USB display protocol enables high-resolution graphics display on the device's 960 x 160 pixel screen. This analysis covers the complete communication stack from USB transfer to framebuffer rendering.

### Key Specifications

* **Resolution**: 960 x 160 pixels
* **Color Format**: RGB565 (16-bit per pixel)
* **Frame Size**: 327,680 bytes total
* **Interface**: USB 2.0 Bulk Transfer
* **Encryption**: XOR pattern encryption
* **Performance**: 20-30ms per frame update

## Frame Structure

### Frame Header

Every display frame begins with a 16-byte header:

```
Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
Data:   FF CC AA 88 00 00 00 00 00 00 00 00 00 00 00 00
```

**Header Analysis:**

* `FF CC AA 88`: Magic number/frame synchronization
* Remaining 12 bytes: Reserved/padding (all zeros)
* **Size**: 16 bytes
* **Purpose**: Frame identification and protocol versioning

### Framebuffer Structure

The framebuffer follows immediately after the header:

```
Total Frame Size: 327,680 bytes
├── Header: 16 bytes
└── Framebuffer: 327,664 bytes
    ├── Line 0: 2,048 bytes (1,920 pixel data + 128 padding)
    ├── Line 1: 2,048 bytes
    ├── ...
    └── Line 159: 2,048 bytes
```

### Line Structure

Each display line contains pixel data plus padding:

```
Line Layout (2,048 bytes per line):
├── Pixel Data: 1,920 bytes (960 pixels x 2 bytes/pixel)
└── Padding: 128 bytes (all zeros)
```

**Pixel Format**: RGB565 Little-Endian

* **Red**: 5 bits (bits 11-15)
* **Green**: 6 bits (bits 5-10)
* **Blue**: 5 bits (bits 0-4)

```c
// RGB565 pixel structure
typedef struct {
    uint16_t blue  : 5;
    uint16_t green : 6;
    uint16_t red   : 5;
} rgb565_pixel;
```

## Encryption Analysis

### XOR Pattern Encryption

The Push 3 implements XOR encryption on the framebuffer data (excluding header):

```python
XOR_PATTERN = [0xE7, 0xF3, 0xE7, 0xFF]

def encrypt_framebuffer(data):
    """Apply XOR encryption to framebuffer data"""
    encrypted = bytearray(data)
    for i in range(len(encrypted)):
        encrypted[i] ^= XOR_PATTERN[i % 4]
    return bytes(encrypted)
```

### Encryption Scope

* **Header**: Not encrypted (transmitted as-is)
* **Framebuffer**: XOR encrypted with 4-byte repeating pattern
* **Padding**: Encrypted (though contains only zeros)

### Decryption Process

```python
def decrypt_framebuffer(encrypted_data):
    """Remove XOR encryption from framebuffer data"""
    decrypted = bytearray(encrypted_data)
    for i in range(len(decrypted)):
        decrypted[i] ^= XOR_PATTERN[i % 4]
    return bytes(decrypted)
```

## USB Transfer Protocol

### Device Identification

```python
USB_VENDOR_ID = 0x2982    # Ableton
USB_PRODUCT_ID = 0x1969   # Push 3
```

### Transfer Configuration

* **Interface**: Bulk Transfer (Endpoint 0x01 OUT)
* **Chunk Size**: 16,384 bytes (16KB)
* **Total Chunks**: \~20 chunks per frame
* **Transfer Direction**: Host -> Device (OUT)

### Transfer Sequence

```python
def send_frame_to_push3(device, frame_data):
    """Send complete frame to Push 3 display"""

    # 1. Send frame header
    header = bytes([0xFF, 0xCC, 0xAA, 0x88] + [0x00] * 12)
    device.write(0x01, header)

    # 2. Encrypt framebuffer
    encrypted = encrypt_framebuffer(frame_data)

    # 3. Send in 16KB chunks
    chunk_size = 16384
    for i in range(0, len(encrypted), chunk_size):
        chunk = encrypted[i:i + chunk_size]
        device.write(0x01, chunk)
```

## Performance Analysis

### Transfer Timing

* **Frame Preparation**: <1ms (image conversion and encryption)
* **USB Transfer**: 15-25ms (dependent on USB bus load)
* **Display Update**: <5ms (hardware processing)
* **Total Latency**: 20-30ms per frame

### Optimization Strategies

#### 1. Chunk Size Optimization

```python
# Push 3 supports large chunks (major improvement over Push 2)
OPTIMAL_CHUNK_SIZE = 16384  # 16KB chunks
# vs Push 2: 512 bytes (32x improvement)
```

#### 2. Selective Updates

```python
def update_dirty_regions(old_frame, new_frame):
    """Only update changed regions to reduce transfer time"""
    dirty_lines = []
    for line_num in range(160):
        old_line = get_line_data(old_frame, line_num)
        new_line = get_line_data(new_frame, line_num)
        if old_line != new_line:
            dirty_lines.append(line_num)
    return dirty_lines
```

#### 3. Frame Rate Control

```python
def maintain_framerate(target_fps=30):
    """Maintain consistent frame rate for smooth animation"""
    frame_time = 1.0 / target_fps  # 33.33ms for 30 FPS
    # Transfer time (~25ms) + processing time (~5ms) = ~30ms
    # Natural frame rate limitation around 30-35 FPS
```

## Color Space and Conversion

### RGB888 to RGB565 Conversion

```python
import struct

def rgb888_to_rgb565(r, g, b):
    """Convert 24-bit RGB to 16-bit RGB565"""
    r5 = (r >> 3) & 0x1F
    g6 = (g >> 2) & 0x3F
    b5 = (b >> 3) & 0x1F
    return (r5 << 11) | (g6 << 5) | b5

def rgb565_to_bytes(rgb565):
    """Convert RGB565 value to little-endian bytes"""
    return struct.pack('<H', rgb565)
```

### Image Processing Pipeline

```python
from PIL import Image

def prepare_image_for_push3(image_path):
    """Complete image processing pipeline"""

    # 1. Load and resize image
    img = Image.open(image_path)
    img = img.resize((960, 160), Image.LANCZOS)
    img = img.convert('RGB')

    # 2. Convert to RGB565 framebuffer
    framebuffer = bytearray()
    for y in range(160):
        line_data = bytearray()
        for x in range(960):
            r, g, b = img.getpixel((x, y))
            rgb565 = rgb888_to_rgb565(r, g, b)
            line_data.extend(rgb565_to_bytes(rgb565))

        # Add line padding
        line_data.extend(bytes(128))  # 128 zero bytes
        framebuffer.extend(line_data)

    return bytes(framebuffer)
```

## Implementation Example

### Complete Display Controller

```python
import usb.core
import usb.util
from PIL import Image

class Push3Display:
    def __init__(self):
        self.device = usb.core.find(idVendor=0x2982, idProduct=0x1969)
        if not self.device:
            raise RuntimeError("Push 3 not found")

        # Configure device
        self.device.set_configuration()

    def display_image(self, image_path):
        """Display image on Push 3 screen"""

        # Prepare frame data
        framebuffer = self.prepare_image(image_path)

        # Send frame header
        header = bytes([0xFF, 0xCC, 0xAA, 0x88] + [0x00] * 12)
        self.device.write(0x01, header)

        # Encrypt and send framebuffer
        encrypted = self.encrypt_framebuffer(framebuffer)

        chunk_size = 16384
        for i in range(0, len(encrypted), chunk_size):
            chunk = encrypted[i:i + chunk_size]
            self.device.write(0x01, chunk)

    def prepare_image(self, image_path):
        return prepare_image_for_push3(image_path)

    def encrypt_framebuffer(self, data):
        """Apply XOR encryption"""
        xor_pattern = [0xE7, 0xF3, 0xE7, 0xFF]
        encrypted = bytearray(data)
        for i in range(len(encrypted)):
            encrypted[i] ^= xor_pattern[i % 4]
        return bytes(encrypted)
```

## Troubleshooting

### Common Issues

#### 1. Device Not Found

```python
# Check USB connection and permissions
devices = usb.core.find(find_all=True, idVendor=0x2982)
for device in devices:
    print(f"Found device: {device.idProduct:04x}")
```

#### 2. Transfer Errors

```python
# Handle USB transfer timeouts
try:
    device.write(0x01, data, timeout=1000)
except usb.core.USBTimeoutError:
    print("Transfer timeout - check USB connection")
```

#### 3. Display Corruption

```python
# Verify frame size and encryption
assert len(framebuffer) == 327664, "Invalid framebuffer size"
assert len(encrypted_data) == len(framebuffer), "Encryption error"
```
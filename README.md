# Push 3 Reverse Engineering Documentation

This section contains comprehensive technical documentation of the Ableton Push 3 hardware interface, communication protocols, and control mappings. 

Most of this comparative analysis is based on Ableton’s official Push 2 documentation available on GitHub. Since I don’t own a Push 2, all comparisons are derived from that documentation rather than direct hardware testing.

## Documentation Overview

### Protocol Analysis
- **[Push 2 vs Push 3 Protocol Comparison](protocol-analysis/push2-push3-protocol.md)** - Detailed compatibility analysis highlighting differences and nuances between both devices.
- **[Push 3 USB Display Protocol](protocol-analysis/push3-display-protocol.md)** - Full documentation of the USB framebuffer implementation specific to the Push 3.
- **[Pad Sensitivity Curve Documentation](protocoal-analysis/push2-push3-curve-protocol.md)** - Explanation of how pad sensitivity curves are generated and how they differ between Push 2 and Push 3.

### Interface Mapping
- **[Button Mapping](interface-mapping/buttons.md)** - All 70+ buttons with CC values
- **[Encoder Mapping](interface-mapping/encoders.md)** - 10 encoders with touch detection

### Research Tools
- **[Research Tools](tools)** - Hardware testing and analysis scripts
  - `display_test.py` - USB display testing with included test image
  - `midi_monitor.py` - Real-time MIDI message monitoring and analysis
  - `midi_test.py` - LED control and MIDI functionality testing  
  - `text_renderer.py` - Dynamic parameter display generation

## Key Findings Summary

### USB Display Protocol
- **Resolution**: 960×160 pixels, RGB565 color format
- **Frame Size**: 327,680 bytes (identical to Push 2)
- **Encryption**: XOR pattern `[0xE7, 0xF3, 0xE7, 0xFF]`
- **Performance**: 20-30ms frame updates (2-3x faster than Push 2)
- **Transfer**: 16KB chunks vs 512B in Push 2

### MIDI Control Protocol
- **Compatibility**: Full Push 2 command compatibility
- **Extensions**: New commands 0x38, 0x3A, 0x3E for Push 3
- **Device ID**: Same manufacturer/device ID as Push 2
- **Handshake**: Identical device inquiry/response sequence

## Research Methodology

1. **USB Protocol Analysis**: Using Wireshark, USBPcap for traffic capture
2. **MIDI Analysis**: MIDI Monitor, SysEx analysis tools  
3. **Hardware Testing**: Systematic button/encoder mapping with `midi_monitor.py`
4. **Performance Testing**: Frame rate and latency measurements with `display_test.py`
5. **LED Control Testing**: Visual feedback validation with `midi_test.py`
6. **Display Prototyping**: Logic Pro interface mockups with `text_renderer.py`

# Push 3 Reverse Engineering Tools

Collection of research tools and test scripts used for reverse engineering the Push 3 hardware interface.

## Available Tools

### Display Control
- **`display_test.py`** - Simple tool to display the test image on Push 3
- **`text_renderer.py`** - Create custom text displays for Logic Pro visualization

### MIDI Analysis  
- **`midi_monitor.py`** - Monitor and analyze MIDI input/output from Push 3
- **`midi_test.py`** - Test MIDI control features like LED control

## Usage Examples

### Display Test Image
```bash
# Display the included test image (hllo_wrld.png)
python3 display_test.py

# Enable debug output
python3 display_test.py --debug
```

### Monitor MIDI Messages
```bash
# Monitor all MIDI messages from Push 3
python3 midi_monitor.py

# Monitor with debug info for 30 seconds
python3 midi_monitor.py --debug --duration 30

# Test SysEx communication
python3 midi_monitor.py --test-sysex
```

### Test MIDI Controls
```bash
# Run complete LED test suite
python3 midi_test.py

# Test only button LEDs
python3 midi_test.py --buttons

# Test only pad LEDs  
python3 midi_test.py --pads

# Run LED chase demo
python3 midi_test.py --chase

# Send device inquiry
python3 midi_test.py --inquiry

# Turn off all LEDs
python3 midi_test.py --all-off
```

### Create Text Displays
```bash
# Create parameter display
python3 text_renderer.py --type parameter

# Create mixer display
python3 text_renderer.py --type mixer

# Create transport display
python3 text_renderer.py --type transport
```

## Development Tools

These scripts were essential for the reverse engineering process:

1. **Protocol Analysis** - Understanding USB frame structure and MIDI communication
2. **Encryption Discovery** - Finding XOR pattern through binary analysis  
3. **Performance Testing** - Measuring transfer speeds and latency
4. **Interface Mapping** - Systematic button/encoder testing
5. **LED Control** - Testing visual feedback systems

## ⚠️ Requirements

```bash
# 1. (Recommended) Create a virtual environment
python3 -m venv venv

# 2. Activate the virtual environment
source venv/bin/activate

# 3. Install dependencies
pip install pyusb pillow mido python-rtmidi

# 4. (macOS only) Install libusb if not already installed
brew install libusb
# Linux: Install libusb development packages
sudo apt install libusb-1.0-0-dev

# 5. Run a tool, e.g.:
python3 display_test.py


```

## Tool Details

### display_test.py
- **Purpose**: Verify USB display communication
- **Features**: Hardcoded test image, debug output, error handling
- **Usage**: Basic display functionality testing

### midi_monitor.py  
- **Purpose**: Real-time MIDI message analysis
- **Features**: Human-readable message formatting, button/encoder identification
- **Usage**: Understanding MIDI protocol and mapping controls

### midi_test.py
- **Purpose**: Test MIDI control output (LEDs, SysEx)
- **Features**: LED color testing, device inquiry, chase patterns
- **Usage**: Verify MIDI communication and LED control

### text_renderer.py
- **Purpose**: Generate Logic Pro-style parameter displays
- **Features**: Multiple display types, custom fonts, color schemes
- **Usage**: Prototype Logic Pro integration visuals

## Research Results

These tools helped discover:

- **USB Protocol**: XOR encryption pattern, frame structure, chunk optimization
- **MIDI Compatibility**: Full Push 2 compatibility + Push 3 extensions  
- **Performance**: 2-3x improvement in display update speed
- **Interface Mapping**: Complete button/encoder CC mapping (70+ controls)
- **LED Control**: RGB LED support with brightness control

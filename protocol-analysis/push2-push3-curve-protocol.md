# Push 2 vs Push 3 – Pad Sensitivity Curve Documentation

This document explains how **Ableton Push 2** and **Push 3** handle pad sensitivity curves via SysEx.
It details the differences between the two devices, including message formats, UI parameters, and migration strategies.

Most of this comparative analysis is based on Ableton’s official Push 2 documentation available on GitHub. Since I don’t own a Push 2, all comparisons are derived from that documentation rather than direct hardware testing.

---

## Overview

Both devices use a **128-entry lookup table (LUT)** to map pad force readings to MIDI velocities (1–127).
Each LUT entry is a single **7-bit value (0–127)**.

| Concept            | Description                                                    |
| ------------------ | -------------------------------------------------------------- |
| **LUT Length**     | 128 entries                                                    |
| **Entry Size**     | 1 byte (7-bit safe)                                            |
| **Monotonic Rule** | Values must **never decrease**                                 |
| **Plateau Rule**   | Once a value hits `127`, all following entries must stay `127` |

---

## Executive Summary

| Aspect           | Push 2                           | Push 3                                          | Compatibility             | Notes                                        |
| ---------------- | -------------------------------- | ----------------------------------------------- | ------------------------- | -------------------------------------------- |
| Curve Transport  | `0x20` – Per-entry SysEx message | `0x43` – Bulk SysEx (all 128 bytes)             | Identical LUT format      | Push 3 adds subheader + optional echo bytes  |
| LUT Format       | 128 × 7-bit, monotonic + plateau | Same                                            | Full compatibility        |                                              |
| Parameters (UI)  | Sensitivity, Gain, Dynamics      | Threshold, Drive, Compand, Range                | Maps 3 → 4                | Threshold is new in Push 3                   |
| Backward Support | Only `0x20`                      | **Confirmed:** `0x43`**Unverified:** `0x20` | Safe to migrate to `0x43` | Don’t rely on `0x20` unless you’ve tested it |

---

## SysEx Message Layouts

### Push 2 – Per-entry Write (`0x20`)

Push 2 requires **128 separate messages**, one for each index.

```
F0 00 21 1D            ; SysEx start + Ableton manufacturer ID
01 01 20               ; Device family + CMD 0x20 (Pad Velocity Curve)
<index>                ; Index: 0–127
<value>                ; Value: 0–127 (7-bit)
F7                     ; SysEx end
```

* Sent **128 times** to upload a full curve.
* No subheader or echo bytes.

---

### Push 3 – Bulk Upload (`0x43`)

Push 3 allows sending **the entire LUT at once** in a single SysEx frame.

```
F0 00 21 1D            ; SysEx start + Ableton manufacturer ID
01 01 43               ; Device family + CMD 0x43 (Pad Sensitivity Curve)
01 01 01 01            ; Constant subheader (always required)
[optional 4 bytes]     ; UI echo: Threshold, Drive, Compand, Range (optional)
<128 curve bytes>      ; LUT: monotonic, plateau at 0x7F
F7                     ; SysEx end
```

#### Notes:

* Echo bytes **optional**, only informational.
* LUT must **always** be 128 bytes, 7-bit safe.
* Once a `127` appears, **all following values must be `127`**.

---

## Key Differences

| Element         | Push 2                   | Push 3              |
| --------------- | ------------------------ | ------------------- |
| Command Byte    | `0x20`                   | `0x43`              |
| Subheader       | None                     | `01 01 01 01`       |
| Echo Bytes      | None                     | Optional (4 bytes)  |
| Transfer Method | 128 separate messages    | Single bulk message |
| LUT Rules       | Monotonic + plateau rule | Same                |

**Recommendation:**
Always use `0x43` for Push 3.
Treat `0x20` as **legacy** and only use if you’ve tested it on actual hardware.

---

## UI Parameter Mapping

Push 2 parameters are high-level and implicit. Push 3 exposes more granular control.

| Concept            | Push 2 (UI)         | Push 3 (UI)           | Notes                                |
| ------------------ | ------------------- | --------------------- | ------------------------------------ |
| Initial Dead-zone  | Implicit in presets | **Threshold (0–100)** | Adds explicit control in Push 3      |
| Mid-slope / Gain   | Gain                | **Drive (-50…+50)**   | Similar exponential effect           |
| Start/End Shaping  | Dynamics            | **Compand (-50…+50)** | Controls curve knee shape (logistic) |
| Saturation / Range | Implicit in preset  | **Range (0–100)**     | Where plateau starts                 |

To replicate Push 2 behavior on Push 3:

* Set **Threshold = 0**.
* Tune **Drive, Compand, Range** to approximate original feel.

---

## Curve Generation Model

The LUT is built in **four sequential steps**:

1. **Threshold** – Adds leading dead-zone (near-zero sensitivity).
2. **Drive** – Adjusts overall steepness.
3. **Compand** – Shapes the curve using an S-shape (logistic blend).
4. **Range** – Determines where saturation (`127`) begins.

Processing order: **Threshold → Drive → Compand → Range**

---

## Step-by-Step Curve Functions

Each function returns a normalized list `[0.0 – 1.0]` of 128 values.

### 1. Threshold

```python
def threshold_curve(threshold: int):
    """
    Apply leading dead-zone before velocity starts to rise.
    """
    threshold = max(0, min(100, threshold))
    Nt = round(threshold * 16 / 100.0)  # 0–16 dead-zone entries
    out = []
    for i in range(128):
        if i < Nt:
            out.append(1/127.0)  # Minimal non-zero to avoid zero velocity
        else:
            out.append((i - Nt) / (127 - Nt) if Nt < 127 else 1.0)
    return out
```

---

### 2. Drive

```python
import math

def drive_curve(drive: int):
    """
    Exponential steepness control:
    Negative = flatter, Positive = steeper.
    """
    drive = max(-50, min(50, drive))
    expo = math.exp(-(drive / 50.0))
    return [(i/127.0) ** expo for i in range(128)]
```

---

### 3. Compand

```python
def compand_curve(compand: int):
    """
    S-shaped knee adjustment for soft/hard touches.
    """
    compand = max(-50, min(50, compand))
    a = compand / 50.0

    def blend(u):
        if a == 0.0:
            return u
        s = 1.0 / (1.0 + math.exp(-6.0 * a * (u - 0.5)))
        return (1.0 - abs(a)) * u + abs(a) * s

    return [blend(i/127.0) for i in range(128)]
```

---

### 4. Range

```python
def range_curve(range_: int):
    """
    Determines where values clamp to 1.0 (velocity 127).
    """
    range_ = max(0, min(100, range_))
    A = (range_ / 100.0) or 1e-6
    return [min((i/127.0)/A, 1.0) for i in range(128)]
```

---

## Full LUT Assembly

```python
def build_curve(threshold: int, drive: int, compand: int, range_: int):
    """
    Generates a final 128-entry LUT.
    """
    tmp = threshold_curve(threshold)            # Step 1: Dead-zone
    expo = math.exp(-(drive / 50.0))
    tmp = [x ** expo for x in tmp]              # Step 2: Drive

    # Step 3: Compand
    a = compand / 50.0
    def blend(u):
        if a == 0.0: return u
        s = 1.0 / (1.0 + math.exp(-6.0 * a * (u - 0.5)))
        return (1.0 - abs(a)) * u + abs(a) * s
    tmp = [blend(x) for x in tmp]

    # Step 4: Range
    A = (range_ / 100.0) or 1e-6
    tmp = [min(x / A, 1.0) for x in tmp]

    # Quantize and enforce plateau
    out, plateau = [], False
    for x in tmp:
        v = int(round(127 * x))
        if plateau:
            v = 127
        elif v >= 127:
            v = 127
            plateau = True
        out.append(v)

    # Ensure strictly monotonic
    for i in range(1, 128):
        if out[i] < out[i-1]:
            out[i] = out[i-1]

    return out
```

---

## SysEx Packing

### Push 2 – Per-entry

```python
def to_sysex_push2_entry(index, value):
    """
    Build a single SysEx message for one LUT entry (Push 2 style).
    """
    assert 0 <= index <= 127 and 0 <= value <= 127
    return [0xF0,0x00,0x21,0x1D,0x01,0x01,0x20, index, value, 0xF7]
```

---

### Push 3 – Bulk

```python
def to_sysex_push3(curve, echo_params=None):
    """
    Build a full bulk SysEx message for Push 3.
    """
    assert len(curve) == 128 and all(0 <= v <= 127 for v in curve)

    frame = [
        0xF0,0x00,0x21,0x1D,  # SysEx + manufacturer
        0x01,0x01,0x43,        # Device + command
        0x01,0x01,0x01,0x01    # Required subheader
    ]

    # Optional echo bytes: [Threshold, Drive, Compand, Range]
    if echo_params:
        if len(echo_params) != 4:
            raise ValueError("echo_params must have 4 values")
        frame.extend([max(0, min(127, int(v))) for v in echo_params])

    frame.extend(curve)
    frame.append(0xF7)
    return frame
```

---

## Minimal Send Example

```python
import mido

curve = build_curve(threshold=0, drive=4, compand=-26, range_=38)
frame = to_sysex_push3(curve)

with mido.open_output("Ableton Push 3") as out:
    # mido strips F0/F7 automatically
    out.send(mido.Message('sysex', data=frame[1:-1]))
```

---

## Validation Before Sending

```python
# Monotonic rule
assert all(curve[i] >= curve[i-1] for i in range(1, 128))

# Plateau rule
first_127 = next((i for i, v in enumerate(curve) if v == 127), None)
if first_127 is not None:
    assert all(v == 127 for v in curve[first_127:])
```

---

## Migration Guidelines

1. **Direct LUT Transfer**

   * **Try** to take a Push 2 LUT and send it directly to Push 3 inside a `0x43` bulk frame.
   * I could not test this, as I dont own a Push 2.

2. **Parameter Matching**

   * On Push 3, set **Threshold = 0** to emulate Push 2 behavior.
   * Adjust Drive, Compand, and Range to approximate the original feel.

3. **Legacy `0x20` on Push 3**

   * Whether Push 3 supports the old per-entry `0x20` messages is **unverified**.
   * Always prefer bulk `0x43`.

---

## Final Comparison

| Item                | Push 2                                     | Push 3              |
| ------------------- | ------------------------------------------ | ------------------- |
| SysEx Command       | `0x20`                                     | `0x43`              |
| Extra Header Bytes  | None                                       | `01 01 01 01`       |
| Optional Echo Bytes | None                                       | 4 bytes (optional)  |
| Transfer Method     | 128 separate messages                      | Single bulk message |
| LUT Format          | Identical (128×7-bit, monotonic + plateau) | Same                |

---

## Key Takeaways

1. **LUT format is identical** -> migration is straightforward.
2. **Push 3 framing adds metadata** -> new subheader + optional echo bytes.
3. **Push 3 bulk uploads are faster and easier** than Push 2 per-entry writes.
4. **Always validate curves** before sending to avoid corrupted input.
5. **Future-proof new code** -> use `0x43` and avoid relying on legacy `0x20`.
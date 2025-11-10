# Examples

This document provides practical examples of I2C communication sequences and implementation code.

## Communication Examples

### Example 1: Basic Initialization Sequence

This is what the THREE does when opening the turntable:

```
Step 1: Open I2C connection to address 0x45
Step 2: Verify turntable is responding and booted

Master → Slave: [0x02] [0x05]
                 CMD    CRC
                 (STATUS_W_POS read request)

Master ← Slave: [0x80] [0x00] [0x00] [0x87]
                 Status Pos_L  Pos_H  CRC
                 
Status = 0x80 (STAT_BOOT set, not rotating, no errors)
Position = 0x0000 (0 degrees)

Step 3: Set position to 0° (zero/calibration)

Master → Slave: [0x03] [0x00] [0x00] [0x5A]
                 POSITION 0     0     CRC

Step 4: Set ramp distance to 15°

Master → Slave: [0x08] [0x0F] [0x08]
                 RAMP_DIST 15   CRC

Step 5: Verify position is 0°

Master → Slave: [0x02] [0x05]
Master ← Slave: [0x80] [0x00] [0x00] [0x87]

Initialization complete. Turntable ready for scanning operations.
```

### Example 2: Simple Rotation (0° → 90°)

```
Initial state: Turntable at 0°

Step 1: Command rotation to 90°

Master → Slave: [0x04] [0x5A] [0x00] [0x51]
                 ROTATE_ABS 90  0     CRC

Step 2: Poll status every 100ms

Poll 1 (at 100ms):
Master → Slave: [0x02] [0x05]
Master ← Slave: [0xC0] [0x1E] [0x00] [0xA1]
                 Status Pos_L  Pos_H  CRC

Status = 0xC0 (STAT_BOOT | STAT_TURN) - rotating
Position = 0x001E (30 degrees) - in progress

Poll 2 (at 200ms):
Master → Slave: [0x02] [0x05]
Master ← Slave: [0xC0] [0x3C] [0x00] [0xBF]

Position = 0x003C (60 degrees) - still rotating

Poll 3 (at 300ms):
Master → Slave: [0x02] [0x05]
Master ← Slave: [0x80] [0x5A] [0x00] [0xC9]

Status = 0x80 (STAT_BOOT only) - rotation complete
Position = 0x005A (90 degrees) - at target

Rotation successful!
```

### Example 3: Full Rotation with Wraparound (270° → 30°)

```
Current position: 270°
Target position: 30°
Shortest path: 270° → 360°/0° → 30° (120° clockwise)

Step 1: Command rotation

Master → Slave: [0x04] [0x1E] [0x00] [0x75]
                 ROTATE_ABS 30  0     CRC

Step 2: Monitor rotation through 0° boundary

Poll 1:
Master ← Slave: [0xC0] [0x12] [0x01] [0x93]
Position = 0x0112 (274°)

Poll 2:
Master ← Slave: [0xC0] [0x2D] [0x01] [0xAE]
Position = 0x012D (301°)

Poll 3:
Master ← Slave: [0xC0] [0x48] [0x01] [0xC9]
Position = 0x0148 (328°)

Poll 4:
Master ← Slave: [0xC0] [0x08] [0x00] [0x29]
Position = 0x0008 (8°) - wrapped through 0°

Poll 5:
Master ← Slave: [0x80] [0x1E] [0x00] [0xB1]
Position = 0x001E (30°) - complete!
```

### Example 4: Emergency Stop During Rotation

```
Scenario: User presses stop button while turntable is rotating

Initial state: Rotating from 0° to 180°
Current position: ~90° (halfway)

Step 1: Send stop command

Master → Slave: [0x00] [0x07]
                 STOP_ROT CRC

Turntable should:
- Stop motor immediately
- Clear STAT_TURN flag
- Maintain current position

Step 2: Verify stopped

Master → Slave: [0x02] [0x05]
Master ← Slave: [0x80] [0x5C] [0x00] [0xCB]

Status = 0x80 (STAT_BOOT only, not rotating)
Position = 0x005C (92°) - stopped at current position
```

### Example 5: Error Handling - Rotation Timeout

```
Scenario: Turntable is jammed and cannot complete rotation

Step 1: Command rotation to 180°

Master → Slave: [0x04] [0xB4] [0x00] [0x43]
                 ROTATE_ABS 180 0    CRC

Step 2: Turntable starts moving but gets stuck at 45°

Poll 1 (100ms):
Master ← Slave: [0xC0] [0x1E] [0x00] [0xA1]
Position = 30° - moving

Poll 2 (200ms):
Master ← Slave: [0xC0] [0x2D] [0x00] [0xB0]
Position = 45° - moving

Poll 3-20 (300-2000ms):
Master ← Slave: [0xC0] [0x2D] [0x00] [0xB0]
Position = 45° - stuck!

After 2000ms with no position change:
Turntable sets ERR_ROT_TIME

Poll 21 (2100ms):
Master → Slave: [0x02] [0x05]
Master ← Slave: [0xC1] [0x2D] [0x00] [0xB1]

Status = 0xC1 (STAT_BOOT | STAT_TURN | STAT_ERR)

Step 3: Read error code

Master → Slave: [0x0B] [0x0C]
                 ERROR  CRC

Master ← Slave: [0x08] [0x0F]
                 ERR    CRC

Error = 0x08 (ERR_ROT_TIME)

Step 4: THREE attempts recovery

THREE reduces ramp distance:
Master → Slave: [0x08] [0x05] [0x02]
                 RAMP_DIST 5   CRC

THREE retries rotation:
Master → Slave: [0x04] [0xB4] [0x00] [0x43]
                 ROTATE_ABS 180 0    CRC

If still fails after 3 attempts, THREE throws exception.
```

### Example 6: CRC Error Handling

```
Scenario: Noisy I2C bus causes CRC error

Step 1: Master sends command with corrupted data

Master → Slave: [0x04] [0xB4] [0x00] [0xFF]
                 ROTATE_ABS 180 0    Wrong_CRC!

Turntable calculates:
Expected CRC = 0x43
Received CRC = 0xFF

Step 2: Turntable sets error

Turntable sets: error_code |= ERR_BAD_COM
                status_byte |= STAT_ERR

Step 3: Command is NOT executed (position unchanged)

Step 4: Master detects error on next poll

Master → Slave: [0x02] [0x05]
Master ← Slave: [0x81] [0x00] [0x00] [0x86]

Status = 0x81 (STAT_BOOT | STAT_ERR)

Step 5: Master reads error

Master → Slave: [0x0B] [0x0C]
Master ← Slave: [0x02] [0x05]

Error = 0x02 (ERR_BAD_COM)

Step 6: Master reports error and stops operation

THREE throws exception: "Communication error. Data may have been corrupted"
```

### Example 7: Reading Status During Active Rotation

```
Turntable rotating from 0° to 359° (359° travel counter-clockwise)

Poll every 100ms:

T=0ms:    Status=0xC0, Position=0°   (just started)
T=100ms:  Status=0xC0, Position=340° (rotating CCW)
T=200ms:  Status=0xC0, Position=320°
T=300ms:  Status=0xC0, Position=300°
T=400ms:  Status=0xC0, Position=280°
T=500ms:  Status=0xC0, Position=260°
T=600ms:  Status=0xC0, Position=240°
T=700ms:  Status=0xC0, Position=220°
T=800ms:  Status=0xC0, Position=200°
T=900ms:  Status=0xC0, Position=180°
T=1000ms: Status=0xC0, Position=160°
T=1100ms: Status=0xC0, Position=140°
T=1200ms: Status=0xC0, Position=120°
T=1300ms: Status=0xC0, Position=100°
T=1400ms: Status=0xC0, Position=80°
T=1500ms: Status=0xC0, Position=60°
T=1600ms: Status=0xC0, Position=40°
T=1700ms: Status=0xC0, Position=20°
T=1800ms: Status=0xC0, Position=10°  (decelerating)
T=1900ms: Status=0xC0, Position=3°   (slowing)
T=2000ms: Status=0x80, Position=359° (complete!)

Total time: ~2 seconds for 359° rotation
Average speed: ~180°/second
```

## Code Examples

### Example Implementation: CRC Functions

```c
// CRC-8 calculation for write operations (forward order)
uint8_t calculate_crc_forward(const uint8_t* data, uint8_t length) {
    const uint8_t generator = 0x07;
    uint8_t crc = 0;
    
    for (uint8_t i = 0; i < length; i++) {
        crc ^= data[i];
        for (uint8_t bit = 0; bit < 8; bit++) {
            if (crc & 0x80) {
                crc = (crc << 1) ^ generator;
            } else {
                crc <<= 1;
            }
        }
    }
    return crc;
}

// CRC-8 calculation for read operations (reverse order)
uint8_t calculate_crc_reverse(const uint8_t* data, uint8_t length) {
    const uint8_t generator = 0x07;
    uint8_t crc = 0;
    
    // Process bytes in reverse order
    for (int8_t i = length - 1; i >= 0; i--) {
        crc ^= data[i];
        for (uint8_t bit = 0; bit < 8; bit++) {
            if (crc & 0x80) {
                crc = (crc << 1) ^ generator;
            } else {
                crc <<= 1;
            }
        }
    }
    return crc;
}

// Validate received command
bool validate_command_crc(const uint8_t* data, uint8_t length) {
    if (length < 2) return false;  // Need at least command + CRC
    
    uint8_t received_crc = data[length - 1];
    uint8_t calculated_crc = calculate_crc_forward(data, length - 1);
    
    return (received_crc == calculated_crc);
}

// Prepare response with CRC
void prepare_response(uint8_t* response, uint8_t data_length) {
    response[data_length] = calculate_crc_reverse(response, data_length);
}
```
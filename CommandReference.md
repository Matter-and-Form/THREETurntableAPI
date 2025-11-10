# Command Reference

This document provides detailed specifications for each command in the turntable protocol and how it would be used (read or write) from the perspective of the THREE i2c master.

## Command Summary Table

| Register | Name         | Direction | Data Bytes | CRC | Description                 |
|----------|--------------|-----------|------------|-----|-----------------------------|
|   0x00   | STOP_ROT     | Write     | 0          |  ✓  | Stop rotation immediately   |
|   0x02   | STATUS_W_POS | Read      | 3          |  ✓  | Read status and position    |
|   0x03   | POSITION     | Write     | 2          |  ✓  | Set absolute position       |
|   0x04   | ROTATE_ABS   | Write     | 2          |  ✓  | Rotate to absolute position |
|   0x08   | RAMP_DIST    | Write     | 1          |  ✓  | Set ramp distance           |
|   0x0B   | ERROR        | Read      | 1          |  ✓  | Read and clear error code   |

----------------------------------------------------------------------------------------

## 0x00 - STOP_ROT (Stop Rotation)

**Type**: Write Command  
**Purpose**: Immediately halt any ongoing rotation

### Command Format
```
[0x00] [CRC]
```

### Parameters
None

### Response
No immediate response. Status can be checked with STATUS_W_POS command.

### Firmware Behavior
- Stops the motor immediately
- Clears STAT_TURN flag in status byte
- May set STAT_HALTED flag (optional, THREE ignores this)
- Does not change the current position
- Should respond within typical I2C timing (microseconds)

### Example
```
Master writes: [0x00] [0x07]
               Command  CRC
```

### Notes
- Used to abort ongoing rotations
- Turntable should be able to receive this command at any time

---

## 0x02 - STATUS_W_POS (Status with Position)

**Type**: Read Command  
**Purpose**: Read current status byte and position in a single transaction

### Command Format
```
Write: [0x02] [CRC]
Read:  [STATUS] [POS_LOW] [POS_HIGH] [CRC]
```

### Parameters
None (write)

### Response
- **Byte 0**: Status byte (see Status Byte section in Protocol Specification)
- **Byte 1**: Position low byte (bits 0-7)
- **Byte 2**: Position high byte (bits 8-15)
- **Byte 3**: CRC (calculated in reverse order over bytes 2, 1, 0)

### Firmware Behavior
- Returns current status and position atomically
- Does not modify any state
- Should respond immediately with current values

### Example
```
Master writes: [0x02] [0x05]
               Command  CRC

Turntable at 90° (0x005A), rotating, boot complete:
Master reads:  [0xC0] [0x5A] [0x00] [0x5E]
               Status  Pos_L  Pos_H  CRC
               
Status 0xC0 = STAT_BOOT (0x80) | STAT_TURN (0x40)
```

### CRC Calculation for Response
```c
// Reverse order: POS_HIGH, POS_LOW, STATUS
data[] = {0x00, 0x5A, 0xC0}
CRC = calculate_reverse_crc(data, 3)
```

### Notes
- This is the most frequently used command
- The THREE polls this every 100ms during rotation operations
- The THREE will retry many times if CRC validation fails
- Critical for tracking rotation progress

---

## 0x03 - POSITION (Set Position)

**Type**: Write Command  
**Purpose**: Set the current absolute position without moving the motor

### Command Format
```
[0x03] [POS_LOW] [POS_HIGH] [CRC]
```

### Parameters
- **POS_LOW**: Position low byte (bits 0-7)
- **POS_HIGH**: Position high byte (bits 8-15)
- **Position Range**: 0-359 degrees

### Response
No immediate response. New position can be verified with STATUS_W_POS command.

### Firmware Behavior
- Sets the internal position counter to the specified angle
- Does NOT move the motor
- Used for calibration/zeroing the position
- Values ≥ 360 should be taken modulo 360
- Should clear STAT_HALTED flag if set

### Example
```
Set position to 0° (zero/home position):
Master writes: [0x03] [0x00] [0x00] [0x5A]
               Cmd    Pos_L  Pos_H  CRC

Set position to 180°:
Master writes: [0x03] [0xB4] [0x00] [0x43]
               Cmd    Pos_L  Pos_H  CRC
```

### Notes
- The THREE calls this on open (turntable physical connection or scan start) to initialize position to 0°
- Should execute immediately without motor movement

---

## 0x04 - ROTATE_ABS (Rotate to Absolute Position)

**Type**: Write Command  
**Purpose**: Begin rotation to an absolute target position

### Command Format
```
[0x04] [POS_LOW] [POS_HIGH] [CRC]
```

### Parameters
- **POS_LOW**: Target position low byte (bits 0-7)
- **POS_HIGH**: Target position high byte (bits 8-15)
- **Target Range**: 0-359 degrees

### Response
No immediate response. Rotation status tracked via STATUS_W_POS command.

### Firmware Behavior
- Begins rotating to the specified absolute position
- Sets STAT_TURN flag immediately
- Should choose shortest angular path (≤180° travel)
- Applies ramp distance for deceleration
- Clears STAT_TURN when target position is reached
- Should clear STAT_HALTED flag if set
- May take several seconds to complete depending on distance

### Rotation Direction
The turntable should automatically determine the shortest path:
- If target is 0-180° clockwise from current position → rotate clockwise
- If target is 0-180° counter-clockwise from current position → rotate counter-clockwise
- For exactly 180° difference, either direction is acceptable

### Example
```
Rotate to 270°:
Master writes: [0x04] [0x0E] [0x01] [0x9F]
               Cmd    Pos_L  Pos_H  CRC

Status while rotating:
[STATUS] includes STAT_TURN (0x40) set

Status when complete:
[STA~10 seconds
Position reads 270° (0x010E)
```

### Timing Expectations
- **Completion Time**: THREE will time out if rotation takes too long. ~10 seconds
- **Timeout**: The THREE expects completion or progress within 2 seconds
- **Polling**: The THREE polls position every 100ms during rotation
- **Recovery**: If no progress for 2 seconds, turntable should set ERR_ROT_TIME

### Error Conditions
- **ERR_ROT_TIME**: Set if rotation doesn't complete or make progress within 2 seconds
- **ERR_ROT_DIR**: Set if rotation direction is incorrect (wiring issue)
- **Position Mismatch**: THREE expects final position to exactly match target

### Notes
- This is the primary motion command
- The THREE may send this command multiple times for error recovery
- The THREE adjusts ramp distance during recovery attempts
- Turntable should handle rapid retries gracefully

---

## 0x08 - RAMP_DIST (Set Ramp Distance)

**Type**: Write Command  
**Purpose**: Configure the deceleration distance for rotation operations

### Command Format
```
[0x08] [RAMP_ANGLE] [CRC]
```

### Parameters
- **RAMP_ANGLE**: Deceleration distance in degrees (8-bit unsigned)
- **Range**: 5-255 degrees (recommended: 5-15 for normal operation)

### Response
No immediate response.

### Firmware Behavior
- Sets the angular distance before target position to begin decelerating
- Smaller values = sharper deceleration, higher overshoot risk
- Larger values = smoother deceleration, slower operation
- Should accept values outside 5-255 range but clamp as needed
- Setting persists until changed or device reset

### Deceleration Profile
The THREE expects smooth deceleration, typically:
- Quadratic ramp: speed ∝ (distance_to_target)²
- Linear ramp: speed ∝ distance_to_target
- Or similar smooth profile avoiding abrupt stops

### Example
```
Set ramp distance to 15° (default):
Master writes: [0x08] [0x0F] [0x08]
               Cmd    Ramp   CRC

Set ramp distance to 5° (aggressive):
Master writes: [0x08] [0x05] [0x02]
               Cmd    Ramp   CRC
```

### Dynamic Adjustment
The THREE will adjust ramp distance during error recovery:
```
Initial rotation:     RAMP_DIST = 15°
If timeout/error:     RAMP_DIST = 5-15° (based on remaining distance)
Multiple retries:     RAMP_DISTa djusted down to minimum 5°
```

### Notes
- Default value on THREE is 15 degrees
- The THREE may set this multiple times during a single rotation for recovery
- Turntable should apply the most recent value to ongoing rotations

---

## 0x0B - ERROR (Read Error Code)

**Type**: Read Command  
**Purpose**: Read the current error code and clear error state.

### Command Format
```
Write: [0x0B] [CRC]
Read:  [ERROR_CODE] [CRC]
```

### Parameters
None (write)

### Response
- **Byte 0**: Error code byte (see Error Codes in Protocol Specification)
- **Byte 1**: CRC (calculated in reverse order, just the error byte)

### Firmware Behavior
- Returns the current accumulated error code
- **Clears all error flags** after successful read
- Clears STAT_ERR flag in status byte
- Multiple errors can be set simultaneously (bits OR'd)
- Should respond immediately

### Example
```
Read error (rotation timeout):
Master writes: [0x0B] [0x0C]
               Cmd    CRC

Turntable responds: [0x08] [0x0F]
                    Error  CRC
                    
Error 0x08 = ERR_ROT_TIME
```

### Error Code Meanings

| Code | Mask | Name | THREE Response |
|------|------|------|----------------|
| 0x01 | ERR_PARAM_COUNT | Wrong number of bytes | Throws exception, stops operation |
| 0x02 | ERR_BAD_COM | CRC/bus error | Throws exception, stops operation |
| 0x04 | ERR_UNRECOGNIZED_COM | Unknown command | Throws exception, stops operation |
| 0x08 | ERR_ROT_TIME | Rotation timeout | Attempts recovery up to 3 times |
| 0x10 | ERR_ROT_DIR | Wrong rotation direction | Throws exception, stops operation |

### Multiple Errors Example
```
If both timeout and bad communication occur:
ERROR_CODE = 0x08 | 0x02 = 0x0A
```

### Notes
- The THREE checks for errors whenever it reads status
- Reading ERROR clears the errors, allowing operation to continue
- ERR_ROT_TIME is special - the THREE will attempt automatic recovery
- All other errors cause the THREE to throw exceptions and halt
- After reading ERROR, subsequent reads should return 0x00 until new errors occur

---

## Command Sequencing

### Typical Operation Sequence

**Initialization:**
```
1. Open I2C to address 0x45
2. Read STATUS_W_POS → verify STAT_BOOT is set
3. Write POSITION 0° → set home position
4. Write RAMP_DIST 15° → set default ramp
5. Read STATUS_W_POS → verify position is 0°
```

**Normal Rotation:**
```
1. Write ROTATE_ABS target_angle → begin rotation
2. Read STATUS_W_POS → check progress (STAT_TURN should be set)
3. Poll STATUS_W_POS every 100ms → track position
4. Continue polling until STAT_TURN clears and position = target
```

**Error Handling:**
```
1. Read STATUS_W_POS → STAT_ERR is set
2. Read ERROR → get error code, clears errors
3. If ERR_ROT_TIME: adjust RAMP_DIST and retry ROTATE_ABS
4. If other errors: report to user, stop operation
```

**Shutdown:**
```
1. Write STOP_ROT → halt any motion
2. Close I2C connection
```

---

**Next**: See [Integration Guide](IntegrationGuide.md) for implementation guidance and requirements.

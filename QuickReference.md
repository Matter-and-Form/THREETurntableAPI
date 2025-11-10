# Quick Reference Card

A quick reference for the THREE Turntable I2C Protocol.

## I2C Configuration
- **Slave Address (7-bit)**: `0x45`
- **Bus Speed**: 10 kHz
- **CRC Polynomial**: `0x07`

## Commands Quick Reference

| Cmd | Name | Write Format | Read Format |
|-----|------|--------------|-------------|
| 0x00 | STOP_ROT | `[0x00][CRC]` | - |
| 0x02 | STATUS_W_POS | `[0x02][CRC]` | `[STATUS][POS_L][POS_H][CRC]` |
| 0x03 | POSITION | `[0x03][POS_L][POS_H][CRC]` | - |
| 0x04 | ROTATE_ABS | `[0x04][POS_L][POS_H][CRC]` | - |
| 0x08 | RAMP_DIST | `[0x08][RAMP][CRC]` | - |
| 0x0B | ERROR | `[0x0B][CRC]` | `[ERROR][CRC]` |

## Status Byte Flags

| Bit | Mask | Name | Description |
|-----|------|------|-------------|
| 0 | 0x01 | STAT_ERR | Error occurred |
| 1 | 0x02 | STAT_BACKLASH | Backlash active (ignored by THREE) |
| 2 | 0x04 | STAT_HALTED | Halted (ignored by THREE) |
| 3 | 0x08 | STAT_RESTING | Resting (ignored by THREE) |
| 6 | 0x40 | STAT_TURN | Currently rotating |
| 7 | 0x80 | STAT_BOOT | Boot complete (required) |

## Error Code Flags

| Bit | Mask | Name | THREE Action |
|-----|------|------|--------------|
| 0 | 0x01 | ERR_PARAM_COUNT | Throws exception |
| 1 | 0x02 | ERR_BAD_COM | Throws exception |
| 2 | 0x04 | ERR_UNRECOGNIZED_COM | Throws exception |
| 3 | 0x08 | ERR_ROT_TIME | Attempts recovery (3x) |
| 4 | 0x10 | ERR_ROT_DIR | Throws exception |

## Position Encoding
- **Format**: 16-bit little-endian
- **Range**: 0-359 degrees
- **Example**: 90° = `[0x5A][0x00]`

## CRC Calculation

### Write (Forward)
```c
crc = 0
for each byte in [CMD, DATA...]:
    crc ^= byte
    for 8 bits:
        if (crc & 0x80): crc = (crc << 1) ^ 0x07
        else: crc <<= 1
```

### Read (Reverse)
```c
crc = 0
for each byte in [DATA_N, ..., DATA_1, DATA_0] (reverse):
    crc ^= byte
    for 8 bits:
        if (crc & 0x80): crc = (crc << 1) ^ 0x07
        else: crc <<= 1
```

## Timing Requirements
- **Poll Period**: 100ms (during rotation)
- **Rotation Timeout**: 2000ms (no progress)
- **Command Response**: <1ms (for reads)
- **Recovery Retries**: 3 attempts (for ERR_ROT_TIME)

## Typical Operation Flow

### Initialization
```
1. Read STATUS_W_POS → verify STAT_BOOT set
2. Write POSITION 0° → set home
3. Write RAMP_DIST 15° → set ramp
4. Read STATUS_W_POS → verify position
```

### Rotation
```
1. Write ROTATE_ABS target
2. Poll STATUS_W_POS every 100ms
3. Wait for STAT_TURN to clear
4. Verify position = target
```

### Error Recovery
```
1. Read STATUS_W_POS → STAT_ERR set
2. Read ERROR → get code, clear error
3. If ERR_ROT_TIME:
   - Write RAMP_DIST (lower value)
   - Write ROTATE_ABS (retry)
   - Repeat up to 3 times
4. If other errors: stop operation
```

## Implementation Checklist

### Required Features
- ✅ I2C slave at address 0x45
- ✅ All 6 commands implemented
- ✅ CRC validation on all messages
- ✅ STAT_BOOT set after initialization
- ✅ STAT_TURN set/cleared during rotation
- ✅ Position tracking with 1° resolution
- ✅ Shortest path rotation (≤180° travel)
- ✅ Ramp distance deceleration
- ✅ Timeout detection (2 seconds)
- ✅ Error reporting via ERROR register
- ✅ Position wraparound at 0°/359°

### Optional Features
- ⭕ STAT_BACKLASH flag
- ⭕ STAT_HALTED flag  
- ⭕ STAT_RESTING flag
- ⭕ Physical homing routine
- ⭕ EEPROM position storage

## Test Vectors

### Commands
```
STOP_ROT:          [0x00][0x07]
POSITION 0°:       [0x03][0x00][0x00][0x5A]
POSITION 90°:      [0x03][0x5A][0x00][0x29]
ROTATE_ABS 180°:   [0x04][0xB4][0x00][0x43]
RAMP_DIST 15:      [0x08][0x0F][0x08]
ERROR read:        [0x0B][0x0C]
```

### Responses
```
STATUS boot, 0°, not rotating:  [0x80][0x00][0x00][0x87]
STATUS boot, 90°, rotating:     [0xC0][0x5A][0x00][0x5E]
ERROR timeout:                  [0x08][0x0F]
```

## Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| "Turntable did not boot" | STAT_BOOT not set | Set bit 7 (0x80) in status |
| Rotation never completes | STAT_TURN not cleared | Clear when position = target |
| Position drifts | Encoder missed counts | Use interrupts, check wiring |
| Frequent timeouts | Too slow or stuck | Increase speed, check mechanics |
| CRC failures | Wrong algorithm | Use 0x07 poly, check forward/reverse |
| Can't reach position | Poor accuracy | Tune ramp, reduce backlash |

## Support Resources

For complete documentation, see:
- [README.md](README.md) - Overview and getting started
- [ProtocolSpecification.md](ProtocolSpecification.md) - Complete protocol details
- [CommandReference.md](CommandReference.md) - Detailed command specifications
- [IntegrationGuide.md](IntegrationGuide.md) - Implementation guidance
- [Examples.md](Examples.md) - Code samples and test scenarios

---

**Version**: 1.0  
**Date**: November 2025  
**Compatible with**: Matter and Form THREE Scanner

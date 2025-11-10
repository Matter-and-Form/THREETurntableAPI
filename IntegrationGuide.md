# Integration Guide

This guide provides practical information for implementing a turntable compatible with the THREE scanner.

## Hardware Requirements

### Required Components

1. **Microcontroller with I2C Slave Support**
   - Must support I2C slave mode
   - Recommended: ATtiny404, ATmega series, STM32, ESP32, or similar
   - Sufficient processing power for position tracking and motor control

2. **Position Sensing**
   - Rotary encoder (quadrature) - recommended
   - Absolute encoder
   - Or equivalent position feedback with 1-degree resolution
   - Must provide 360 distinct positions (0-359°)

3. **Motor and Drive System**
   - Stepper motor with driver, or
   - DC motor with H-bridge driver
   - Must provide smooth, controlled rotation
   - Should support variable speed for ramping

4. **I2C Interface**
   - SDA and SCL lines
   - Pull-up resistors (typically 4.7kΩ) if not provided by THREE
   - I2C slave address must be configurable to 0x45 (7-bit)

### Optional Components
- Limit switches or home sensor (for calibration)
- LED indicators for status
- Backlash compensation mechanisms

## Electrical Interface

### I2C Connection

| Pin | Name | Value |
|------------|------|-------|
| 1 | PWR | 12V Supply from THREE     |
| 2 | GND | Ground |
| 3 | DET | 5V Detection (Active pull to GND) |
| 4 | SCL | 5V Clock |
| 5 | SDA | 5V Data |

### I2C Specifications
- **Voltage Level**: 5V
- **Speed**: 10 kHz due to long cables on the turntable
- **Pull-ups**: 2.2K to 5V on THREE's side
- **Cable Length**: Max 1 meter

## Firmware Implementation

### Initialization Sequence

Your turntable firmware should perform these steps on power-up:

```
1. Initialize I2C slave at address 0x45
2. Initialize motor control system
3. Initialize position encoder
4. Perform self-test:
   - Verify encoder is working
   - Test motor in both directions
   - Verify motor rotates encoder correctly
5. Set initial position (0° or read from encoder)
6. Set STAT_BOOT flag in status byte
7. Enter main loop ready to receive commands
```

**Critical**: The THREE will not operate until STAT_BOOT (0x80) is set in the status byte.

### Suggestions

#### Position Tracking

Accurate position tracking is essential:

```c
// Position must be maintained continuously
uint16_t current_position = 0;  // 0-359 degrees

// Update on encoder changes
void on_encoder_change(int8_t direction) {
    current_position += direction;
    
    // Wrap to 0-359 range
    if (current_position >= 360) {
        current_position -= 360;
    } else if (current_position < 0) {
        current_position += 360;
    }
}
```

#### Rotation Control

Implement rotation with these characteristics:

**Speed Ramping**:
```c
// Example quadratic ramp profile
float calculate_speed(int distance_to_target, int ramp_distance) {
    if (distance_to_target <= 0) {
        return 0.0;  // At target, stop
    }
    
    if (distance_to_target > ramp_distance) {
        return max_speed;  // Full speed region
    }
    
    // Ramp region: quadratic deceleration
    float ratio = (float)distance_to_target / ramp_distance;
    return max_speed * ratio * ratio;
}
```

**Direction Selection**:
```c
// Choose shortest path
int calculate_direction(int current, int target) {
    int diff = target - current;
    
    // Normalize to -180 to +180
    if (diff > 180) {
        diff -= 360;
    } else if (diff < -180) {
        diff += 360;
    }
    
    return (diff >= 0) ? 1 : -1;  // 1=CW, -1=CCW
}
```

**Timeout Detection**:
```c
// Detect if rotation is stuck
unsigned long last_position_change = millis();
const unsigned long TIMEOUT_MS = 2000;

void check_timeout() {
    if (is_rotating && 
        (millis() - last_position_change) > TIMEOUT_MS) {
        // Set error flag
        error_code |= ERR_ROT_TIME;
        status_byte |= STAT_ERR;
        stop_motor();
    }
}
```

#### Error Handling

Implement robust error detection:

**CRC Validation**:
```c
bool validate_command(uint8_t* data, uint8_t length) {
    uint8_t received_crc = data[length - 1];
    uint8_t calculated_crc = calculate_crc(data, length - 1);
    
    if (received_crc != calculated_crc) {
        error_code |= ERR_BAD_COM;
        status_byte |= STAT_ERR;
        return false;
    }
    return true;
}
```

**Parameter Validation**:
```c
void process_command(uint8_t command, uint8_t* params, uint8_t param_count) {
    switch(command) {
        case ROTATE_ABS:
            if (param_count != 3) {  // 2 data bytes + 1 CRC
                error_code |= ERR_PARAM_COUNT;
                status_byte |= STAT_ERR;
                return;
            }
            // Process valid command
            break;
            
        default:
            error_code |= ERR_UNRECOGNIZED_COM;
            status_byte |= STAT_ERR;
            break;
    }
}
```

## Behavior Requirements

### Boot Behavior

✅ **Required**:
- Set STAT_BOOT flag after successful initialization
- Respond to STATUS_W_POS with valid position
- Initialize position to 0° or last known position

❌ **Not Required**:
- Physical homing routine (unless needed for your design)

### Rotation Behavior

✅ **Required**:
- Set STAT_TURN flag when rotation begins
- Clear STAT_TURN flag when target position is reached
- Report accurate position continuously during rotation
- Complete rotation or report ERR_ROT_TIME within 2 seconds per segment
- Achieve final position accuracy of ±1 degree
- Apply ramp distance for smooth deceleration
- Backlash compensation (physical or accounted for in reporting)

❌ **Not Required**:
- Specific acceleration profile (as long as smooth)
- Exact speed (as long as reasonable: 30-180°/sec typical)

### Status Reporting

✅ **Required**:
- STAT_BOOT: Set after initialization
- STAT_TURN: Set during rotation
- STAT_ERR: Set when errors occur

🔵 **Optional** (Master ignores):
- STAT_BACKLASH: Internal use
- STAT_HALTED: Internal use
- STAT_RESTING: Internal use

### Error Reporting

✅ **Must Report**:
- ERR_ROT_TIME: No position change for 2+ seconds during rotation
- ERR_BAD_COM: CRC validation failure
- ERR_PARAM_COUNT: Wrong number of command bytes
- ERR_UNRECOGNIZED_COM: Unknown command received

🔵 **Optional**:
- ERR_ROT_DIR: Rotation direction verification

## Timing Requirements

### Command Response Times

| Operation | Maximum Time | Notes |
|-----------|--------------|-------|
| STATUS_W_POS read | <1ms | Must respond immediately |
| STOP_ROT execution | <100ms | Should stop quickly |
| POSITION set | <1ms | No motor movement |
| RAMP_DIST set | <1ms | Parameter storage only |
| ERROR read | <1ms | Must respond immediately |

### Rotation Timing

| Parameter | Value | Notes |
|-----------|-------|-------|
| Polling interval | 100ms | THREE polls position every 100ms |
| Timeout threshold | 2000ms | Must show progress every 2 seconds |
| Maximum rotation time | ~10 seconds | For full 360° rotation |
| Position update rate | ≥10 Hz | During rotation |

### Recovery Behavior

When ERR_ROT_TIME occurs:
1. THREE reads ERROR (clears error state)
2. THREE adjusts RAMP_DIST (typically smaller value)
3. THREE resends ROTATE_ABS command
4. THREE will retry up to 3 times
5. If still failing, THREE throws exception

Your turntable should:
- Accept the new RAMP_DIST value
- Accept the repeated ROTATE_ABS command
- Attempt rotation again with new parameters

## Testing and Validation

### Basic Functionality Tests

**1. Boot Test**:
```
- Power on turntable
- Read STATUS_W_POS
- Verify STAT_BOOT (0x80) is set
- Verify position is in valid range 0-359
```

**2. Position Setting Test**:
```
- Write POSITION 0°
- Read STATUS_W_POS
- Verify position reads 0°
- No motor movement should occur
```

**3. Simple Rotation Test**:
```
- Write ROTATE_ABS 90°
- Poll STATUS_W_POS until complete
- Verify STAT_TURN set during rotation
- Verify STAT_TURN cleared when complete
- Verify final position = 90°
```

**4. Full Rotation Test**:
```
- Rotate to 0°, 90°, 180°, 270°, 0° in sequence
- Verify each position reached accurately
- Verify both CW and CCW rotation used
```

**5. Stop Test**:
```
- Send ROTATE_ABS 180°
- After 500ms, send STOP_ROT
- Verify rotation stops
- Verify STAT_TURN cleared
- Read final position
```

### Error Condition Tests

**6. CRC Error Test**:
```
- Send command with incorrect CRC
- Verify ERR_BAD_COM reported
- Verify command not executed
```

**7. Invalid Command Test**:
```
- Send unrecognized command (e.g., 0xFF)
- Verify ERR_UNRECOGNIZED_COM reported
```

**8. Timeout Test** (if applicable):
```
- Block motor physically
- Send ROTATE_ABS command
- Wait 2+ seconds
- Verify ERR_ROT_TIME reported
```

### Performance Tests

**9. Accuracy Test**:
```
- Rotate to 36°, 72°, 108°, ..., 324° (10° increments)
- Measure actual position (external measurement)
- Verify accuracy within ±1°
```

**10. Repeatability Test**:
```
- Rotate to 180° and back to 0°, 20 times
- Verify 0° position consistent within ±1°
```

**11. Speed Test**:
```
- Measure time for 360° rotation
- Should complete in 2-10 seconds (typical)
- Rotation should be smooth without jerking
```

## Best Practices

1. **Use Interrupts**: Handle encoder changes in interrupts for accurate position tracking
2. **Atomic Operations**: Protect shared variables accessed by I2C interrupts and main loop
3. **Validate Everything**: Check CRC, parameter counts, and value ranges
4. **Fail Gracefully**: Report errors clearly, don't hang or crash
5. **Test Thoroughly**: Test all commands, error conditions, and edge cases
6. **Document Your Hardware**: Provide clear connection diagrams and specs
7. **Make It Robust**: Handle power interruptions, noisy signals, and mechanical issues

## Support and Resources

### Useful Tools
- Logic analyzer for debugging I2C communication
- Oscilloscope for checking signal quality
- Protractor or angle measurement tool for position verification
- I2C scanner tool for verifying slave address

---

**Next**: See [Examples](Examples.md) for complete communication sequences and code samples.

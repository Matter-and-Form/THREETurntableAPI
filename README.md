# THREE Turntable I2C Communication API

This documentation describes the I2C communication protocol used by the Matter and Form THREE 3D scanner to control its turntable. This specification allows third-party developers to build compatible turntables that will work seamlessly with the THREE master device.

## Documentation Overview

This API documentation is organized into the following sections:

1. **[Protocol Specification](ProtocolSpecification.md)** - Complete I2C protocol details including command format, registers, status flags, and error codes

2. **[Integration Guide](IntegrationGuide.md)** - Practical guidance for implementing a compatible turntable including hardware requirements, behavior expectations, and timing constraints

3. **[Command Reference](CommandReference.md)** - Detailed reference for each command supported by the protocol

4. **[Examples](Examples.md)** - Common communication sequences and example implementations

## Quick Start

Your turntable must:
- Respond to I2C 7-bit slave address **0x45**
- Implement the required command set (STOP_ROT, STATUS_W_POS, POSITION, ROTATE_ABS, RAMP_DIST, ERROR)
- Use CRC-8 with polynomial 0x07 for data integrity
- Support 360-degree absolute positioning with 1-degree resolution
- Report status and position accurately

## Hardware Requirements

- I2C slave device operating at standard I2C speeds (100 kHz or 400 kHz)
- Rotary encoder or equivalent position sensing with 1-degree resolution
- Motor control capable of smooth rotation and positioning
- 360-degree continuous rotation capability

## Protocol Version

This documentation describes the protocol used by the THREE scanner as of 2024-10-20.

## Support

This is a third-party integration specification. The protocol is provided as-is for developers who wish to create compatible turntables for use with the THREE scanner.

---

**Copyright © 2024 Matter and Form. All rights reserved.**

# Swiftlet Counter - KiCad 9 Schematics

This repository contains KiCad 9 schematic files for the Swiftlet Counter project, implementing an IR-based counting system for swiftlet monitoring.

## Files

- `controller.kicad_sch` - Top-level schematic with ESP32 interface and control circuits
- `lane.kicad_sch` - Hierarchical sub-sheet for one counting lane (2 emitters + 2 receivers)

## ESP32 Signal Mapping

| GPIO | Function | Description |
|------|----------|-------------|
| GPIO33 | OE_595 | Output Enable for 74HC595 (PWM 38 kHz, active-low) |
| GPIO25 | 595_SER | Serial data input for 74HC595 chain |
| GPIO26 | 595_SRCLK | Shift register clock for 74HC595 |
| GPIO27 | 595_RCLK | Storage register clock for 74HC595 |
| GPIO21 | I2C_SDA | I2C data line for MCP23017 expanders |
| GPIO22 | I2C_SCL | I2C clock line for MCP23017 expanders |

## Current Implementation (Single Lane Demo)

The current schematic shows implementation for **Lane 0 only** for clarity and demonstration:

### Components Instantiated
- **1× ESP32 DevKitC** (represented as dual 19-pin connectors)
- **1× 74HC595** (8-bit shift register) - U1
- **1× ULN2803A** (Darlington driver array) - U2  
- **1× MCP23017** (I2C GPIO expander) - U3
- **1× Lane0** hierarchical sheet (2 emitters + 2 receivers)

### Power Supply Design
- **+5V rail**: Powers LED emitters through series resistors (47Ω)
- **+3.3V rail**: Powers all logic ICs (ESP32, 74HC595, MCP23017)
- **Bulk capacitance**: 470µF electrolytic + 1µF + 100nF ceramics on +5V
- **Decoupling**: 100nF ceramic per IC near power pins

## Scaling to 20 Lanes

To implement the full 20-lane system, duplicate and extend as follows:

### 74HC595 Chain (5× ICs for 20 lanes)
```
74HC595 #1: Q0,Q1 → Lane0; Q2,Q3 → Lane1; Q4,Q5 → Lane2; Q6,Q7 → Lane3
74HC595 #2: Q0,Q1 → Lane4; Q2,Q3 → Lane5; Q4,Q5 → Lane6; Q6,Q7 → Lane7
74HC595 #3: Q0,Q1 → Lane8; Q2,Q3 → Lane9; Q4,Q5 → Lane10; Q6,Q7 → Lane11
74HC595 #4: Q0,Q1 → Lane12; Q2,Q3 → Lane13; Q4,Q5 → Lane14; Q6,Q7 → Lane15
74HC595 #5: Q0,Q1 → Lane16; Q2,Q3 → Lane17; Q4,Q5 → Lane18; Q6,Q7 → Lane19
```

**Daisy-chain connection**: QH' output of each IC connects to SER input of next IC. All ICs share SRCLK, RCLK, and OE signals.

### ULN2803A Drivers (5× ICs)
Each ULN2803A maps its inputs 1:1 from corresponding 74HC595 outputs:
- ULN2803A #1: IN1-8 from 74HC595 #1 Q0-7
- ULN2803A #2: IN1-8 from 74HC595 #2 Q0-7
- etc.

Output mapping: OUT1 → Lane_X_LUAR_CATHODE, OUT2 → Lane_X_DALAM_CATHODE

### MCP23017 I2C Expanders (3× ICs)

| IC | I2C Address | Port A (GPA0-7) | Port B (GPB0-7) |
|----|-------------|-----------------|-----------------|
| MCP23017 #1 | 0x20 | Lanes 0-7 LUAR | Lanes 0-7 DALAM |
| MCP23017 #2 | 0x21 | Lanes 8-15 LUAR | Lanes 8-15 DALAM |
| MCP23017 #3 | 0x22 | Lanes 16-19 LUAR | Lanes 16-19 DALAM |

**Note**: MCP23017 #3 uses only GPA0-3 and GPB0-3 for the final 4 lanes.

### Lane Duplication in KiCad

1. **Copy the hierarchical sheet**: Select Lane0 sheet, copy (Ctrl+C), paste (Ctrl+V)
2. **Rename**: Change sheet name to Lane1, Lane2, etc.
3. **Update sheet pins**: Modify pin names from L0_* to L1_*, L2_*, etc.
4. **Reconnect nets**: Wire new sheet pins to appropriate:
   - 74HC595 Q outputs (via ULN2803A for cathodes)
   - MCP23017 GPIO pins (for TSOP outputs)
   - +5V rail (for emitter anodes via series resistors)
   - +3.3V and GND (for TSOP power)

**Note**: Hierarchical sheet pins for power rails (VCC/GND) must use type `input` or `passive`. KiCad 9 does not accept `power_in` as a valid pin type.

## Operation Notes

### LED Modulation
- **OE_595 (GPIO33)**: Generates 38 kHz PWM signal, **active-low**
- Only **2 LEDs active** at any time (one lane's LUAR+DALAM pair)
- Lane scanning prevents crosstalk between adjacent lanes

### TSOP Receivers  
- Powered at **+3.3V** with local 100nF decoupling per connector
- Enable **internal pull-ups** on MCP23017 (GPPU register) in firmware
- Output goes LOW when 38 kHz IR signal detected

### Power Considerations
- **+5V current**: ~40mA per active emitter pair (2×20mA)
- **+3.3V current**: Logic ICs (~50mA) + TSOP receivers (~5mA each)
- **Bulk capacitance**: Handles LED switching transients

## Firmware Configuration

### MCP23017 Setup
```c
// Enable pull-ups on all input pins
mcp23017_write_register(0x20, GPPU_A, 0xFF);  // Port A pull-ups
mcp23017_write_register(0x20, GPPU_B, 0xFF);  // Port B pull-ups
```

### Lane Scanning Sequence
1. Shift appropriate bit pattern to 74HC595 chain
2. Enable OE_595 with 38 kHz PWM for measurement period
3. Read corresponding MCP23017 pins for TSOP detection
4. Disable OE_595, advance to next lane
5. Repeat

This design provides a scalable, interference-free counting system suitable for swiftlet monitoring applications.
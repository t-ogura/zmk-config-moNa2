# Prospector Dongle Integration Guide

## Overview

This guide provides step-by-step instructions for integrating Prospector dongle functionality into any ZMK keyboard configuration. Based on the successful implementation for moNa2 keyboard, this guide will help you avoid common pitfalls and achieve a working dongle mode with ZMK Studio compatibility.

## Prerequisites

- Existing ZMK keyboard configuration repository
- Basic understanding of ZMK configuration structure
- Git repository access for your keyboard project

## Hardware Requirements

- **Microcontroller**: Seeed Studio XIAO nRF52840 (or compatible)
- **Display**: 1.69" Round ST7789V LCD (240x240 pixels)
- **Wiring Configuration**:
  ```
  LCD_DIN  -> Pin 10 (MOSI)
  LCD_CLK  -> Pin 8  (SCK)
  LCD_CS   -> Pin 9  (CS)
  LCD_DC   -> Pin 7  (Data/Command)
  LCD_RST  -> Pin 3  (Reset)
  LCD_BL   -> Pin 6  (Backlight PWM)
  ```

## Step-by-Step Integration

### Step 1: Add Prospector Module to West Manifest

**File**: `config/west.yml`

Add the prospector-zmk-module to your west.yml manifest:

```yaml
manifest:
  remotes:
    # Your existing remotes...
    - name: carrefinho                            # Add this remote
      url-base: https://github.com/carrefinho     # for prospector module

  projects:
    # Your existing projects...
    - name: prospector-zmk-module                 # Add this project
      remote: carrefinho
      revision: main

  self:
    path: config
```

**Example of complete addition**:
```diff
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
+   - name: carrefinho
+     url-base: https://github.com/carrefinho
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
+   - name: prospector-zmk-module
+     remote: carrefinho
+     revision: main
  self:
    path: config
```

### Step 2: Update Build Configuration

**File**: `build.yaml`

Add the dongle build configuration to your build matrix:

```yaml
include:
  # Your existing keyboard builds...
  - board: seeeduino_xiao_ble
    shield: [YOUR_KEYBOARD]_L rgbled_adapter
  - board: seeeduino_xiao_ble
    shield: [YOUR_KEYBOARD]_R rgbled_adapter
  
  # Add this dongle configuration
  - board: seeeduino_xiao_ble
    shield: [YOUR_KEYBOARD]_dongle prospector_adapter
    snippet: studio-rpc-usb-uart  # Required for ZMK Studio compatibility
```

### Step 3: Make Physical Layout ZMK Studio Compatible

**File**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD].dtsi`

**Critical Change**: Remove `zmk,matrix_transform` from the chosen block for ZMK Studio compatibility:

```diff
/ {
    chosen {
        zmk,kscan = &kscan0;
-       zmk,matrix_transform = &default_transform;  // Remove this line!
        zmk,physical-layout = &default_layout;
    };
```

**Add ZMK Studio Required Headers and Layout**:

```diff
+ #include <dt-bindings/zmk/matrix_transform.h>
+ #include <physical_layouts.dtsi>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,physical-layout = &default_layout;  // Keep this
    };

+   default_layout: default_layout {
+       compatible = "zmk,physical-layout";
+       display-name = "Default Layout";
+       transform = <&default_transform>;
+
+       keys  //                     w   h    x    y     rot    rx    ry
+           = <&key_physical_attrs 100 100    0   61       0     0     0>
+           , <&key_physical_attrs 100 100  100   28       0     0     0>
+           // ... Add all your key positions here
+           ;
+   };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <11>;  // Adjust to your keyboard
        rows = <4>;      // Adjust to your keyboard
        map = <
            // Your existing key mapping
        >;
    };
```

### Step 4: Add Dongle Kconfig Definitions

**File**: `config/boards/shields/[YOUR_KEYBOARD]/Kconfig.shield`

```kconfig
# Existing keyboard configurations...

config SHIELD_[YOUR_KEYBOARD]_DONGLE
    def_bool $(shields_list_contains,[YOUR_KEYBOARD]_dongle)
```

**File**: `config/boards/shields/[YOUR_KEYBOARD]/Kconfig.defconfig`

```kconfig
# Add dongle configuration block
if SHIELD_[YOUR_KEYBOARD]_DONGLE

config ZMK_KEYBOARD_NAME
    default "[YOUR_KEYBOARD]"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
    default 2

endif

# Your existing keyboard configurations...

if SHIELD_[YOUR_KEYBOARD]_DONGLE || SHIELD_[YOUR_KEYBOARD]_L || SHIELD_[YOUR_KEYBOARD]_R

config ZMK_SPLIT
    default y

endif
```

### Step 5: Create Dongle Device Tree Overlay

**File**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD]_dongle.overlay`

```dts
#include "[YOUR_KEYBOARD].dtsi"

/ {
    chosen {
        zmk,kscan = &mock_kscan;  // Override kscan only
    };

    mock_kscan: kscan_1 {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };
};

// Disable the original kscan if it has incomplete configuration
&kscan0 {
    status = "disabled";
};
```

### Step 6: Create Dongle Configuration File

**File**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD]_dongle.conf`

```conf
# Dongle Configuration based on forager implementation
# CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2
CONFIG_ZMK_STUDIO=y
```

### Step 7: Create Dongle Keymap

**File**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD]_dongle.keymap`

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            bindings = <&none>;  // Minimal keymap for dongle
        };
    };
};
```

### Step 8: Update .gitignore (Optional)

**File**: `.gitignore`

```gitignore
# Add build artifacts if not already present
*.uf2
build/
.vscode/
```

## Critical Success Factors

### 1. ZMK Studio Compatibility

**The most important change**: Remove `zmk,matrix_transform` from chosen block in your main `.dtsi` file. This is required for the ZMK Studio `USE_PHY_LAYOUTS` assertion to pass.

**Why this matters**: ZMK Studio requires physical layouts without chosen matrix transforms. The assertion `!IS_ENABLED(CONFIG_ZMK_STUDIO) || USE_PHY_LAYOUTS` will fail if `zmk,matrix_transform` is in the chosen block.

### 2. Physical Layout Definition

Ensure you have proper physical layout definitions with:
- `compatible = "zmk,physical-layout"`
- `keys` property with actual key positions
- `transform = <&default_transform>` reference

### 3. Build Configuration

- Use `studio-rpc-usb-uart` snippet for ZMK Studio compatibility
- Include both `[YOUR_KEYBOARD]_dongle` and `prospector_adapter` shields
- Target `seeeduino_xiao_ble` board

## Verification Steps

### 1. Build Success Verification

After making changes, push to GitHub and verify:
- GitHub Actions build completes successfully
- Artifact `[YOUR_KEYBOARD]_dongle-seeeduino_xiao_ble-zmk.uf2` is generated
- No ZMK Studio assertion errors in build log

### 2. Hardware Testing

1. Flash the generated `.uf2` file to your Seeed Studio XIAO nRF52840
2. Connect the 1.69" round LCD display according to wiring configuration
3. Power on the device
4. Verify display shows keyboard information (layer names, status, etc.)

### 3. Expected Results

- Display initializes and shows prospector interface
- Keyboard layer names appear on screen
- No build errors or assertion failures
- Clean GitHub Actions build process

## Troubleshooting Common Issues

### Issue 1: ZMK Studio Assertion Error

**Error**: `"ISSUE FOUND: Keyboards require additional configuration to allow for firmware with ZMK Studio enabled"`

**Solution**: Remove `zmk,matrix_transform = &default_transform;` from chosen block in your main `.dtsi` file.

### Issue 2: Devicetree Compilation Errors

**Error**: Various devicetree parsing errors

**Solution**: 
- Ensure all includes are present (`#include <physical_layouts.dtsi>`)
- Verify physical layout `keys` property is properly formatted
- Check that all references (`&default_transform`) are defined

### Issue 3: Missing Shield Errors

**Error**: Shield not found during build

**Solution**:
- Verify `Kconfig.shield` has proper shield definition
- Check `Kconfig.defconfig` has matching shield configuration
- Ensure all required files are created and properly named

### Issue 4: Build Artifacts Not Generated

**Error**: Build succeeds but no dongle firmware generated

**Solution**:
- Verify `build.yaml` has correct shield combination
- Check that `seeeduino_xiao_ble` board is specified
- Ensure `studio-rpc-usb-uart` snippet is included

## Example File Structure

After following this guide, your repository should have this structure:

```
your-zmk-config/
├── .gitignore
├── build.yaml                                    # Updated
├── config/
│   ├── west.yml                                 # Updated
│   └── boards/shields/[YOUR_KEYBOARD]/
│       ├── Kconfig.defconfig                    # Updated
│       ├── Kconfig.shield                       # Updated  
│       ├── [YOUR_KEYBOARD].dtsi                 # Updated (critical!)
│       ├── [YOUR_KEYBOARD]_dongle.conf          # New
│       ├── [YOUR_KEYBOARD]_dongle.keymap        # New
│       ├── [YOUR_KEYBOARD]_dongle.overlay       # New
│       └── ... (your existing keyboard files)
└── modules/                                     # New (git submodule)
    └── prospector-zmk-module/
```

## Success Criteria

You'll know the integration is successful when:

1. ✅ GitHub Actions build completes without errors
2. ✅ Dongle firmware (.uf2) is generated in artifacts
3. ✅ No ZMK Studio assertion errors in build log
4. ✅ Hardware displays keyboard information correctly
5. ✅ Layer names and status appear on the round LCD

## Reference Implementation

This guide is based on the successful moNa2 implementation:
- **Repository**: https://github.com/t-ogura/zmk-config-moNa2
- **Success Commit**: `9205d0a`
- **Reference**: https://github.com/carrefinho/forager-zmk-module-dongle

## Support and Additional Resources

- **ZMK Documentation**: https://zmk.dev/docs/features/studio
- **Prospector Module**: https://github.com/t-ogura/prospector-zmk-module
- **ZMK Studio Guide**: https://zmk.dev/docs/features/studio#adding-zmk-studio-support-to-a-keyboard

---

**Note**: This guide represents the distilled knowledge from months of debugging and testing. The key insight is that ZMK Studio compatibility requires removing `zmk,matrix_transform` from the chosen devicetree block - a single line change that resolves complex build assertion errors.
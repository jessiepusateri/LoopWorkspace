# How to Create Your Own Patches in the Patch Directory

This guide explains how to create custom patches directly in your local `patches` folder for automatic application during Loop builds.

## Overview

You can create your own `.patch` files in the `LoopWorkspace/patches/` directory to automatically apply custom code modifications during the build process.

## Creating a Patch File

### Method 1: Using Git Diff (Recommended)

1. **Make your changes** to the target files in the appropriate submodule
2. **Generate the patch** from the modified repository:
   ```bash
   cd Loop  # or LoopKit, or other submodule
   git diff > ../patches/your-patch-name.patch
   ```

3. **Edit the patch file** to include correct module paths:
   - Open `patches/your-patch-name.patch`
   - Add module name to file paths in the patch headers

### Method 2: Manual Patch Creation

Create a new `.patch` file with this structure:

```diff
--- a/ModuleName/path/to/file.swift
+++ b/ModuleName/path/to/file.swift
@@ -line_number,lines_changed +line_number,lines_changed @@
 context line
-old line to remove
+new line to add
 context line
```

## Patch File Structure

### Header Format
```diff
--- a/Loop/Loop/Managers/DeviceDataManager.swift
+++ b/Loop/Loop/Managers/DeviceDataManager.swift
```

### Hunk Header
```diff
@@ -123,7 +123,8 @@ class DeviceDataManager {
```
- `-123,7`: Starting at old line 123, showing 7 lines
- `+123,8`: Starting at new line 123, showing 8 lines

### Change Lines
- ` ` (space): Unchanged context line
- `-`: Line to be removed
- `+`: Line to be added

## Common Module Paths

- **Loop core**: `Loop/Loop/`
- **LoopKit**: `LoopKit/LoopKit/`
- **CGMBLEKit**: `CGMBLEKit/CGMBLEKit/`
- **RileyLinkBLEKit**: `RileyLinkBLEKit/RileyLinkBLEKit/`

## Step-by-Step Workflow

### 1. Identify Target File
```bash
# Find the file you want to modify using grep
grep -r "Settings" --include="*.swift" . | head -5
# Or search for specific text in files
grep '"Settings"' **/Loop/**/*.swift
```

### 2. Make Your Changes
Edit the target file directly using the Edit tool or text editor:
```bash
# Example: Change "Settings" to "Settings2" in SettingsView.swift
# Edit /path/to/Loop/Views/SettingsView.swift
```

### 3. Generate Patch from Submodule
```bash
# Navigate to the submodule directory
cd Loop

# Generate patch from the git diff
git diff Views/SettingsView.swift > ../patches/change_settings_to_settings2.patch
```

### 4. Fix Module Paths (Important!)
The generated patch will have incorrect paths. Edit the patch file:
```diff
# Original generated patch:
--- a/Views/SettingsView.swift
+++ b/Views/SettingsView.swift

# Keep it as-is for LoopWorkspace root application:
--- a/Loop/Views/SettingsView.swift  
+++ b/Loop/Views/SettingsView.swift
```

### 5. Test the Patch
```bash
# Reset the original file
git checkout Views/SettingsView.swift

# Apply patch from within the submodule directory
cd Loop
git apply /full/path/to/patches/change_settings_to_settings2.patch --allow-empty -v --whitespace=fix

# Verify the change was applied
grep -n "Settings2" Views/SettingsView.swift
```

## Example Patch Creation

Let's say you want to modify `Loop/Managers/DeviceDataManager.swift`:

1. **Navigate and edit**:
   ```bash
   cd Loop
   # Edit Loop/Managers/DeviceDataManager.swift
   ```

2. **Generate patch**:
   ```bash
   git diff Loop/Managers/DeviceDataManager.swift > ../patches/fix_device_data_manager.patch
   ```

3. **Edit patch file** to ensure correct paths:
   ```diff
   --- a/Loop/Loop/Managers/DeviceDataManager.swift
   +++ b/Loop/Loop/Managers/DeviceDataManager.swift
   @@ -45,7 +45,7 @@ class DeviceDataManager {
        private func updateData() {
   -        // old implementation
   +        // new implementation
        }
   ```

## Best Practices

- **Descriptive filenames**: Use clear names like `fix_insulin_calculation.patch`
- **Small, focused patches**: One logical change per patch file
- **Test thoroughly**: Always build and test after creating patches
- **Comment your changes**: Add comments in the code explaining modifications
- **Version control**: Consider backing up your patches

## Troubleshooting

**Patch won't apply:**
- Check file paths match exactly
- Verify line numbers and context
- Ensure base code hasn't changed since patch creation

**Build errors:**
- Review syntax in modified code
- Check for missing imports or dependencies
- Verify patch format is correct

## Building Locally with Patches

### Automatic Application
Patches in the `patches/` directory are automatically applied during builds using:
```bash
git apply ./patches/* --allow-empty -v --whitespace=fix
```

### Build Methods

**Method 1: Using Fastlane (Recommended)**
```bash
bundle install
bundle exec fastlane build_loop
```

**Method 2: Manual Application + Xcode**
```bash
# Apply patches manually first
git apply ./patches/* --allow-empty -v --whitespace=fix

# Then build through Xcode
open Loop.xcworkspace
```

**Method 3: GitHub Actions Build**
- Patches are applied automatically in the CI/CD workflow
- No additional steps needed

## Notes

- Patches are applied automatically during build
- No need to modify build scripts
- Multiple patches can be used simultaneously
- Patches are applied in alphabetical order by filename
- The build system handles patch application before compilation
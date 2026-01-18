# How to Add a Patch Using the Patch Folder

This guide explains how to add patches to your LoopWorkspace using the `patches` folder for Loop 3.4.0 and newer.

## Overview

The patches folder (`LoopWorkspace/patches/`) allows you to automatically apply custom modifications during the build process without manually editing source code files.

## Prerequisites

- Loop 3.4.0 or newer
- Access to your LoopWorkspace fork on GitHub
- Patch file from a trusted source

## Step-by-Step Process

### 1. Obtain the Patch File
- Open the patch URL in a browser
- Download the patch file to your local machine
- Note the original filename for reference

### 2. Prepare the Patch File
The most critical step is editing the patch file to include the correct module path:

- Open the downloaded patch file in a text editor
- Locate lines that start with `---` and `+++` (these define file paths)
- Add the module name to both "a" and "b" directory paths

**Example:**
```diff
# Original patch paths:
--- a/Loop/Managers/DeviceDataManager.swift
+++ b/Loop/Managers/DeviceDataManager.swift

# Should become (if targeting Loop module):
--- a/Loop/Loop/Managers/DeviceDataManager.swift
+++ b/Loop/Loop/Managers/DeviceDataManager.swift
```

### 3. Upload to GitHub
1. Navigate to your LoopWorkspace fork on GitHub
2. Click on the `patches` folder
3. Click "Add file" → "Upload files"
4. Drag and drop your edited patch file
5. Add a descriptive commit message
6. Commit directly to your default branch

### 4. Verify the Patch
- Test build your Loop app to ensure the patch applies correctly
- Check build logs for any patch-related errors
- Verify the intended functionality works as expected

## Important Notes

- **Path Accuracy**: Ensure patch file paths match the actual module structure
- **Automatic Application**: Patches are automatically applied during build - no need to modify `build_loop.yml`
- **File Naming**: Use descriptive filenames for your patches (e.g., `fix_insulin_delivery.patch`)
- **Testing**: Always test build after adding a patch

## Common Module Paths

- Loop core functionality: `Loop/`
- LoopKit framework: `LoopKit/`
- Third-party dependencies: Check the specific submodule name

## Troubleshooting

If your patch fails to apply:
1. Check that file paths in the patch match the actual repository structure
2. Verify the patch format is correct (unified diff format)
3. Ensure the base code matches what the patch expects
4. Check build logs for specific error messages

## Safety Reminders

- Only use patches from trusted sources
- Review patch contents before applying
- Keep backups of working builds
- Test thoroughly before using for actual diabetes management

## Example Patch Structure

```diff
--- a/Loop/Loop/Managers/DeviceDataManager.swift
+++ b/Loop/Loop/Managers/DeviceDataManager.swift
@@ -123,7 +123,7 @@ class DeviceDataManager {
     private func processInsulinData() {
-        // Original code
+        // Modified code
     }
 }
```
# How to Add App Intents to Loop

This guide explains how to add new App Intents to the Loop app, building on the existing Intent Extension framework.

## Overview

Loop already has an Intent Extension that supports override presets via Siri shortcuts. App Intents (iOS 16+) provide a more modern and flexible approach to creating shortcuts and automations.

## Existing Intent Infrastructure

Loop currently includes:
- **Intent Extension**: `Loop Intent Extension/` directory
- **Intent Definition**: `Common/Base.lproj/Intents.intentdefinition`
- **Intent Handlers**: 
  - `IntentHandler.swift` - Main handler router
  - `OverrideIntentHandler.swift` - Override preset handling
- **Extensions**: 
  - `NewCarbEntryIntent+Loop.swift`
  - `UserDefaults+LoopIntents.swift`

## Prerequisites

- iOS 16.0+ target for App Intents
- Xcode 14+
- Understanding of Loop's architecture and data managers

## Step-by-Step Implementation

### 1. Create App Intent File

Create a new Swift file in the Loop project (not the Intent Extension):

```swift
// Example: LoopAppIntents.swift
import AppIntents
import Foundation

@available(iOS 16.0, *)
struct GetCurrentGlucoseIntent: AppIntent {
    static var title: LocalizedStringResource = "Get Current Glucose"
    static var description = IntentDescription("Get the current glucose reading from Loop")
    static var isDiscoverable = true
    
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // Access Loop's glucose data
        guard let latestGlucose = await getLatestGlucoseValue() else {
            return .result(dialog: "No glucose data available")
        }
        
        return .result(dialog: "Current glucose is \(latestGlucose) mg/dL")
    }
    
    private func getLatestGlucoseValue() async -> Double? {
        // Implementation to get glucose from LoopDataManager
        // This would need to access the shared app group data
        return nil
    }
}
```

### 2. Create App Intent Entity (if needed)

For intents that work with specific objects:

```swift
@available(iOS 16.0, *)
struct OverridePreset: AppEntity {
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Override Preset")
    static var defaultQuery = OverridePresetQuery()
    
    var id: String
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(id)")
    }
}

@available(iOS 16.0, *)
struct OverridePresetQuery: EntityQuery {
    func entities(for identifiers: [String]) async throws -> [OverridePreset] {
        // Return specific presets by ID
        return []
    }
    
    func suggestedEntities() async throws -> [OverridePreset] {
        // Return all available override presets
        return []
    }
}
```

### 3. Register App Intents

Create an App Intents provider:

```swift
@available(iOS 16.0, *)
struct LoopAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: GetCurrentGlucoseIntent(),
            phrases: [
                "Get my glucose from Loop",
                "What's my current glucose?",
                "Check my blood sugar in Loop"
            ],
            shortTitle: "Current Glucose",
            systemImageName: "drop.fill"
        )
    }
}
```

### 4. Update Info.plist

Add App Intents support to the main app's `Info.plist`:

```xml
<key>NSSupportsLiveActivities</key>
<true/>
<key>NSUserActivityTypes</key>
<array>
    <string>GetCurrentGlucoseIntent</string>
    <!-- Add other intent names -->
</array>
```

### 5. Add to AppDelegate or App

In your main app file, ensure App Intents are initialized:

```swift
import AppIntents

class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Existing code...
        
        if #available(iOS 16.0, *) {
            // App Intents are automatically registered
        }
        
        return true
    }
}
```

### 6. Patch Creation Workflow

To add these changes via patches:

1. **Create the App Intent files**:
   ```bash
   # Create new Swift files in Loop/Loop/AppIntents/
   mkdir -p Loop/AppIntents
   # Add your App Intent Swift files
   ```

2. **Generate patches**:
   ```bash
   cd Loop
   git add .
   git diff --cached > ../patches/add_app_intents.patch
   ```

3. **Edit patch paths**:
   ```diff
   --- a/Loop/AppIntents/LoopAppIntents.swift
   +++ b/Loop/AppIntents/LoopAppIntents.swift
   ```

## Common App Intent Examples for Loop

### 1. Glucose Reading Intent
```swift
@available(iOS 16.0, *)
struct GetCurrentGlucoseIntent: AppIntent {
    static var title: LocalizedStringResource = "Get Current Glucose"
    
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // Access glucose data from shared UserDefaults
        let defaults = UserDefaults(suiteName: Bundle.main.appGroupSuiteName)
        // Implementation here
        return .result(dialog: "Glucose data retrieved")
    }
}
```

### 2. Override Preset Intent
```swift
@available(iOS 16.0, *)
struct EnableOverrideIntent: AppIntent {
    static var title: LocalizedStringResource = "Enable Override Preset"
    
    @Parameter(title: "Override Name")
    var overrideName: String
    
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // Implementation to enable override
        return .result(dialog: "Override \(overrideName) enabled")
    }
}
```

### 3. Carb Entry Intent
```swift
@available(iOS 16.0, *)
struct LogCarbsIntent: AppIntent {
    static var title: LocalizedStringResource = "Log Carbohydrates"
    
    @Parameter(title: "Carb Amount", description: "Amount of carbs in grams")
    var carbAmount: Double
    
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // Implementation to log carbs
        return .result(dialog: "\(carbAmount)g carbs logged")
    }
}
```

## Integration with Loop Data

### Accessing Loop Data Managers

App Intents run in the main app context, so you can access Loop's managers:

```swift
func perform() async throws -> some IntentResult {
    // Access through shared app group or singleton managers
    guard let loopManager = LoopDataManager.shared else {
        throw AppIntentError.dataUnavailable
    }
    
    // Use Loop's data managers
    let glucose = loopManager.glucoseStore.latestGlucose
    // Process and return result
}
```

### Shared Data Access

Use Loop's existing app group for data sharing:

```swift
let defaults = UserDefaults(suiteName: Bundle.main.appGroupSuiteName)
let intentInfo = defaults?.intentExtensionInfo
```

## Testing App Intents

1. **Build and install** the app with App Intents
2. **Open Shortcuts app** to see your intents
3. **Test via Siri**: "Hey Siri, get my glucose from Loop"
4. **Add to Lock Screen** widgets or Control Center

## How Other Apps Can Call Loop App Intents

### Using Shortcuts App
Other apps can trigger Loop intents through the Shortcuts app:

1. **Create a Shortcut**:
   - Open Shortcuts app
   - Tap "+" to create new shortcut
   - Search for "Loop" or your intent name
   - Add the Loop intent to the shortcut
   - Configure parameters if needed

2. **Call from Another App**:
   ```swift
   import Shortcuts
   
   // Trigger a Loop intent from another app
   func triggerLoopGlucoseCheck() {
       let intent = GetCurrentGlucoseIntent()
       
       Task {
           do {
               let result = try await intent.perform()
               print("Glucose result: \(result)")
           } catch {
               print("Failed to get glucose: \(error)")
           }
       }
   }
   ```

### URL Scheme Integration
Add URL schemes to allow external app integration and direct deep linking.

#### Setting Up URL Schemes in Loop

1. **Update Info.plist**:
   Add URL scheme support to Loop's `Info.plist`:
   ```xml
   <key>CFBundleURLTypes</key>
   <array>
       <dict>
           <key>CFBundleURLName</key>
           <string>com.loopkit.loop</string>
           <key>CFBundleURLSchemes</key>
           <array>
               <string>loop</string>
           </array>
       </dict>
   </array>
   ```

2. **Handle URL Schemes in AppDelegate**:
   ```swift
   // In AppDelegate.swift
   func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
       guard url.scheme == "loop" else { return false }
       
       return handleLoopURL(url)
   }
   
   private func handleLoopURL(_ url: URL) -> Bool {
       guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true),
             let host = components.host else {
           return false
       }
       
       switch host {
       case "glucose":
           return handleGlucoseRequest(components: components)
       case "override":
           return handleOverrideRequest(components: components)
       case "carbs":
           return handleCarbRequest(components: components)
       case "status":
           return handleStatusRequest(components: components)
       default:
           return false
       }
   }
   
   private func handleGlucoseRequest(components: URLComponents) -> Bool {
       // Get current glucose and return via callback URL if provided
       guard let callbackScheme = components.queryItems?.first(where: { $0.name == "callback" })?.value else {
           // Show glucose in app
           showCurrentGlucose()
           return true
       }
       
       // Return glucose data to callback URL
       Task {
           let glucose = await getCurrentGlucoseValue()
           let callbackURL = URL(string: "\(callbackScheme)://glucose?value=\(glucose ?? 0)")!
           await UIApplication.shared.open(callbackURL)
       }
       
       return true
   }
   
   private func handleOverrideRequest(components: URLComponents) -> Bool {
       guard let overrideName = components.queryItems?.first(where: { $0.name == "name" })?.value else {
           return false
       }
       
       // Enable the override preset
       enableOverridePreset(named: overrideName)
       
       // Call back if callback URL provided
       if let callbackScheme = components.queryItems?.first(where: { $0.name == "callback" })?.value {
           let callbackURL = URL(string: "\(callbackScheme)://override?result=success&name=\(overrideName)")!
           Task { await UIApplication.shared.open(callbackURL) }
       }
       
       return true
   }
   
   private func handleCarbRequest(components: URLComponents) -> Bool {
       guard let carbAmountString = components.queryItems?.first(where: { $0.name == "amount" })?.value,
             let carbAmount = Double(carbAmountString) else {
           return false
       }
       
       // Log carbs
       logCarbohydrates(amount: carbAmount)
       
       // Call back if callback URL provided
       if let callbackScheme = components.queryItems?.first(where: { $0.name == "callback" })?.value {
           let callbackURL = URL(string: "\(callbackScheme)://carbs?result=logged&amount=\(carbAmount)")!
           Task { await UIApplication.shared.open(callbackURL) }
       }
       
       return true
   }
   
   private func handleStatusRequest(components: URLComponents) -> Bool {
       // Get comprehensive Loop status
       Task {
           let status = await getLoopStatus()
           
           if let callbackScheme = components.queryItems?.first(where: { $0.name == "callback" })?.value {
               let statusData = "glucose=\(status.glucose)&iob=\(status.iob)&cob=\(status.cob)"
               let callbackURL = URL(string: "\(callbackScheme)://status?\(statusData)")!
               await UIApplication.shared.open(callbackURL)
           }
       }
       
       return true
   }
   ```

#### URL Scheme Examples

**Available Loop URL Schemes:**

1. **Get Current Glucose**:
   ```
   loop://glucose?callback=yourapp
   ```

2. **Enable Override Preset**:
   ```
   loop://override?name=Exercise&callback=yourapp
   ```

3. **Log Carbohydrates**:
   ```
   loop://carbs?amount=45&callback=yourapp
   ```

4. **Get Loop Status**:
   ```
   loop://status?callback=yourapp
   ```

#### How Other Apps Can Use Loop URL Schemes

1. **Setup Callback URL Scheme**:
   First, the calling app needs its own URL scheme in its `Info.plist`:
   ```xml
   <key>CFBundleURLTypes</key>
   <array>
       <dict>
           <key>CFBundleURLName</key>
           <string>com.yourapp.callback</string>
           <key>CFBundleURLSchemes</key>
           <array>
               <string>yourapp</string>
           </array>
       </dict>
   </array>
   ```

2. **Call Loop from Your App**:
   ```swift
   import UIKit
   
   class LoopIntegration {
       
       // Call Loop to get current glucose
       func requestCurrentGlucose() {
           guard let loopURL = URL(string: "loop://glucose?callback=yourapp") else { return }
           
           if UIApplication.shared.canOpenURL(loopURL) {
               UIApplication.shared.open(loopURL)
           } else {
               print("Loop app not installed")
           }
       }
       
       // Enable Loop override preset
       func enableLoopOverride(named presetName: String) {
           let encodedName = presetName.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? presetName
           guard let loopURL = URL(string: "loop://override?name=\(encodedName)&callback=yourapp") else { return }
           
           UIApplication.shared.open(loopURL)
       }
       
       // Log carbs in Loop
       func logCarbsInLoop(amount: Double) {
           guard let loopURL = URL(string: "loop://carbs?amount=\(amount)&callback=yourapp") else { return }
           
           UIApplication.shared.open(loopURL)
       }
       
       // Get comprehensive Loop status
       func requestLoopStatus() {
           guard let loopURL = URL(string: "loop://status?callback=yourapp") else { return }
           
           UIApplication.shared.open(loopURL)
       }
       
       // Check if Loop is installed
       func isLoopInstalled() -> Bool {
           guard let loopURL = URL(string: "loop://") else { return false }
           return UIApplication.shared.canOpenURL(loopURL)
       }
   }
   ```

3. **Handle Callbacks in Your App**:
   ```swift
   // In your app's AppDelegate or SceneDelegate
   func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
       guard url.scheme == "yourapp" else { return false }
       
       return handleLoopCallback(url)
   }
   
   private func handleLoopCallback(_ url: URL) -> Bool {
       guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true),
             let host = components.host else {
           return false
       }
       
       switch host {
       case "glucose":
           if let glucoseValue = components.queryItems?.first(where: { $0.name == "value" })?.value {
               handleGlucoseResponse(glucose: glucoseValue)
           }
           return true
           
       case "override":
           if let result = components.queryItems?.first(where: { $0.name == "result" })?.value,
              let name = components.queryItems?.first(where: { $0.name == "name" })?.value {
               handleOverrideResponse(result: result, overrideName: name)
           }
           return true
           
       case "carbs":
           if let result = components.queryItems?.first(where: { $0.name == "result" })?.value,
              let amount = components.queryItems?.first(where: { $0.name == "amount" })?.value {
               handleCarbResponse(result: result, amount: amount)
           }
           return true
           
       case "status":
           handleStatusResponse(components: components)
           return true
           
       default:
           return false
       }
   }
   
   private func handleGlucoseResponse(glucose: String) {
       print("Received glucose from Loop: \(glucose) mg/dL")
       // Update your app's UI with glucose data
   }
   
   private func handleOverrideResponse(result: String, overrideName: String) {
       print("Override \(overrideName) \(result)")
       // Update your app's state
   }
   
   private func handleCarbResponse(result: String, amount: String) {
       print("Carbs \(result): \(amount)g")
       // Update your app's meal logging
   }
   
   private func handleStatusResponse(components: URLComponents) {
       let glucose = components.queryItems?.first(where: { $0.name == "glucose" })?.value ?? "N/A"
       let iob = components.queryItems?.first(where: { $0.name == "iob" })?.value ?? "N/A"
       let cob = components.queryItems?.first(where: { $0.name == "cob" })?.value ?? "N/A"
       
       print("Loop Status - Glucose: \(glucose), IOB: \(iob), COB: \(cob)")
       // Update your app with comprehensive status
   }
   ```

#### Complete Integration Example

**Fitness App Integration:**
```swift
class FitnessLoopIntegration {
    let loopIntegration = LoopIntegration()
    
    func checkGlucoseBeforeWorkout() {
        guard loopIntegration.isLoopInstalled() else {
            showAlert("Loop app not found. Please install Loop to check glucose.")
            return
        }
        
        // Request glucose from Loop
        loopIntegration.requestCurrentGlucose()
        
        // Response will be handled in AppDelegate callback
        // which can then call adjustWorkoutBasedOnGlucose()
    }
    
    func enableExerciseMode() {
        loopIntegration.enableLoopOverride(named: "Exercise")
    }
    
    func logPreWorkoutCarbs(amount: Double) {
        loopIntegration.logCarbsInLoop(amount: amount)
    }
}
```

#### Security Considerations for URL Schemes

1. **Validate Input**: Always validate parameters from URL schemes
2. **Rate Limiting**: Implement rate limiting to prevent abuse
3. **Authentication**: Consider requiring user confirmation for sensitive actions
4. **Sanitization**: Properly encode/decode URL parameters

```swift
private func validateAndSanitizeInput(_ input: String) -> String? {
    // Remove potentially harmful characters
    let allowedCharacters = CharacterSet.alphanumerics.union(CharacterSet(charactersIn: ".-"))
    let filtered = input.components(separatedBy: allowedCharacters.inverted).joined()
    
    // Additional validation logic
    guard !filtered.isEmpty && filtered.count < 100 else {
        return nil
    }
    
    return filtered
}
```

### Cross-App Communication Examples

#### From Health Apps
```swift
// In a health tracking app
import AppIntents

func checkLoopGlucose() async {
    if #available(iOS 16.0, *) {
        let intent = GetCurrentGlucoseIntent()
        do {
            let result = try await intent.perform()
            // Use the glucose data in your health app
        } catch {
            print("Could not get Loop glucose data")
        }
    }
}
```

#### From Automation Apps (like Toolbox Pro)
```swift
// Create a shortcut that other automation apps can trigger
@available(iOS 16.0, *)
struct LoopStatusIntent: AppIntent {
    static var title: LocalizedStringResource = "Get Loop Status"
    
    func perform() async throws -> some IntentResult & ReturnsValue<LoopStatus> {
        let status = await getCurrentLoopStatus()
        return .result(value: status)
    }
}

struct LoopStatus: AppIntentValue {
    var glucose: Double
    var trend: String
    var lastUpdate: Date
    
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Loop Status")
}
```

#### Using Shortcuts Automation
1. **Time-based triggers**: Run Loop intents at specific times
2. **Location-based**: Check glucose when arriving at gym
3. **NFC triggers**: Tap NFC tag to get Loop status
4. **Widget interactions**: Tap widget to trigger intent

### API-Style Integration for Advanced Users

For developers who want to integrate with Loop data:

```swift
@available(iOS 16.0, *)
struct LoopDataAPIIntent: AppIntent {
    static var title: LocalizedStringResource = "Loop Data API"
    
    @Parameter(title: "Data Type")
    var dataType: LoopDataType
    
    @Parameter(title: "Time Range (hours)", default: 1)
    var timeRange: Double
    
    func perform() async throws -> some IntentResult & ReturnsValue<String> {
        switch dataType {
        case .glucose:
            let glucoseData = await getGlucoseData(hours: timeRange)
            return .result(value: formatGlucoseData(glucoseData))
        case .iob:
            let iobData = await getIOBData()
            return .result(value: formatIOBData(iobData))
        case .cob:
            let cobData = await getCOBData()
            return .result(value: formatCOBData(cobData))
        }
    }
}

enum LoopDataType: String, AppEnum {
    case glucose = "glucose"
    case iob = "iob"
    case cob = "cob"
    
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Loop Data Type")
    static var caseDisplayRepresentations: [Self: DisplayRepresentation] = [
        .glucose: "Glucose",
        .iob: "Insulin on Board",
        .cob: "Carbs on Board"
    ]
}
```

### Integration Examples by App Type

#### Smart Home Apps (HomeKit, etc.)
```swift
// Trigger in HomeKit automation
// "When I arrive home, check my Loop glucose"
// Can be set up through Shortcuts app automation
```

#### Fitness Apps
```swift
// Before workout routine
func preWorkoutCheck() {
    // Call Loop glucose intent
    // Adjust workout intensity based on glucose level
}
```

#### Meal Planning Apps
```swift
// Check current glucose before suggesting meal
func suggestMealBasedOnGlucose() async {
    let glucoseIntent = GetCurrentGlucoseIntent()
    let result = try? await glucoseIntent.perform()
    // Use glucose data to suggest appropriate meal
}
```

## Best Practices

1. **Keep intents lightweight** - avoid long-running operations
2. **Handle errors gracefully** - provide meaningful error messages
3. **Use appropriate privacy levels** - glucose data should be secure
4. **Test on device** - App Intents require device testing
5. **Follow Loop's existing patterns** - use similar data access methods

## Security Considerations

- App Intents can be triggered from Lock Screen
- Consider requiring authentication for sensitive actions
- Use appropriate intent authentication policies
- Be mindful of health data privacy requirements

## Troubleshooting

**Intent not appearing in Shortcuts:**
- Check iOS version compatibility (16.0+)
- Verify intent is properly registered
- Rebuild and reinstall the app

**Data access issues:**
- Ensure app group entitlements are correct
- Check shared UserDefaults configuration
- Verify Loop data managers are accessible

**Siri recognition problems:**
- Add more phrase variations
- Use clear, distinct command phrases
- Test with different voice patterns
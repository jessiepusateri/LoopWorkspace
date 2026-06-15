# How to Create a Carb Entry App Intent for Loop

This guide provides complete step-by-step instructions for adding a carbohydrate entry App Intent to Loop, including patch creation and external app integration.

## Overview

This App Intent will allow users and external apps to log carbohydrates directly into Loop via Siri shortcuts, Shortcuts app automations, or direct app-to-app communication.

## Implementation Steps

### 1. Create the App Intent Files

#### Create AppIntents Directory Structure
```bash
mkdir -p Loop/AppIntents
```

#### Create the Main Carb Entry Intent

**File: `Loop/AppIntents/CarbEntryAppIntent.swift`**
```swift
//
//  CarbEntryAppIntent.swift
//  Loop
//
//  Created by App Intent Implementation
//  Copyright © 2024 LoopKit Authors. All rights reserved.
//

import AppIntents
import Foundation
import LoopKit
import LoopCore

@available(iOS 16.0, *)
struct LogCarbsIntent: AppIntent {
    static var title: LocalizedStringResource = "Log Carbohydrates in Loop"
    static var description = IntentDescription("Log carbohydrate intake for insulin dosing calculations")
    static var isDiscoverable = true
    static var openAppWhenRun: Bool = false
    
    @Parameter(
        title: "Carb Amount",
        description: "Amount of carbohydrates in grams",
        default: 0,
        inclusiveRange: (0, 500)
    )
    var carbAmount: Double
    
    @Parameter(
        title: "Absorption Time",
        description: "How long the carbs will take to absorb (in hours)",
        default: 3.0,
        inclusiveRange: (1.0, 8.0)
    )
    var absorptionTime: Double?
    
    @Parameter(
        title: "Food Type",
        description: "Type of food being consumed"
    )
    var foodType: String?
    
    func perform() async throws -> some IntentResult & ProvidesDialog & ShowsSnippetView {
        guard carbAmount > 0 else {
            return .result(
                dialog: "Please enter a carb amount greater than 0 grams",
                view: CarbEntryResultView(result: .failure("Invalid amount"))
            )
        }
        
        do {
            let success = try await logCarbohydrates(
                amount: carbAmount,
                absorptionTime: absorptionTime ?? 3.0,
                foodType: foodType
            )
            
            if success {
                let message = formatSuccessMessage()
                return .result(
                    dialog: message,
                    view: CarbEntryResultView(result: .success(carbAmount, absorptionTime ?? 3.0))
                )
            } else {
                return .result(
                    dialog: "Failed to log carbohydrates. Please try again.",
                    view: CarbEntryResultView(result: .failure("Logging failed"))
                )
            }
        } catch {
            return .result(
                dialog: "Error logging carbohydrates: \(error.localizedDescription)",
                view: CarbEntryResultView(result: .failure(error.localizedDescription))
            )
        }
    }
    
    private func formatSuccessMessage() -> String {
        var message = "Logged \(Int(carbAmount))g carbs"
        if let foodType = foodType, !foodType.isEmpty {
            message += " from \(foodType)"
        }
        if let absorption = absorptionTime, absorption != 3.0 {
            message += " with \(String(format: "%.1f", absorption))h absorption"
        }
        return message
    }
    
    private func logCarbohydrates(amount: Double, absorptionTime: Double, foodType: String?) async throws -> Bool {
        return await withCheckedContinuation { continuation in
            DispatchQueue.main.async {
                guard let appDelegate = UIApplication.shared.delegate as? AppDelegate,
                      let loopManager = appDelegate.loopManager else {
                    continuation.resume(returning: false)
                    return
                }
                
                let now = Date()
                let absorptionTimeInterval = TimeInterval(absorptionTime * 60 * 60) // Convert to seconds
                
                let carbEntry = NewCarbEntry(
                    quantity: HKQuantity(unit: .gram(), doubleValue: amount),
                    startDate: now,
                    foodType: foodType?.isEmpty == false ? foodType : nil,
                    absorptionTime: absorptionTimeInterval
                )
                
                loopManager.addCarbEntry(carbEntry) { result in
                    switch result {
                    case .success:
                        continuation.resume(returning: true)
                    case .failure:
                        continuation.resume(returning: false)
                    }
                }
            }
        }
    }
}
```

#### Create the Result View

**File: `Loop/AppIntents/CarbEntryResultView.swift`**
```swift
//
//  CarbEntryResultView.swift
//  Loop
//
//  Created by App Intent Implementation
//  Copyright © 2024 LoopKit Authors. All rights reserved.
//

import AppIntents
import SwiftUI

@available(iOS 16.0, *)
struct CarbEntryResultView: View {
    let result: CarbEntryResult
    
    var body: some View {
        VStack(spacing: 16) {
            switch result {
            case .success(let amount, let absorptionTime):
                Image(systemName: "checkmark.circle.fill")
                    .font(.system(size: 50))
                    .foregroundColor(.green)
                
                Text("Carbs Logged Successfully")
                    .font(.headline)
                
                VStack(alignment: .leading, spacing: 8) {
                    HStack {
                        Text("Amount:")
                            .fontWeight(.medium)
                        Spacer()
                        Text("\(Int(amount))g")
                    }
                    
                    HStack {
                        Text("Absorption Time:")
                            .fontWeight(.medium)
                        Spacer()
                        Text(String(format: "%.1f hours", absorptionTime))
                    }
                }
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
                
            case .failure(let error):
                Image(systemName: "exclamationmark.triangle.fill")
                    .font(.system(size: 50))
                    .foregroundColor(.orange)
                
                Text("Failed to Log Carbs")
                    .font(.headline)
                
                Text(error)
                    .font(.body)
                    .foregroundColor(.secondary)
                    .multilineTextAlignment(.center)
            }
        }
        .padding()
    }
}

enum CarbEntryResult {
    case success(Double, Double) // amount, absorption time
    case failure(String)
}
```

#### Create App Shortcuts Provider

**File: `Loop/AppIntents/LoopAppShortcuts.swift`**
```swift
//
//  LoopAppShortcuts.swift
//  Loop
//
//  Created by App Intent Implementation
//  Copyright © 2024 LoopKit Authors. All rights reserved.
//

import AppIntents

@available(iOS 16.0, *)
struct LoopAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: LogCarbsIntent(),
            phrases: [
                "Log carbs in Loop",
                "Add carbohydrates to Loop",
                "Enter carbs in Loop",
                "Log \(.applicationName) carbohydrates",
                "Add \(\.$carbAmount) grams of carbs to Loop",
                "I ate \(\.$carbAmount) grams of carbs"
            ],
            shortTitle: "Log Carbs",
            systemImageName: "fork.knife"
        )
    }
}
```

### 2. Update Existing Files

#### Update AppDelegate.swift

Add App Intent initialization and URL scheme handling:

```swift
// Add to imports
import AppIntents

// Add to applicationDidFinishLaunching
if #available(iOS 16.0, *) {
    // App Intents are automatically registered when app launches
}

// Add URL scheme handling method (if not already present)
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
    case "carbs":
        return handleCarbRequest(components: components)
    // Add other cases as needed
    default:
        return false
    }
}

private func handleCarbRequest(components: URLComponents) -> Bool {
    guard let carbAmountString = components.queryItems?.first(where: { $0.name == "amount" })?.value,
          let carbAmount = Double(carbAmountString) else {
        return false
    }
    
    let absorptionTime = components.queryItems?.first(where: { $0.name == "absorption" })?.value
        .flatMap { Double($0) } ?? 3.0
    let foodType = components.queryItems?.first(where: { $0.name == "food" })?.value
    
    // Create carb entry
    let now = Date()
    let absorptionTimeInterval = TimeInterval(absorptionTime * 60 * 60)
    
    let carbEntry = NewCarbEntry(
        quantity: HKQuantity(unit: .gram(), doubleValue: carbAmount),
        startDate: now,
        foodType: foodType?.isEmpty == false ? foodType : nil,
        absorptionTime: absorptionTimeInterval
    )
    
    loopManager?.addCarbEntry(carbEntry) { result in
        // Handle result
        if let callbackScheme = components.queryItems?.first(where: { $0.name == "callback" })?.value {
            let resultString = result.isSuccess ? "success" : "failure"
            let callbackURL = URL(string: "\(callbackScheme)://carbs?result=\(resultString)&amount=\(carbAmount)")!
            Task { await UIApplication.shared.open(callbackURL) }
        }
    }
    
    return true
}
```

#### Update Info.plist

Add App Intent support and URL schemes:

```xml
<!-- Add to Info.plist -->
<key>NSSupportsLiveActivities</key>
<true/>
<key>NSUserActivityTypes</key>
<array>
    <string>LogCarbsIntent</string>
</array>
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

### 3. Create Patches

#### Generate the Implementation Patch

1. **Add all new files to git**:
   ```bash
   cd Loop
   git add AppIntents/
   git add -u  # Add modifications to existing files
   ```

2. **Create the patch**:
   ```bash
   git diff --cached > ../patches/add_carb_entry_app_intent.patch
   ```

3. **Edit patch file paths**:
   Edit `patches/add_carb_entry_app_intent.patch` to ensure correct module paths:
   ```diff
   --- a/Loop/AppIntents/CarbEntryAppIntent.swift
   +++ b/Loop/AppIntents/CarbEntryAppIntent.swift
   ```

#### Example Patch File Structure

The patch will include:
- New files: `AppIntents/CarbEntryAppIntent.swift`
- New files: `AppIntents/CarbEntryResultView.swift`
- New files: `AppIntents/LoopAppShortcuts.swift`
- Modified: `AppDelegate.swift` (URL handling)
- Modified: `Info.plist` (App Intent support)

### 4. How External Apps Call the Carb Entry Intent

#### Via App Intents (iOS 16+)

**In the calling app:**
```swift
import AppIntents

@available(iOS 16.0, *)
func logCarbsInLoop(amount: Double, absorptionTime: Double? = nil, foodType: String? = nil) async {
    let intent = LogCarbsIntent()
    intent.carbAmount = amount
    intent.absorptionTime = absorptionTime
    intent.foodType = foodType
    
    do {
        let result = try await intent.perform()
        print("Carbs logged successfully")
    } catch {
        print("Failed to log carbs: \(error)")
    }
}
```

#### Via URL Schemes

**Setup in calling app's Info.plist:**
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

**Call Loop via URL scheme:**
```swift
class LoopCarbIntegration {
    func logCarbs(amount: Double, absorptionTime: Double? = nil, foodType: String? = nil) {
        var urlString = "loop://carbs?amount=\(amount)&callback=yourapp"
        
        if let absorption = absorptionTime {
            urlString += "&absorption=\(absorption)"
        }
        
        if let food = foodType?.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) {
            urlString += "&food=\(food)"
        }
        
        guard let url = URL(string: urlString) else { return }
        
        if UIApplication.shared.canOpenURL(url) {
            UIApplication.shared.open(url)
        } else {
            print("Loop app not installed")
        }
    }
    
    func isLoopInstalled() -> Bool {
        guard let url = URL(string: "loop://") else { return false }
        return UIApplication.shared.canOpenURL(url)
    }
}
```

**Handle callback in calling app's AppDelegate:**
```swift
func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
    guard url.scheme == "yourapp" else { return false }
    
    if url.host == "carbs" {
        handleCarbLoggingResult(url: url)
        return true
    }
    
    return false
}

private func handleCarbLoggingResult(url: URL) {
    guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true) else { return }
    
    let result = components.queryItems?.first(where: { $0.name == "result" })?.value ?? "unknown"
    let amount = components.queryItems?.first(where: { $0.name == "amount" })?.value ?? "0"
    
    if result == "success" {
        print("Successfully logged \(amount)g carbs in Loop")
        // Update your app's UI
    } else {
        print("Failed to log carbs in Loop")
        // Show error message
    }
}
```

### 5. Complete Integration Examples

#### Meal Planning App Integration

```swift
class MealPlannerLoopIntegration {
    private let loopIntegration = LoopCarbIntegration()
    
    func logMealCarbs(meal: Meal) {
        guard loopIntegration.isLoopInstalled() else {
            showLoopNotInstalledAlert()
            return
        }
        
        loopIntegration.logCarbs(
            amount: meal.totalCarbs,
            absorptionTime: meal.estimatedAbsorptionTime,
            foodType: meal.name
        )
    }
    
    @available(iOS 16.0, *)
    func logMealCarbsWithAppIntent(meal: Meal) async {
        let intent = LogCarbsIntent()
        intent.carbAmount = meal.totalCarbs
        intent.absorptionTime = meal.estimatedAbsorptionTime
        intent.foodType = meal.name
        
        do {
            _ = try await intent.perform()
            updateMealAsLogged(meal)
        } catch {
            showErrorAlert("Failed to log carbs: \(error.localizedDescription)")
        }
    }
}
```

#### Restaurant App Integration

```swift
class RestaurantAppLoopIntegration {
    func logDishCarbs(dish: Dish) {
        let carbAmount = dish.nutritionalInfo.carbohydrates
        let absorptionTime = dish.carbAbsorptionEstimate ?? 3.0
        
        // Using URL scheme
        let integration = LoopCarbIntegration()
        integration.logCarbs(
            amount: carbAmount,
            absorptionTime: absorptionTime,
            foodType: dish.name
        )
    }
    
    func addLoopIntegrationToMenu() {
        // Add "Log to Loop" buttons next to dishes
        // Show carb amounts and Loop integration status
    }
}
```

#### Shortcuts App Automation Examples

Users can create automations like:
1. **NFC Tag**: Tap phone to NFC tag at dining table → Ask for carb amount → Log to Loop
2. **Time-based**: Every day at meal times → Remind to log carbs
3. **Location-based**: Arrive at restaurant → Open carb logging shortcut
4. **Voice**: "Hey Siri, I ate 45 grams of carbs" → Automatically logs to Loop

### 6. Testing the Implementation

#### Test via Shortcuts App
1. Build and install Loop with the patch
2. Open Shortcuts app
3. Look for "Log Carbs" in Loop section
4. Create test shortcut and run it

#### Test via Siri
- "Hey Siri, log carbs in Loop"
- "Hey Siri, I ate 45 grams of carbs"
- "Hey Siri, add carbohydrates to Loop"

#### Test via URL Scheme
```swift
// Test URL in app or Safari
loop://carbs?amount=30&absorption=2.5&food=Pasta&callback=yourapp
```

### 7. Troubleshooting

**Intent not appearing:**
- Check iOS 16+ requirement
- Verify App Intent files are added to project
- Rebuild and reinstall app

**URL scheme not working:**
- Verify Info.plist URL scheme configuration
- Check URL encoding for special characters
- Ensure callback URL scheme is registered

**Carb logging failing:**
- Verify LoopManager is accessible
- Check carb amount validation (0-500g range)
- Ensure proper error handling in callback

### 8. Security Considerations

- **Input Validation**: Validate carb amounts (reasonable range)
- **Rate Limiting**: Prevent spam logging attempts  
- **User Confirmation**: Consider requiring confirmation for large amounts
- **Health Data Privacy**: Ensure proper permissions and data handling

This implementation provides a complete carb entry App Intent that integrates seamlessly with Loop's existing architecture while providing multiple ways for external apps to interact with it.
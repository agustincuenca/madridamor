# Skill: Hotwire Native

## Purpose
Build native iOS and Android apps using the existing Rails views with Hotwire Native (formerly Turbo Native).

## Overview

Hotwire Native wraps your Rails web app in a native shell, providing:
- Native navigation (push/pop)
- Native UI elements (tab bars, toolbars)
- Bridge components for native features
- Offline support
- Push notifications

## iOS Setup

### Project Structure
```
ios/
├── App/
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   └── Configuration.swift
├── Navigation/
│   ├── Navigator.swift
│   └── PathConfiguration.swift
└── Bridge/
    └── Components/
```

### Basic Configuration
```swift
// Configuration.swift
import HotwireNative

struct Configuration {
    static let baseURL = URL(string: "https://yourapp.com")!

    static var pathConfiguration: PathConfiguration {
        PathConfiguration(sources: [
            .server(baseURL.appendingPathComponent("/configurations/ios_v1.json"))
        ])
    }
}
```

### Scene Delegate
```swift
// SceneDelegate.swift
import UIKit
import HotwireNative

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private let navigator = Navigator()

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigator.rootViewController
        window?.makeKeyAndVisible()

        navigator.route(Configuration.baseURL)
    }
}
```

## Android Setup

### Project Structure
```
android/
├── app/src/main/
│   ├── java/com/yourapp/
│   │   ├── MainActivity.kt
│   │   ├── MainSessionNavHostFragment.kt
│   │   └── bridge/
│   └── res/
│       └── navigation/
│           └── main.xml
```

### Main Activity
```kotlin
// MainActivity.kt
import android.os.Bundle
import dev.hotwire.navigation.activities.HotwireActivity
import dev.hotwire.navigation.navigator.NavigatorConfiguration

class MainActivity : HotwireActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }

    override fun navigatorConfigurations() = listOf(
        NavigatorConfiguration(
            name = "main",
            startLocation = "https://yourapp.com",
            navigatorHostId = R.id.main_nav_host
        )
    )
}
```

## Path Configuration

### Server-Side JSON
```json
// public/configurations/ios_v1.json
{
  "settings": {
    "screenshots_enabled": true
  },
  "rules": [
    {
      "patterns": ["/"],
      "properties": {
        "presentation": "default"
      }
    },
    {
      "patterns": ["/new$", "/edit$"],
      "properties": {
        "presentation": "modal"
      }
    },
    {
      "patterns": ["/articles/\\d+"],
      "properties": {
        "presentation": "push"
      }
    }
  ]
}
```

### Presentation Types
- `default` - Standard push navigation
- `modal` - Present as modal sheet
- `replace` - Replace current screen
- `pop` - Pop back
- `refresh` - Refresh current
- `none` - Handle in native code

## Bridge Components

### Rails Helper
```ruby
# app/helpers/hotwire_native_helper.rb
module HotwireNativeHelper
  def hotwire_native_app?
    request.user_agent&.include?("Hotwire Native")
  end

  def hotwire_native_ios?
    request.user_agent&.include?("Hotwire Native iOS")
  end

  def hotwire_native_android?
    request.user_agent&.include?("Hotwire Native Android")
  end
end
```

### Conditional Rendering
```erb
<% if hotwire_native_app? %>
  <%# Native-specific UI %>
  <div data-bridge-component="navbar" data-bridge-title="<%= @article.title %>"></div>
<% else %>
  <%# Web navigation %>
  <%= render "shared/navigation" %>
<% end %>
```

### Bridge Component (JavaScript)
```javascript
// app/javascript/controllers/bridge/button_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "button"
  static targets = ["button"]

  connect() {
    super.connect()
    this.send("connect", { title: this.buttonTarget.textContent })
  }

  // Called from native
  clicked() {
    this.buttonTarget.click()
  }
}
```

### iOS Bridge Handler
```swift
// ButtonComponent.swift
import HotwireNative

final class ButtonComponent: BridgeComponent {
    override class var name: String { "button" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .connect:
            handleConnect(message: message)
        }
    }

    private func handleConnect(message: Message) {
        guard let data: ConnectData = message.data() else { return }
        // Add native button to toolbar
        addButton(title: data.title)
    }

    private func addButton(title: String) {
        let button = UIBarButtonItem(title: title, style: .plain, target: self, action: #selector(buttonTapped))
        delegate?.webView?.viewController?.navigationItem.rightBarButtonItem = button
    }

    @objc private func buttonTapped() {
        reply(to: "connect", with: MessageData(action: "clicked"))
    }
}

private extension ButtonComponent {
    enum Event: String {
        case connect
    }

    struct ConnectData: Decodable {
        let title: String
    }
}
```

## Native Features

### Pull to Refresh
```erb
<%# Enable in layout %>
<body data-turbo-refresh-method="morph" data-turbo-refresh-scroll="preserve">
```

### Native Form Inputs
```erb
<% if hotwire_native_app? %>
  <%= f.date_field :date, data: { bridge_component: "date-picker" } %>
<% else %>
  <%= f.date_field :date %>
<% end %>
```

### Camera/Photos
```erb
<div data-controller="bridge--photo" data-bridge-component="photo">
  <button data-action="bridge--photo#capture">Take Photo</button>
  <input type="hidden" name="photo_data" data-bridge--photo-target="input">
</div>
```

## Styling for Native

```erb
<%# app/views/layouts/application.html.erb %>
<html class="<%= 'hotwire-native' if hotwire_native_app? %>">

<%# In CSS %>
<style>
  /* Hide web-only elements in native */
  .hotwire-native .web-only {
    display: none;
  }

  /* Safe area insets */
  .hotwire-native body {
    padding-top: env(safe-area-inset-top);
    padding-bottom: env(safe-area-inset-bottom);
  }
</style>
```

## Best Practices

1. **Mobile-first design** - Views should work well on small screens
2. **Use semantic HTML** - Native can style based on elements
3. **Minimize JavaScript** - Let native handle interactions
4. **Test on devices** - Simulator ≠ real device
5. **Handle offline** - Show appropriate messages
6. **Fast responses** - Native users expect instant feedback

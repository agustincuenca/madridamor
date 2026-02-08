# Comando: /movil

Prepara y genera la versión móvil de la aplicación usando Hotwire Native.

## Flujo de trabajo

### Paso 1: Verificar requisitos

1. Confirmar que la app web funciona correctamente
2. Verificar que las vistas son responsive
3. Comprobar que Hotwire está configurado

### Paso 2: Preguntar plataformas

"¿Para qué plataformas quieres generar la app?"
- [ ] iOS
- [ ] Android
- [ ] Ambas

### Paso 3: Crear configuración de path

Crear archivo de configuración para la navegación nativa:

```bash
mkdir -p public/configurations
```

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
      "patterns": ["/sign_in", "/sign_up"],
      "properties": {
        "presentation": "replace"
      }
    }
  ]
}
```

### Paso 4: Crear helper para detectar app nativa

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

### Paso 5: Adaptar layout para móvil

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html lang="<%= I18n.locale %>" class="<%= 'hotwire-native' if hotwire_native_app? %>">
<head>
  <!-- ... -->
</head>
<body class="<%= 'native-app' if hotwire_native_app? %>">
  <% unless hotwire_native_app? %>
    <%= render "shared/navigation" %>
  <% end %>

  <main class="<%= hotwire_native_app? ? 'pt-safe-top pb-safe-bottom' : '' %>">
    <%= yield %>
  </main>

  <% unless hotwire_native_app? %>
    <%= render "shared/footer" %>
  <% end %>
</body>
</html>
```

### Paso 6: Generar proyecto iOS (si seleccionado)

Crear instrucciones para el proyecto iOS:

```markdown
## Configuración iOS

### Requisitos
- macOS
- Xcode 15+
- CocoaPods o Swift Package Manager

### Pasos

1. Crear nuevo proyecto Xcode (iOS App)

2. Añadir Hotwire Native via SPM:
   - URL: https://github.com/hotwired/hotwire-native-ios

3. Configurar SceneDelegate:
```swift
import HotwireNative

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private let navigator = Navigator()

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }

        Hotwire.config.pathConfiguration.sources = [
            .server(URL(string: "https://tu-app.com/configurations/ios_v1.json")!)
        ]

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigator.rootViewController
        window?.makeKeyAndVisible()

        navigator.route(URL(string: "https://tu-app.com")!)
    }
}
```

4. Configurar permisos en Info.plist según necesidades
```

### Paso 7: Generar proyecto Android (si seleccionado)

```markdown
## Configuración Android

### Requisitos
- Android Studio Hedgehog o superior
- JDK 17+

### Pasos

1. Crear nuevo proyecto Android (Empty Activity)

2. Añadir dependencia en build.gradle:
```kotlin
dependencies {
    implementation("dev.hotwire:navigation:1.0.0")
}
```

3. Configurar MainActivity:
```kotlin
class MainActivity : HotwireActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }

    override fun navigatorConfigurations() = listOf(
        NavigatorConfiguration(
            name = "main",
            startLocation = "https://tu-app.com",
            navigatorHostId = R.id.main_nav_host
        )
    )
}
```

4. Configurar permisos en AndroidManifest.xml
```

### Paso 8: Verificar funcionalidad

Checklist de verificación:
- [ ] Navegación funciona correctamente
- [ ] Formularios envían datos
- [ ] Turbo Frames actualizan sin problemas
- [ ] Modales se abren correctamente
- [ ] Back button funciona
- [ ] Pull to refresh funciona
- [ ] Links externos abren en navegador

### Paso 9: Generar resumen

```markdown
# App Móvil Generada

## Configuración completada
- ✅ Path configuration creado
- ✅ Helper de detección añadido
- ✅ Layout adaptado para nativo
- ✅ Instrucciones iOS generadas
- ✅ Instrucciones Android generadas

## Archivos creados/modificados
- `public/configurations/ios_v1.json`
- `public/configurations/android_v1.json`
- `app/helpers/hotwire_native_helper.rb`
- `app/views/layouts/application.html.erb`

## Próximos pasos

### Para iOS:
1. Sigue las instrucciones en `docs/ios_setup.md`
2. Configura tu URL de producción
3. Añade iconos y splash screen
4. Prueba en simulador y dispositivo real
5. Publica en App Store

### Para Android:
1. Sigue las instrucciones en `docs/android_setup.md`
2. Configura tu URL de producción
3. Añade iconos y splash screen
4. Prueba en emulador y dispositivo real
5. Publica en Play Store

## Recursos útiles
- [Hotwire Native iOS](https://github.com/hotwired/hotwire-native-ios)
- [Hotwire Native Android](https://github.com/hotwired/hotwire-native-android)
- [Documentación oficial](https://hotwired.dev)
```

## Notas importantes

- La app móvil usa las mismas vistas que la web
- Solo se necesita código nativo mínimo para el "shell"
- Los cambios en la web se reflejan automáticamente en móvil
- Para funcionalidades nativas específicas (cámara, GPS, etc.) se necesitan bridge components
- Probar siempre en dispositivos reales antes de publicar

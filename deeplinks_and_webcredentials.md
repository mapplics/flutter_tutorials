# Guía de implementación de DeepLinks y WebCredentials en Flutter

Es recomendable leer todo una vez antes de empezar. 

**DeepLinks** son URIs que al clickearlos abren nuestra app y nos llevan a una parte especifica de ella. 
Google los llama Deeplinks mientras que Apple prefiere llamarlos Universal Links, la lógica detrás de ambos es la misma.

**WebCredentials** permite vincular las credenciales de nuestra app con las de una plataforma web así el gestor de contraseñas que estemos usando nos facilita el inicio de sesión.

---

## Implementar DeepLinks

Se puede dividir en 3 pasos, primero tenemos que hostear un archivo de configuración en el dominio de las URIs que queremos manejar, despues tenemos que vincular dicho dominio a nuestra app y por último implementar el código necesario para reaccionar cuando la app recibe un link.

### 1 - Hostear el archivo de configuración

#### Android
Lo primero que necesitamos hacer es hostear una carpeta llamada `.well-known` con un archivo de configuración llamado `assetlinks.json` en la raíz del dominio que queremos asociar. Este archivo tiene que poder ser accedido públicamente. 

Ejemplo: https://plataforma.coderhouse.com/.well-known/assetlinks.json


El archivo tiene que tener la siguiente estructura:
```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example",
    "sha256_cert_fingerprints":
    ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
}]
```

Lo único que hay que modificar es el `package_name` y el `sha256_cert_fingerprints`. El json entero con los datos correctos se puede sacar desde la consola de desarrollador de GooglePlay.

>![](/images/deeplink_webcredentials/play_console_assetlink.png)

Una vez que el archivo esta hosteado se puede testear utilizando la siguiente herramienta [Test assetlinks.json](https://developers.google.com/digital-asset-links/tools/generator)

>![](/images/deeplink_webcredentials/assetlink_test.png)

#### iOS

El proceso en iOS es muy similar al de Android, junto con el archivo `assetlinks.json` tenemos que hostear otro llamado `apple-app-site-association` (si bien es un archivo json es muy importante que el archivo subido NO tenga ninguna extension).

Ejemplo: https://plataforma.coderhouse.com/.well-known/apple-app-site-association

El archivo tiene que tener la siguiente estructura:
```json
{
  "applinks": {
    "details": [
      {
        "appIDs": [ "2FM736MG22.com.coderhouse.mobile" ],
        "components": [
          {
            "/": "/test/*"
          }
        ]
      }
    ]
  }
}
```
Los datos del campo `appIDs` los sacamos de la consola de desarrollo del AppStore.

En el ejemplo anterior nuestra app está reaccionando unicamente a los links dentro del path "/test/*". A diferencia de Android los paths que soporta nuestra app se definen en el archivo hosteado.

Podemos testear que el archivo se haya subido correctamente utilizando la siguiente herramienta [AASA Validator](https://branch.io/resources/aasa-validator/)

>![](/images/deeplink_webcredentials/aasa_validator.png)

>![](/images/deeplink_webcredentials/aasa_validator_result.png)

### 2 - Vincular el dominio a nuestra app

Ahora solo falta agregar el dominio en nuestra app en Flutter. 

#### Android

Para Android tenemos que modificar el archivo `AndroidManifest.xml` y agregar las siguientes lineas:

```xml
<!-- Deep linking -->
<meta-data android:name="flutter_deeplinking_enabled" android:value="true" />
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="DOMINIO_ASOCIADO" android:pathPrefix="PATH"/>
</intent-filter>
<!-- Deep linking -->
```
Solo hay que modificar el host y el path en caso que solo queramos reaccionar a ciertas rutas dentro del dominio. En caso que queramos reaccionar a multiples paths dentro del mismo dominio debemos agregar multiples lineas con el tag `data` y la misma estructura.

Ejemplo: en esta configuración solo estamos reaccionando al link `https://plataforma.coderhouse.com/test`

>![](/images/deeplink_webcredentials/android_manifest.png)

Podemos testear la configuración simulando la apertura de un link. Con el emulador abierto y la app en background o killed procedemos a abrir una terminal y usamos el siguiente comando:
`adb shell am start -W -a android.intent.action.VIEW -d <URI> <PACKAGE>`

Si la configuración se hizo correctamente se debería abrir un menu inferior preguntándonos si queremos abrir el link con la app o con el navegador.

Para mas info:
[Guía oficial de Google](https://developer.android.com/training/app-links)
[Android App Linking - The Ultimate Guide](https://simonmarquis.github.io/Android-App-Linking/)


#### iOS

En iOS esto se debe hacer desde XCode siguiendo los pasos listados en el siguiente link en la sección `Add the Associated Domains Entitlement to Your App`:
[Apple Developer - Supporting Associated Domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains)

Ejemplo del dominio a asociar: `applinks:plataforma.coderhouse.com`

Para mas info:
[Guía oficial de Apple](https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app)

### 3 - Implementar uni_links

Ahora que tenemos el dominio asociado a nuestra app solo falta escribir el código que se va a ejecutar cuando a nuestra app le llegue un link. Para eso vamos a usar la librería [uni_links](https://pub.dev/packages/uni_links) que nos facilita listeners y handlers para realizar esta tarea.

Una vez que instalamos la librería tenemos que agregar las siguientes lineas en `main.dart`

```dart
/// Esta linea va a la altura del metodo main()
/// Para saber si ya se leyó la url con la que se abrió la app si estaba killed
bool _initialUriIsHandled = false;


class _MyAppState extends State<MyApp> {
  /// Para deeplink
  Uri? _initialUri; // Url con la que se abrio la app si estaba killed
  Uri? _latestUri; // Ultima url que recibio la app en background o foreground
  StreamSubscription? _deeplinkSubscription;

  @override
  void initState() {
    super.initState();

    /// Deeplinks handlers
    _handleIncomingLinks();
    _handleInitialUri();
  }

  @override
  void dispose() {
    _deeplinkSubscription?.cancel();
    super.dispose();
  }


  /// Deeplinks
  /// Handle the initial Uri - the one the app was started with
  Future<void> _handleInitialUri() async {
    if (_initialUriIsHandled) return;

    _initialUriIsHandled = true;
    try {
      final uri = await getInitialUri();
      print(uri == null ? 'No initial uri' : 'Initial uri: $uri');

      if (!mounted) return;
      setState(() => _initialUri = uri);
    } catch (err) {
      if (!mounted) return;
    }
  }

  void _handleIncomingLinks() {
    if (kIsWeb) return;
    // It will handle app links while the app is already started - be it in
    // the foreground or in the background.
    _deeplinkSubscription = uriLinkStream.listen((Uri? uri) async {
      if (!mounted) return;
      print('Received uri: $uri');
      setState(() {
        _latestUri = uri;
      });
    }, onError: (Object err) {
      if (!mounted) return;
      setState(() {
        _latestUri = null;
      });
    });
  }

  /// End deeplinks

  ...
```

Según los requisitos de nuestra app debemos modificar los dos handlers para que parseen la url y la app reaccione a ella.

---

## Implementar WebCredentials
#### Android
Primero tenemos que agregar una lineas extras en el archivo `assetlinks.json`

```json
[{
  "relation": ["delegate_permission/common.get_login_creds"],
  "target": {
    "namespace": "web",
    "site": "https://signin.example.com"
  }
},
{
  "relation": ["delegate_permission/common.handle_all_urls", "delegate_permission/common.get_login_creds"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example",
    "sha256_cert_fingerprints":
    ["14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"]
  }
}]
```
Agregamos un objeto al principio con la web que queremos vincular y en el array `"relation"` de nuestra app agregamos `"delegate_permission/common.get_login_creds"`

Luego tenemos que agregar la siguiente linea en el `AndroidManifest.xml`

```xml
<meta-data android:name="asset_statements" android:resource="@string/asset_statements"/>
```

>![](/images/deeplink_webcredentials/android_manifest_webcredentials.png)

Por ultimo hay que crear un archivo llamado `strings.xml` en la carpeta `/android/app/src/main/res/values` y agregarle las siguiente linea:

```xml
<resources>
  <string name="asset_statements" translatable="false">
    [{
      \"include\": \"https://plataforma.coderhouse.com/.well-known/assetlinks.json\"
    }]
  </string>
</resources>
```

Cambiar el Link por el de tu `assetlink.json` 
>![](/images/deeplink_webcredentials/strings_values_file.png)


#### iOS

Para iOS solo hay que agregar un dominio desde XCode con el prefijo `webcredentials`
[Apple Developer - Supporting Associated Domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains)

Ejemplo del dominio a asociar: `webcredentials:plataforma.coderhouse.com`
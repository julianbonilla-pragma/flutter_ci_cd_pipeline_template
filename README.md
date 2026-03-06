# Plantilla CI/CD para Flutter (Android e iOS)
Este proyecto es una plantilla de CI/CD para proyectos Flutter que automatiza calidad, pruebas y despliegues a tiendas.

- ## 1. Configuración en GitHub

En la carpeta .github/workflows se definen tres workflows principales:

- New Feature (features.yaml): para ramas feature/** y PR hacia dev o main.
- Open Test (beta.yaml): para push a dev, ejecuta calidad y despliegues de prueba.
- Production (prod.yaml): para push a main, ejecuta calidad y despliegues a producción.

Flujo recomendado de ramas:

- feature/** → pull request hacia dev.
- dev → despliegues de prueba abierta (Android) y TestFlight (iOS).
- main → despliegues a producción (Google Play y App Store).

- ## 2. Secrets en GitHub

Los secrets se configuran en Settings → Secrets and variables → Actions.

### 2.1 Calidad y cobertura

- CODECOV_TOKEN: token del proyecto en Codecov, usado para subir la cobertura en los tres workflows.

### 2.2 Android (codificados en base64)

Estos secrets se almacenan como base64 y se decodifican en los workflows:

Codificación: `base64 -i ruta/al/archivo | pbcopy`

- ANDROID_KEYSTORE: contenido de android/keystore.jks en base64.
- ANDROID_KEY_PROPERTIES: contenido de android/key.properties en base64.
- ANDROID_RELEASE_SERVICE_ACCOUNT: contenido en base64 del JSON de la cuenta de servicio de Google Play.

### 2.3 iOS / App Store Connect

- APP_STORE_CONNECT_API_KEY_ID: Key ID de la API Key.
- APP_STORE_CONNECT_ISSUER_ID: Issuer ID de la cuenta.
- APP_STORE_CONNECT_API_KEY_CONTENT: contenido completo del archivo .p8 (sin base64).

### 2.4 iOS / Fastlane Match

- MATCH_PASSWORD: contraseña usada por Match para cifrar certificados.
- MATCH_GIT_PRIVATE_KEY: clave privada SSH (id_rsa) con acceso al repo de certificados.

Para usar Match es necesario crear previamente un repositorio **privado** (por ejemplo en GitHub) que actuará como almacén de certificados y perfiles. Ese repo se referencia en `git_url` del Matchfile y se accede únicamente vía SSH usando la clave configurada en `MATCH_GIT_PRIVATE_KEY`.

- ## 3. Android

### 3.1 Configuración general

En el job build-android de .github/workflows/beta.yaml se usan variables como:

- APP_PACKAGE_NAME: nombre del paquete (applicationId en Android).
- AAB_PATH: ruta donde se espera el archivo .aab.
- KEYSTORE_PATH, KEY_PROPS_PATH, SERVICE_ACCOUNT_PATH: rutas donde se decodifican los secrets base64.

Los archivos sensibles (keystore.jks, key.properties y JSON de cuenta de servicio) **no se versionan**, solo se inyectan vía secrets codificados en base64.

**Origen de certificados Android y claves:**

- El keystore (`keystore.jks`) se genera localmente con `keytool` o desde Android Studio.
- Las credenciales de `key.properties` (storePassword, keyPassword, etc.) las defines al crear el keystore.
- El JSON de la cuenta de servicio se descarga desde Google Cloud / Google Play Console al crear una cuenta de servicio con permisos sobre tu app.

### 3.2 Flujo de beta (Open Test)

Desencadenante: push a la rama dev.

- Job quality: valida formato, análisis, pruebas y cobertura mínima.
- Job build-android: decodifica los secrets de Android, instala dependencias y ejecuta `flutter build appbundle`.
- Job deploy-android: descarga el .aab y lo sube a la pista beta (prueba abierta) en Google Play usando r0adkll/upload-google-play. Los **release notes** de cada versión se leen desde la carpeta raíz `whatsnew` (por ejemplo, archivos `whatsnew/whatsnew-en-US`, etc.).

### 3.3 Flujo de producción (Google Play Store)

Desencadenante: push a la rama main.

- Job quality: mismas validaciones que en beta.
- Job promote-android: usa kevin-david/promote-play-release para promover el release desde la pista beta a production, usando el JSON de cuenta de servicio decodificado.

- ## 4. iOS

### 4.1 Archivos de configuración

En la carpeta ios se usan principalmente:

- ios/Gemfile: define la dependencia de fastlane.
- ios/fastlane/Appfile: contiene app_identifier, apple_id e itc_team_id.
- ios/fastlane/Matchfile: define git_url (repo privado de certificados) y type (appstore).
- ios/fastlane/Fastfile: define las lanes sync_signing, build, upload_testflight y promote_to_app_store.

**Origen de certificados iOS y perfiles de aprovisionamiento:**

- Los certificados de distribución y los perfiles de aprovisionamiento provienen de tu cuenta del Apple Developer Program.
- Fastlane Match puede crear nuevos certificados/perfiles o reutilizar los existentes en el portal de desarrolladores de Apple.
- Una vez creados/ubicados, Match los descarga, los cifra con `MATCH_PASSWORD` y los sube al repo privado configurado en `git_url`.

Antes de integrar con CI se asume que, de forma local, se ejecutó:

- Exportar variables de entorno de la API Key de App Store Connect y MATCH_PASSWORD.
- bundle install dentro de ios.
- Crear un repositorio git **privado** para que Match guarde certificados y perfiles (este repo se configura en `git_url` del Matchfile).
- Ejecutar una lane de Match para poblar ese repo con certificados y perfiles, por ejemplo:
	- Desde la carpeta `ios`: `bundle exec fastlane match appstore`
- Configurar SSH al repo y usar esa clave privada como MATCH_GIT_PRIVATE_KEY.

### 4.2 Flujo de beta (TestFlight)

Desencadenante: push a la rama dev.

- Job build-ios (beta.yaml):
	- Instala Flutter, Xcode, Ruby, CocoaPods y gems.
	- Configura SSH con MATCH_GIT_PRIVATE_KEY para acceder al repo de Match.
	- Ejecuta `bundle exec fastlane sync_signing`.
	- Ejecuta `bundle exec fastlane build` usando el número de build y versión derivados de github.run_number.
	- Publica el IPA como artefacto.
- Job deploy-ios (beta.yaml): descarga el IPA y ejecuta `bundle exec fastlane upload_testflight` para subirlo a TestFlight.

### 4.3 Flujo de producción (App Store)

Desencadenante: push a la rama main.

- Job promote-ios (prod.yaml):
	- Instala Ruby y dependencias en ios.
	- Obtiene la versión desde pubspec.yaml.
	- Ejecuta `bundle exec fastlane promote_to_app_store` pasando versión y número de build (github.run_number) para promover un build existente de TestFlight a App Store.

### 4.4 Comandos locales útiles

Desde la carpeta ios se pueden ejecutar, por ejemplo:

- `bundle install`
- `bundle exec fastlane sync_signing`
- `bundle exec fastlane build build_number:1 version_name:1.0.0`
- `bundle exec fastlane upload_testflight`
- `bundle exec fastlane promote_to_app_store version:1.0.0 build_number:1`

- ## 5. Acciones externas (GitHub Marketplace)

Acciones externas más relevantes usadas por los workflows:

- Very Good Workflows (semantic PR, spell check, quality): https://github.com/VeryGoodOpenSource/very_good_workflows
- Very Good Coverage (umbral de cobertura): https://github.com/VeryGoodOpenSource/very_good_coverage
- Flutter Action (instalar Flutter): https://github.com/subosito/flutter-action
- Codecov Action (subir cobertura): https://github.com/codecov/codecov-action
- Upload to Google Play (subida de AAB a Play Console): https://github.com/r0adkll/upload-google-play
- Promote Play Release (promoción de beta a production): https://github.com/kevin-david/promote-play-release
- Setup Ruby: https://github.com/ruby/setup-ruby
- Setup Xcode: https://github.com/maxim-lobanov/setup-xcode

## 6. Consideraciones finales

- No se deben versionar certificados, keystores ni JSON de cuentas de servicio.
- Todos los valores sensibles deben ir en secrets de GitHub y, para iOS, en el repo privado de Match.
- Reemplaza los placeholders (com.example.app, TU_FLAVOR, example_app.ipa, SSH_GET_REPO_URL, etc.) por los valores reales de tu proyecto.
- Mantén sincronizadas las versiones entre pubspec.yaml y la configuración de Fastlane/Xcode.
- Ante fallos en CI, revisa los logs de cada job (quality, build-android, deploy-android, build-ios, deploy-ios, promote-android, promote-ios) para identificar qué secret o configuración falta.
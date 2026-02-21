---
name: integrate-embrace
description: Integrate the Embrace Android SDK into an Android app with logcat telemetry export
disable-model-invocation: true
user-invocable: true
argument-hint: "[path-to-android-project]"
allowed-tools: Read, Glob, Grep, Write, Edit, Bash(./gradlew *), Bash(adb *)
---

You are integrating the Embrace Android SDK into an Android project. Follow these steps precisely.

## Prerequisites Check

1. Confirm you are in an Android project by finding `build.gradle.kts` or `build.gradle` at the root, and `settings.gradle.kts` or `settings.gradle`.
2. If `$ARGUMENTS` is provided, `cd` to that directory first.
3. If this is not an Android project, inform the user and stop.

## Step 1: Analyze Project Structure

Gather all of the following before making any changes:

- [ ] Root build file path and type (kts or groovy)
- [ ] Settings file path and type
- [ ] Whether `libs.versions.toml` exists at `gradle/libs.versions.toml`
- [ ] The app module name and path (find the module applying `com.android.application`)
- [ ] The app's `namespace` or `applicationId` (from the app build file)
- [ ] The app's source root (`src/main/kotlin/` or `src/main/java/`)
- [ ] The existing Application class (from `AndroidManifest.xml` `android:name` attribute)
- [ ] Whether Embrace is already integrated (check for `io.embrace` in build files)
- [ ] Whether `mavenCentral()` is in repositories
- [ ] The app's `minSdk` value (check app build file, settings.gradle.kts, or version catalog)
- [ ] Whether core library desugaring is already enabled (`isCoreLibraryDesugaringEnabled` in compileOptions)
- [ ] The app's language (Kotlin preferred; fall back to Java if no Kotlin source exists)

Report your findings to the user before proceeding. If Embrace is already integrated, inform the user and stop.

## Step 2: Add Gradle Plugin and Dependencies

**Current Embrace SDK version: 8.1.0**
**Current OpenTelemetry BOM version: 1.59.0**

### If using version catalog (`libs.versions.toml`):

Add to `[versions]`:
```toml
embrace = "8.1.0"
opentelemetry = "1.59.0"
```

Add to `[plugins]`:
```toml
embrace = { id = "io.embrace.gradle", version.ref = "embrace" }
```

Add to `[libraries]`:
```toml
embrace-otel-java = { group = "io.embrace", name = "embrace-android-otel-java", version.ref = "embrace" }
opentelemetry-bom = { group = "io.opentelemetry", name = "opentelemetry-bom", version.ref = "opentelemetry" }
opentelemetry-sdk = { group = "io.opentelemetry", name = "opentelemetry-sdk" }
```

In the app module's build file, add to `plugins {}`:
```kotlin
alias(libs.plugins.embrace)
```

Add to `dependencies {}`:
```kotlin
implementation(libs.embrace.otel.java)
implementation(platform(libs.opentelemetry.bom))
implementation(libs.opentelemetry.sdk)
```

### If NOT using version catalog:

Ensure `settings.gradle.kts` has `mavenCentral()` in `pluginManagement.repositories` and add:
```kotlin
plugins {
    id("io.embrace.gradle") version "8.1.0" apply false
}
```

In the app module build file, add to `plugins {}`:
```kotlin
id("io.embrace.gradle")
```

Add to `dependencies {}`:
```kotlin
implementation("io.embrace:embrace-android-otel-java:8.1.0")
implementation(platform("io.opentelemetry:opentelemetry-bom:1.59.0"))
implementation("io.opentelemetry:opentelemetry-sdk")
```

Also ensure `mavenCentral()` is in `dependencyResolutionManagement.repositories` in the settings file.

### For Groovy build files:

Use `id 'io.embrace.gradle'` syntax instead of `id("io.embrace.gradle")`.
Use `implementation 'group:name:version'` syntax instead of parenthesized form.

## Step 3: Handle Core Library Desugaring (minSdk < 26 only)

If the project's `minSdk` is **26 or higher**, skip this step entirely.

If the project's `minSdk` is **less than 26**, the Embrace SDK requires core library desugaring. Apply ALL of the following:

### 3a. Enable desugaring in the app module's build file

Add inside the `android {}` block (create `compileOptions` if it doesn't exist):
```kotlin
compileOptions {
    isCoreLibraryDesugaringEnabled = true
}
```

For Groovy:
```groovy
compileOptions {
    coreLibraryDesugaringEnabled true
}
```

### 3b. Add the desugaring dependency

Add to the app module's `dependencies {}`:
```kotlin
coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.5")
```

### 3c. Add the dexing transform property

Add to `gradle.properties`:
```
android.useFullClasspathForDexingTransform=true
```

This property is required by the Embrace Gradle plugin when minSdk < 26. Without it the build will fail with a clear error message.

**Note**: If desugaring is already enabled (check for `isCoreLibraryDesugaringEnabled = true` and a `coreLibraryDesugaring` dependency), only add `android.useFullClasspathForDexingTransform=true` to `gradle.properties` if it's missing.

## Step 4: Create embrace-config.json

Create `<app-module>/src/main/embrace-config.json` with content:
```json
{}
```

Do NOT add a trailing newline. If the file already exists, skip this step.

## Step 5: Create Logcat Exporters

Determine the package name from the app's `namespace` or `applicationId`.
Create two files in the app's main source directory, in the same package as the Application class (or the root package if no Application class exists).

Read the templates from the `templates/` directory in this skill's directory and substitute `{{PACKAGE_NAME}}` with the correct package name.

The template files are:
- `templates/LogcatSpanExporter.kt.tmpl` -> `LogcatSpanExporter.kt`
- `templates/LogcatLogRecordExporter.kt.tmpl` -> `LogcatLogRecordExporter.kt`

If the project uses Java instead of Kotlin, create equivalent `.java` files.

## Step 6: Wire Up the Application Class

### If an Application subclass already exists:

Add these imports at the top of the file:
```kotlin
import io.embrace.android.embracesdk.Embrace
import io.embrace.android.embracesdk.otel.java.addJavaSpanExporter
import io.embrace.android.embracesdk.otel.java.addJavaLogRecordExporter
```

**If the class already has an `onCreate()` method**, add these lines at the BEGINNING of `onCreate()`, right after `super.onCreate()`:
```kotlin
Embrace.addJavaSpanExporter(LogcatSpanExporter())
Embrace.addJavaLogRecordExporter(LogcatLogRecordExporter())
Embrace.start(this)
```

**If the class does NOT have an `onCreate()` method** (e.g., `class MyApp : Application()` on a single line), add a full `onCreate()` override with a class body:
```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        Embrace.addJavaSpanExporter(LogcatSpanExporter())
        Embrace.addJavaLogRecordExporter(LogcatLogRecordExporter())
        Embrace.start(this)
    }
}
```
Preserve any existing annotations (e.g., `@HiltAndroidApp`) on the class.

### If no Application subclass exists:

Create a new `MainApplication.kt` file in the source root with the correct package:

```kotlin
package <app.package.name>

import android.app.Application
import io.embrace.android.embracesdk.Embrace
import io.embrace.android.embracesdk.otel.java.addJavaSpanExporter
import io.embrace.android.embracesdk.otel.java.addJavaLogRecordExporter

class MainApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Embrace.addJavaSpanExporter(LogcatSpanExporter())
        Embrace.addJavaLogRecordExporter(LogcatLogRecordExporter())
        Embrace.start(this)
    }
}
```

Then update `AndroidManifest.xml` to add `android:name=".MainApplication"` to the `<application>` tag.

**Important**: Exporter registration MUST come before `Embrace.start()`. The SDK silently ignores exporters added after start.

## Step 7: Build

Run `./gradlew :<app-module>:assembleDebug` to verify the project compiles.

If it fails, analyze the error and attempt to fix it. Common issues:
- Missing `mavenCentral()` in repositories
- Version conflicts with OpenTelemetry
- "Desugaring must be enabled when minSdk is < 26" -- add `compileOptions { isCoreLibraryDesugaringEnabled = true }` and the desugaring dependency (see Step 3)
- "must use AGP 8.3.0+ and add android.useFullClasspathForDexingTransform=true" -- add the property to `gradle.properties` (see Step 3c)

## Step 8: Verify on Device

1. Confirm a device or emulator is connected by running `adb devices`. If no device is listed, ask the user to connect one and stop.
2. Install and launch the app:
   ```
   adb install -r <app-module>/build/outputs/apk/debug/<apk-name>.apk
   adb shell am start -n <applicationId>/<applicationId>.MainActivity
   ```
   Adjust the activity name if the launcher activity is different (check `AndroidManifest.xml` for the activity with `android.intent.action.MAIN`).
3. Wait a few seconds for the app to start, then capture logcat output filtered for Embrace span exports:
   ```
   adb logcat -d -s EmbraceSpans
   ```
4. If the output contains any span data (any spans being emitted indicate the SDK has started successfully), the integration is successful.
5. If no spans appear:
   - Verify the app actually launched: `adb shell pidof <applicationId>`
   - Try a broader filter: `adb logcat -d | grep -i embrace`
   - Check that `Embrace.start()` is called after the exporter registration in the Application class
   - Ensure the logcat exporters were created correctly

If verification succeeds, inform the user:
- "Embrace SDK integrated and verified. Spans are being emitted to logcat."
- "To send data to the Embrace dashboard, create an account at https://dash.embrace.io and add your app_id and api_token to embrace-config.json."

If verification fails after troubleshooting, inform the user of what was observed and suggest they check the device logs manually.

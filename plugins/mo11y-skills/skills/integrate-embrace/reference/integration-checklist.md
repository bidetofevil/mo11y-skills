# Embrace SDK Integration Checklist

This reference document covers all supported project configurations and edge cases for the `/integrate-embrace` skill.

## Supported Project Configurations

### Build File Formats

| Format | Plugin Syntax | Dependency Syntax |
|--------|--------------|-------------------|
| Kotlin DSL (`.kts`) | `id("io.embrace.gradle")` | `implementation("group:name:version")` |
| Groovy DSL (`.gradle`) | `id 'io.embrace.gradle'` | `implementation 'group:name:version'` |

### Dependency Management

| Style | Plugin Declaration | Library Declaration |
|-------|-------------------|---------------------|
| Version Catalog (`libs.versions.toml`) | `alias(libs.plugins.embrace)` | `implementation(libs.embrace.otel.java)` |
| Direct Declaration | `id("io.embrace.gradle") version "8.1.0"` | `implementation("io.embrace:embrace-android-otel-java:8.1.0")` |

### Language Support

| Language | Exporter Files | Application Class |
|----------|---------------|-------------------|
| Kotlin | `.kt` files with Kotlin idioms | Kotlin class extending `Application` |
| Java | `.java` files with Java idioms | Java class extending `Application` |

## Edge Cases

### Already Integrated
- Check for `io.embrace` in build files (plugin or dependency)
- If found, inform user and stop -- do not double-integrate

### No Application Subclass
- Create `MainApplication.kt` (or `.java`) in the root package
- Register in `AndroidManifest.xml` with `android:name=".MainApplication"`

### Existing Application Class in Java
- Add Java equivalents of the exporter registration and `Embrace.start(this)`
- Use `Embrace.addJavaSpanExporter()` and `Embrace.addJavaLogRecordExporter()` (static methods)

### Multi-Module Projects
- The `com.android.application` plugin identifies the app module (may not be named `app/`)
- Only integrate into the app module, not library modules
- If multiple app modules exist, ask the user which one

### Missing `mavenCentral()` in Repositories
- Check both `pluginManagement.repositories` and `dependencyResolutionManagement.repositories`
- Add `mavenCentral()` if missing

### Existing OpenTelemetry Dependencies
- If the project already depends on `io.opentelemetry`, check version compatibility
- The Embrace SDK 8.1.0 is compatible with OpenTelemetry BOM 1.59.0
- If there's a version conflict, prefer the higher version (newer OTel is backward compatible)

### ProGuard / R8
- No special rules needed -- the Embrace Gradle plugin handles this automatically
- The plugin adds consumer ProGuard rules that are applied during app builds

### Existing Application Class Without `onCreate()`
- Some projects have minimal Application subclasses (e.g., `class MyApp : Application()` with no body)
- Common with Hilt (`@HiltAndroidApp class MyApp : Application()`)
- Must convert to a full class body with `onCreate()` override
- Preserve all existing annotations on the class

### Core Library Desugaring (minSdk < 26)
- The Embrace SDK requires core library desugaring when `minSdk < 26` due to `java.time` usage
- Three things are needed:
  1. Enable desugaring in `compileOptions`:
     ```kotlin
     android {
         compileOptions {
             isCoreLibraryDesugaringEnabled = true
         }
     }
     ```
  2. Add the desugaring dependency:
     ```kotlin
     dependencies {
         coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.5")
     }
     ```
  3. Add to `gradle.properties`:
     ```
     android.useFullClasspathForDexingTransform=true
     ```
- The Embrace Gradle plugin enforces requirement #3 and will fail the build with a clear error if missing
- If desugaring is already enabled, only add `android.useFullClasspathForDexingTransform=true` if missing

### Content-Filtered Repositories
- Some projects use `content {}` blocks to restrict which groups are fetched from each repository
- The Embrace Gradle plugin resolves from `gradlePluginPortal()` (unfiltered) if present
- The Embrace SDK libraries resolve from any unfiltered `mavenCentral()`
- If `mavenCentral()` in `dependencyResolutionManagement` has a `content {}` filter, add `includeGroupByRegex("io\\.embrace.*")` and `includeGroupByRegex("io\\.opentelemetry.*")`

### Compose vs View-Based Projects
- Integration is identical for both -- the SDK's compose tap instrumentation is handled automatically by the Gradle plugin's bytecode instrumentation
- No special setup needed for either

## Verification

After integration, run:
```bash
./gradlew :<app-module>:assembleDebug
```

Common build failures and fixes:
1. **Missing repository**: Add `mavenCentral()` to both `pluginManagement` and `dependencyResolutionManagement`
2. **Version conflict**: Check for existing OTel deps and align versions
3. **"Desugaring must be enabled"**: Add `compileOptions { isCoreLibraryDesugaringEnabled = true }` and `coreLibraryDesugaring` dependency
4. **"must add android.useFullClasspathForDexingTransform=true"**: Add this property to `gradle.properties`
5. **Duplicate class**: If another SDK bundles OTel classes, use `exclude` in dependency declaration
6. **Content-filtered repository**: If `mavenCentral()` has `content {}` filters, add `io.embrace` and `io.opentelemetry` groups

## Logcat Verification

After running the app, filter logcat:
- Tag `EmbraceSpans` -- shows completed spans with trace/span IDs
- Tag `EmbraceLogs` -- shows log records with severity and attributes

Expected output within ~10 seconds of app launch:
- Startup spans (app launch, activity creation)
- Session lifecycle logs
- Any user-triggered telemetry
# Technical report — BlockStart ARMv7-compatible external launcher

Date: 2026-10-30

## 1. Selected architecture

A standalone Android Activity written in Java, using the Android SDK only. It is intentionally an **external launcher**: the installed Minecraft application remains the game process and owns its rendering, native loading, lifecycle, files, and licensing.

The launcher contains no JNI source and no third-party AARs. As a result, there are no packaged native libraries to ABI-filter or relocate. On a 32-bit Android 10 installation, Android runs the app in the 32-bit runtime available to that OS. The screen reports the process bitness with `Process.is64Bit()` and lists `Build.SUPPORTED_ABIS`; it does not infer process architecture solely from the SoC's capabilities.

## 2. Minecraft launch mechanism

Target package: `com.mojang.minecraftpe`.

The app declares visibility for only that package in `<queries>`, then calls `PackageManager.getLaunchIntentForPackage(packageName)`. If the method returns `null`, the UI shows that the package is installed but Android did not provide a front-door launch intent. The Activity starts the returned intent using `startActivity()`; it does not hardcode Minecraft's internal Activity class name.

Missing package, null launch intent, `ActivityNotFoundException`, `SecurityException`, and other runtime failures are surfaced in the UI. A boolean launch guard prevents a rapid second tap while a launch request is outstanding.

## 3. Expected application structure

- `org.pococ3.blockstart`: launcher package ID.
- `com.mojang.minecraftpe`: detected target package ID.
- Minecraft's base APK and any split APKs stay installed and managed by Android. The launcher reads package metadata only; it does not open, extract, copy, or rewrite `sourceDir`, `splitSourceDirs`, app data, or worlds.

The launcher displays version name, version code, and split count when Android exposes them. A split count is informational only; it does not mean the launcher has loaded or validated those splits.

## 4. Native dependencies and ABI

None. No NDK, CMake, Vulkan, OpenGL, `GameActivity`, Preloader, FMOD, PairIP, PlayFab, or `libminecraftpe.so` is packaged or loaded by this app. CI asserts that any future `.so` in the APK is under `lib/armeabi-v7a/` and has ELF machine `ARM`; the current intended build should contain no `.so` at all.

This design avoids the prior embedded-runtime failure classes: missing proprietary libraries, JNI class-loader binding, classpath gaps, and interception of Minecraft's own Activity. It also cannot remedy an incompatible or missing native library inside the official Minecraft installation.

## 5. ARMv7 and Android compatibility

The target handset is stated to run Android 10 / API 29 with a 32-bit application environment. The app's minimum API is 23 and all APIs used by its main path are available by that level. `Process.is64Bit()` is available from API 23. The actual runtime bitness and supported ABIs are displayed for diagnosis.

The launcher itself is Java-only. An APK without native libraries is not an ARM64-only APK and does not need a fake ABI filter. The game is a separate package and must independently contain a build compatible with the installed Android environment.

## 6. Android build stack

- Android Gradle Plugin: 8.13.2
- Gradle wrapper distribution: 8.13
- JDK: 17 (Temurin in CI)
- Android compile SDK: 35
- Android Build Tools: 35.0.0
- minSdk: 23
- targetSdk: 35
- Unit tests: JUnit 4.13.2

This is a deliberately conservative AGP 8.x stack rather than adopting a newer plugin major version without need. The workflow pins the Gradle distribution SHA-256 and the downloaded wrapper JAR SHA-256.

## 7. CI configuration

GitHub-hosted `ubuntu-24.04` runner. The workflow checks out the repository, extracts `BlockStart-Android-Source.zip`, sets up JDK 17 and Android SDK API 35, bootstraps/verifies the official Gradle wrapper JAR if absent, then runs:

`./gradlew --no-daemon --stacktrace clean testDebugUnitTest lintDebug assembleDebug`

It validates APK application ID, version, min/target SDK metadata, verifies the APK signature with `apksigner`, enumerates native libraries and inspects ELF machine type if any are present, calculates SHA-256, and uploads the APK/checksum/validation text as a workflow artifact.

## 8. Known risks and limitations

1. The app cannot guarantee that the currently installed Minecraft build supports ARMv7; Android itself and Minecraft's own package determine that.
2. Package launch can still fail because Minecraft or Android reports an internal error after the launch intent is started. The external launcher cannot inspect or fix game-internal native crashes.
3. A successful CI build proves that the launcher compiles and passes its automated checks; it does not prove the physical POCO C3 can start Minecraft.
4. The debug APK is signed with Gradle's standard debug key. It is intended for installation/testing, not Play Store publishing.
5. No world backup, import/export, version manager, or native mod loader is implemented. These are deliberately out of scope for this reliable launch-only baseline.

## 9. Research references

- Android NDK ABI guide: https://developer.android.com/ndk/guides/abis
- Android `PackageManager`: https://developer.android.com/reference/android/content/pm/PackageManager#getLaunchIntentForPackage(java.lang.String)
- Android package visibility: https://developer.android.com/training/package-visibility
- Android `ApplicationInfo`: https://developer.android.com/reference/android/content/pm/ApplicationInfo
- Android `Process.is64Bit()`: https://developer.android.com/reference/android/os/Process#is64Bit()
- AGP 8.13 release notes: https://developer.android.com/build/releases/past-releases/agp-8-13-0-release-notes
- Gradle 8.13 checksum reference: https://gradle.org/release-checksums/
- Gradle wrapper verification: https://docs.gradle.org/current/userguide/gradle_wrapper.html
- LeviLaunchroid repository: https://github.com/LiteLDev/LeviLaunchroid
- LeviLaunchroid docs: https://docs.levimc.org/LeviLaunchroid/
- GitHub Actions: https://docs.github.com/en/actions

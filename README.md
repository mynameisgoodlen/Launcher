# BlockStart — Android Minecraft Bedrock external launcher

**Purpose:** launch the official Minecraft Bedrock Android application already installed for the current Android user. This is a small external launcher, not a replacement game runtime.

## Project facts

- App ID: `org.pococ3.blockstart`
- Target game package: `com.mojang.minecraftpe`
- `minSdk`: 23 (Android 6.0+; includes Android 10 / API 29)
- `compileSdk` / `targetSdk`: 35
- Java-only application: no JNI, no `.so` files, no NDK, no ARM64-only dependency
- No Android permissions are requested
- No game APK, game asset, world, save, or proprietary library is included or copied

## Build using GitHub Actions

1. Upload this ZIP to the repository root with the exact filename `BlockStart-Android-Source.zip`.
2. Put the separate workflow file at `.github/workflows/build-blockstart.yml`.
3. Open **Actions → Build BlockStart Android APK → Run workflow**.
4. When the workflow completes, open its **Artifacts** section and download `BlockStart-debug-apk`.

The workflow extracts this archive automatically, downloads the official Gradle 8.13 wrapper JAR only when it is missing, checks that JAR against Gradle's published SHA-256 checksum, and then runs tests, lint, and the APK build. No local Android Studio, Java, Gradle, SDK, Termux, or ADB setup is required.

## What the app does

- Asks Android `PackageManager` for the installed package `com.mojang.minecraftpe`.
- Displays the installed version name and version code when Android exposes them.
- Displays whether the package has split APKs without reading or extracting them.
- Gets the launch intent with `PackageManager.getLaunchIntentForPackage()` and starts it from the launcher's Activity.
- Reports a missing installation, missing launch intent, or caught launch exception instead of silently failing.
- Opens the official Play Store listing as a convenience when Minecraft is absent. This only opens the listing; it does not download, install, or unlock the game.
- Prevents rapid repeated launch-button taps from starting multiple launch requests.

## Important limitations

- This launcher cannot convert a 64-bit-only Minecraft installation or native game binary into ARMv7. The installed Minecraft build must already be compatible with the device's Android ABI and system libraries.
- The app only asks Android to start Minecraft. It does not embed `GameActivity`, load Minecraft libraries, patch package state, imitate Google Play billing, or bypass ownership checks.
- The exact Minecraft version's ability to run on a particular handset is not proven by the launcher's build or unit tests. Validate launch on the actual POCO C3 with the official game installed.
- GitHub-hosted CI can build and statically validate the launcher, but it cannot certify that a remote runner has launched the official game on the POCO C3.
- The app deliberately does not provide version importing, mod injection, custom class loaders, world access, or APK extraction. Those require a separate design and compatibility investigation.

## Android package visibility

The manifest declares a single `<queries><package android:name="com.mojang.minecraftpe"/></queries>` entry. It does not request `QUERY_ALL_PACKAGES` or enumerate unrelated installed apps.

## Source reuse

The LeviLaunchroid repository was inspected for its game-package manager, ABI selection, split-APK source handling, native-library extraction, and embedded `GameActivity`/preloader architecture. No LeviLaunchroid source code is copied into this project. BlockStart uses Android's ordinary external-application launch path, so it does not need LeviLaunchroid's embedded game runtime or its native dependencies.

## Reproducibility

The source archive pins Android Gradle Plugin 8.13.2 and Gradle 8.13. The Gradle distribution checksum is pinned in `gradle/wrapper/gradle-wrapper.properties`. CI downloads the corresponding wrapper JAR and verifies its SHA-256 before invoking `./gradlew`.

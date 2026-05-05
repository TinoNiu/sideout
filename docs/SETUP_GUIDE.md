# Sideout — Setup Guide (Windows + Codemagic for iOS)

This guide takes you from a fresh Windows machine to running a Flutter app on the Android emulator, with a path to iOS builds via Codemagic when you need them.

**Estimated time:** Half a day. Don't try to do this in 30 minutes between meetings — the SDK downloads alone take a while.

---

## Part 1 — Local development (Windows)

### Step 1: Install Git

Download Git for Windows from https://git-scm.com/download/win. During install, accept all defaults except:
- **Default editor:** choose "Visual Studio Code" if you'll use VS Code
- **PATH environment:** "Git from the command line and also from 3rd-party software"
- **Line ending conversions:** "Checkout Windows-style, commit Unix-style"

Verify by opening Command Prompt or PowerShell and running:
```
git --version
```

You should see something like `git version 2.43.0.windows.1`.

### Step 2: Install Flutter SDK

1. Go to https://docs.flutter.dev/get-started/install/windows
2. Download the Flutter SDK zip (~1 GB)
3. Extract it to `C:\flutter`. **Do not put it in `Program Files`** — the space breaks things.
4. Add Flutter to your PATH:
   - Press Windows key, type "environment variables", open "Edit the system environment variables"
   - Click "Environment Variables…"
   - Under "User variables", select `Path` and click Edit
   - Click "New" and add `C:\flutter\bin`
   - OK, OK, OK out

5. Open a NEW PowerShell window (it must be new to pick up PATH changes) and run:
   ```
   flutter --version
   ```

   You should see Flutter version info. If you see "command not found," PATH didn't update — restart your computer if needed.

### Step 3: Install Android Studio

Even if you'll write code in VS Code, you need Android Studio for the Android SDK and emulator.

1. Download from https://developer.android.com/studio
2. Run the installer with default options
3. On first launch, the setup wizard installs the Android SDK. Let it finish completely.
4. Open Android Studio → "More Actions" → "SDK Manager"
5. In "SDK Platforms" tab, check **Android 14 (API 34)** at minimum, plus Android 8 (API 26) for compatibility testing
6. In "SDK Tools" tab, ensure these are checked:
   - Android SDK Build-Tools
   - Android SDK Command-line Tools (latest)
   - Android Emulator
   - Android SDK Platform-Tools
7. Apply, accept licenses, wait for downloads.

### Step 4: Create an Android emulator

1. Android Studio → "More Actions" → "Virtual Device Manager"
2. Click "Create Device"
3. Choose **Pixel 7** (good middle-of-the-road profile)
4. Choose system image **API 34** (Android 14). Download it if needed.
5. Name it "Pixel_7_API_34" and click Finish
6. Click the play button next to the device to launch it. First boot takes 1-2 minutes.

If the emulator is painfully slow, ensure Hyper-V or HAXM is enabled in your BIOS. On most modern Windows machines this is automatic.

### Step 5: Install VS Code and extensions

1. Download from https://code.visualstudio.com/
2. Install with default options, plus check "Add to PATH" if offered
3. Open VS Code, press Ctrl+Shift+X to open Extensions, install:
   - **Flutter** (publisher: Dart Code) — installs Dart automatically
   - **Material Icon Theme** (optional, makes file tree more readable)
   - **Error Lens** (optional, surfaces errors inline)

### Step 6: Run flutter doctor

Back in PowerShell:
```
flutter doctor
```

You'll see something like:
```
[√] Flutter (Channel stable, 3.16.0)
[√] Windows Version
[√] Android toolchain - develop for Android devices
[√] Chrome - develop for the web
[X] Visual Studio - develop Windows apps      <- ignore unless building Windows desktop
[!] Android Studio (version X)
[√] VS Code
[√] Connected device
```

Run any commands `flutter doctor` suggests. The most common one:
```
flutter doctor --android-licenses
```
Press `y` and Enter for each license. There are several.

If `flutter doctor` reports an issue under Android toolchain or VS Code, fix those before continuing. Visual Studio (for Windows desktop) and Xcode (for iOS) being missing is fine — we don't need them.

---

## Part 2 — Verify everything works

### Create a test project

```
cd C:\
mkdir code
cd code
flutter create flutter_test_app
cd flutter_test_app
```

Make sure your Android emulator is running (from Step 4). Then:

```
flutter run
```

After 2-5 minutes (first builds are slow), you should see the default Flutter counter app running in your emulator. Tap the FAB; the counter should increment.

If this works, your setup is correct. If it doesn't, fix this before going further — debugging your own code is much harder if your toolchain is broken.

---

## Part 3 — The Sideout project

### Clone or create the project

You have two options:
- **A. Use the starter we provide.** Copy the `sideout_starter` folder we generated as your starting point.
- **B. Start fresh.** `flutter create sideout` and copy our `lib/` folder over the generated one.

Either way, from the project folder:
```
flutter pub get
```

This downloads all dependencies listed in `pubspec.yaml`.

If you see errors about Isar or build_runner, run:
```
flutter pub run build_runner build --delete-conflicting-outputs
```

Isar uses code generation — this command generates the database boilerplate. You'll re-run it whenever you change a model class.

### Run on emulator

```
flutter run
```

You should see the Sideout home screen on your emulator.

---

## Part 4 — iOS builds via Codemagic (when ready)

You don't need this until Phase 4 of the plan, when you start beta testing on iPhones. Skip this section for now and come back later.

### When the time comes:

1. Push your code to a GitHub repo (private is fine)
2. Sign up at https://codemagic.io/signup using your GitHub account
3. Authorize Codemagic to access your repo
4. In Codemagic, "Add application" → select your Sideout repo
5. Codemagic auto-detects Flutter projects. For "Build for platforms," check iOS.
6. For iOS code signing, you'll need:
   - An Apple Developer account ($99/year, sign up at https://developer.apple.com/programs/)
   - An iOS Distribution certificate (Codemagic can generate this for you)
   - A provisioning profile (Codemagic can generate this too)
   - The bundle ID `com.sideoutjournal.app` (or whatever you choose) registered in App Store Connect

7. First iOS build takes 20-40 minutes. Subsequent ones, 10-15.

For the first month or two, you can use Codemagic exclusively for iOS builds. Their free tier of 500 minutes/month is plenty for an indie app being built by one person.

If you find yourself building iOS dozens of times a day, that's the signal to either rent a Mac in the cloud (MacInCloud, ~$30/month) or buy a used Mac Mini.

---

## Part 5 — Useful commands cheat sheet

```bash
# Run on connected device (emulator or physical)
flutter run

# Run with hot reload disabled (sometimes needed for Isar)
flutter run --no-fast-start

# Run a specific device
flutter devices              # list devices
flutter run -d <device-id>

# Generate Isar code after model changes
flutter pub run build_runner build --delete-conflicting-outputs

# Clean build artifacts (do this if weird errors)
flutter clean
flutter pub get

# Build a release APK (Android)
flutter build apk --release

# Build a release App Bundle (Android, what Play Store wants)
flutter build appbundle --release

# Update Flutter itself
flutter upgrade

# Run tests
flutter test
```

---

## Part 6 — Common Windows-specific gotchas

**Path length errors during build.** Windows has a 260-character path limit by default. If you see "filename too long" errors, enable long paths:
1. Run `gpedit.msc` as administrator
2. Computer Configuration → Administrative Templates → System → Filesystem
3. Enable "Enable Win32 long paths"

**Slow emulator.** Make sure Hyper-V is enabled in Windows Features. Also try the WHPX accelerator: in Android Studio's emulator settings, set "Graphics" to "Hardware - GLES 2.0".

**Flutter doctor says CMake or Ninja missing.** Install via Visual Studio Build Tools (free): https://visualstudio.microsoft.com/visual-cpp-build-tools/. During install, select "Desktop development with C++."

**Antivirus blocking builds.** Add `C:\flutter` and your project folder to Windows Defender exclusions. AV scanning every file during builds slows them dramatically.

---

## Part 7 — When to ask for help

For Flutter-specific issues:
- Official Discord: https://flutter.dev/community
- r/FlutterDev on Reddit
- Stack Overflow with the `flutter` tag

For Sideout-specific questions, come back to me with:
- The exact error message
- What command you ran
- What you expected to happen vs. what did happen

I can help debug most things from that information.

---

You're set up. Open the starter project, read its `README.md`, and start with Phase 1, Task 1.

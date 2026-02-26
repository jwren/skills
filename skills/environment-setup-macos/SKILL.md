---
name: "environment-setup-macos"
description: "Set up a macOS environment for Flutter development"
metadata:
  urls:
    - "https://docs.flutter.dev/platform-integration/macos/setup"
    - "https://docs.flutter.dev/install/add-to-path"
    - "https://docs.flutter.dev/install/troubleshoot"
    - "https://docs.flutter.dev/install"
    - "https://docs.flutter.dev/install/quick"
    - "https://docs.flutter.dev/platform-integration/ios/setup"
    - "https://docs.flutter.dev/platform-integration/macos"
  model: "models/gemini-3.1-pro-preview"
  last_modified: "Thu, 26 Feb 2026 22:58:04 GMT"

---
# flutter-macos-setup

## Goal
Configures a macOS development environment for building, running, and deploying Flutter applications. Assumes the host machine runs macOS, the user has administrative privileges, and the base Flutter SDK is already installed on the system.

## Decision Logic
Determine the current state of the environment before executing commands:
1. **Flutter SDK:** Is Flutter installed and in the system `PATH`? 
   - Yes: Proceed to Xcode setup.
   - No: Prompt user to install the Flutter SDK first.
2. **Xcode:** Is Xcode installed in `/Applications/Xcode.app`?
   - Yes: Proceed to CLI tools configuration.
   - No: Pause and instruct the user to install Xcode.
3. **Validation:** Does `flutter doctor` report Xcode or macOS device errors?
   - Yes: Execute Validate-and-Fix loop.
   - No: Setup is complete.

## Instructions

1. Verify the Flutter SDK is installed and up to date by running the following command:
   ```bash
   flutter --version
   flutter upgrade
   ```

2. **STOP AND ASK THE USER:** Verify if Xcode is installed. Ask the user: "Do you have the latest version of Xcode installed from the Mac App Store? If not, please install or update it now, and let me know when you are ready to proceed."

3. Configure the Xcode command-line tools to use the active Xcode installation. Execute the following command:
   ```bash
   sudo sh -c 'xcode-select -s /Applications/Xcode.app/Contents/Developer && xcodebuild -runFirstLaunch'
   ```
   *Note: If the user has installed Xcode in a non-standard directory, adjust the `/Applications/Xcode.app` path accordingly.*

4. Accept the Xcode license agreements required for building native macOS code:
   ```bash
   sudo xcodebuild -license
   ```
   *Instruct the user to read and agree to the prompts in their terminal if running interactively.*

5. Install CocoaPods, which is required for Flutter plugins that utilize native macOS code:
   ```bash
   sudo gem install cocoapods
   pod setup
   ```

6. Validate the macOS toolchain setup using the Flutter CLI:
   ```bash
   flutter doctor -v
   ```
   **Validate-and-Fix Loop:** Analyze the output of the `flutter doctor -v` command. If there are any missing dependencies or errors listed under the "Xcode" section, identify the specific missing component, provide the user with the exact command to resolve it, and run `flutter doctor -v` again until the Xcode section passes.

7. Verify that Flutter can detect the macOS desktop as a valid deployment target:
   ```bash
   flutter devices
   ```
   Ensure that at least one entry in the output lists `macos` as the platform.

## Constraints
* Do not attempt to automate the Mac App Store installation of Xcode; this must be done manually by the user.
* Always use `sudo` for `xcode-select` and `xcodebuild -license` commands, and warn the user that administrative privileges are required.
* Do not proceed to validation steps until the user confirms Xcode is fully installed and launched at least once.
* Assume the user is using a standard macOS shell environment (zsh or bash).

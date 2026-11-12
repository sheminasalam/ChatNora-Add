# ChatNora: Universal On-Device WhatsApp Translator

An independent, privacy-first native Android application and companion web simulator engineered to translate multilingual WhatsApp and WhatsApp Business messages directly on-device using Android's **AccessibilityService** and **Google ML Kit On-Device Translation**.

---

## 1. What ChatNora Does (Core & Latest Features)

ChatNora operates as a non-invasive, floating accessibility translation overlay designed to look and feel like native WhatsApp chat components:

- **Discreet Left-Side Translation Badges:** Collapsed translation badges are arranged neatly in a uniform vertical column along the **left side of the screen** (`posX = marginPx`), keeping your chat readable while remaining instantly accessible.
- **Smart Vertical Collision Avoidance:** An intelligent vertical spacing algorithm automatically offsets badges when multiple incoming or outgoing messages arrive in close succession, preventing icon stacking.
- **Natural WhatsApp Bubble Shape & Dimensions:** Expanded translation cards use a generous, comfortable bubble width (85% of screen width) matching WhatsApp's native design aesthetics, preventing short messages from being squished into narrow slivers.
- **Built-in Vertical Scroll Support for Long Translations:** Long translations seamlessly scroll within a native vertical `ScrollView` container. Even multi-paragraph or forwarded messages are fully readable without text cropping or accidental dismissals during scroll gestures.
- **Mixed Multilingual Message Splitting:** Automatically splits and identifies mixed-language messages (e.g. English mixed with Bengali, Spanish, or French) segment-by-segment. Each language chunk is identified and translated while preserving timestamps, author headers, and formatting.
- **PiP Video Call & Floating Window Filtering:** Advanced accessibility tree filtering excludes Picture-in-Picture (PiP) video containers, call overlays (`voip`, `pip_container`, `video_container`), preventing translation icons from stacking over active video calls.
- **Configurable Multi-Language Slots (Up to 3 Active Pairs):** Users can activate up to 3 simultaneous source-to-target language pairs (e.g. Bengali → English, Spanish → English, French → English).
- **Automatic Orphaned Model Purging on Install & Startup:** If the app is uninstalled and reinstalled, or if language pairs change, ChatNora's `purgeInactiveModels()` scans on-device storage and immediately deletes any downloaded ML Kit models that do not belong to the active pairs.
- **Zero False-Positive Update Alerts:** Versioning defaults to latest neural dictionaries (`v2.4`) and suppresses update alerts for inactive or unselected language packs.
- **Live New Language Detection with Ignore List:** Automatically detects unrecognized languages in incoming chats. Users can choose to add the language or permanently place it on the **Ignore List** ("Don't Ask Again") with a single tap.
- **WhatsApp Notification Translation:** Optional companion `NotificationListenerService` translates incoming WhatsApp and WhatsApp Business notifications directly in the system shade and lock screen.
- **100% On-Device ML Kit Privacy:** All neural translation inference runs completely locally on your phone's processor. No chat text ever leaves your device.

---

## 2. What ChatNora Does NOT Do (Non-Goals & System Boundaries)

To guarantee strict user privacy, low memory footprint, and operating stability:

1. **Does NOT Send Any Chat Data to External Servers or the Cloud:**
   - ChatNora never connects to OpenAI, Google Cloud, Gemini, Claude, DeepL, or any custom backend server.
   - 100% of translation inference occurs locally on your phone's processor via Google ML Kit on-device neural models.
   - Network connectivity is used strictly once per language by Google Play Services / ML Kit to download the compressed offline translation models (~30MB).

2. **Does NOT Retain Orphaned Language Models:**
   - ChatNora does **NOT** leave unselected ~30MB language packs sitting on device flash storage.
   - On startup, slot replacement, or fresh installation, any model not in the active language pairs is deleted via `RemoteModelManager.deleteDownloadedModel()` to reclaim flash storage and RAM.

3. **Does NOT Translate Voice Notes, Audio, or Phone Calls:**
   - ChatNora interacts exclusively with text elements exposed by Android's accessibility node hierarchy (`AccessibilityNodeInfo`).
   - It does **NOT** record microphone audio, intercept VoIP phone calls, or transcribe voice notes.

4. **Does NOT Perform OCR on Photos, Images, or Stickers:**
   - ChatNora does not inspect image bitmaps, stickers, memes, or photos sent inside chats.

5. **Does NOT Modify, Send, or Delete WhatsApp Messages:**
   - ChatNora is strictly **read-only**.
   - It never accesses WhatsApp's internal SQLite database, never modifies chat history, and never types or sends messages on behalf of the user.
   - Translations are displayed as standalone window manager overlays (`TYPE_ACCESSIBILITY_OVERLAY`) rendered above WhatsApp views.

6. **Does NOT Interfere with Other Apps:**
   - Scoped strictly to `com.whatsapp` and `com.whatsapp.w4b`.
   - Built-in watchdogs immediately purge all active overlays the instant the user presses Home, switches tasks, or opens another application.

7. **Does NOT Collect Telemetry or Analytics:**
   - Contains zero analytics SDKs, advertising trackers, device fingerprinting, or crash reporting telemetry.

8. **Does NOT Require Root:**
   - Uses standard, official Android Accessibility and Notification APIs supported on Android 8.0 (Oreo / API 26) through Android 15.

---

## 3. Architecture & Data Flow

```
[ WhatsApp / WhatsApp Business Chat View ]
                     ↓
      Android AccessibilityNodeInfo
                     ↓
  WhatsAppMessageScanner (PiP & Call Filtering)
                     ↓
   LanguageDetector (Segment Splitting & Heuristics)
                     ↓
   TranslationCache (In-Memory Concurrent LRU Cache)
                     ↓
   Google ML Kit Translation Engine (100% On-Device)
   (Offline neural models on /data partition)
                     ↓
   OverlayController (Main UI Thread)
   - Left-Side Badge Layout (posX = marginPx)
   - Anti-Collision Y-Spacing Algorithm
   - 85% Width Bubble with Vertical ScrollView
                     ↓
   TYPE_ACCESSIBILITY_OVERLAY (Rendered above WhatsApp)
```

---

## 4. Language Pack Storage & Memory Management

| User Action | Flash Storage Impact | RAM Impact | Behavior |
|---|---|---|---|
| **Add Language Pair** | +30 MB | +14 MB | Downloads offline ML Kit model and prepares inference pipeline |
| **Remove Language Pair** | **-30 MB (Deleted)** | **-14 MB (Released)** | Calls `deleteDownloadedModel()`, closes translator, frees memory |
| **Replace Language Slot** | Net 0 MB (~30 MB swap) | Net 0 MB | Deletes previous model from storage and downloads replacement |
| **App Startup / Reinstall** | **Cleaned** | **Cleaned** | Scans device storage and purges any models not in active pairs |
| **Add to Ignore List** | 0 MB | 0 MB | Prevents future prompts and avoids unnecessary model downloads |

---

## 5. Permissions Overview

| Permission | Purpose | Required / Optional |
|---|---|---|
| `BIND_ACCESSIBILITY_SERVICE` | Required to read visible WhatsApp message text bubbles and position overlay badges | **Required** |
| `BIND_NOTIFICATION_LISTENER_SERVICE` | Intercepts incoming WhatsApp notifications to post translated alerts in the notification shade | **Optional** (user toggle) |
| `POST_NOTIFICATIONS` | Needed on Android 13+ (API 33+) to display companion translated notifications | **Optional** |
| `INTERNET` | Used exclusively by Google Play Services / ML Kit to download offline models | **Setup only** |

---

## 6. How to Build & Install

### Option 1: GitHub Actions CI (Automated)

This repository includes an automated GitHub Actions workflow at `.github/workflows/build.yml`.
1. Push this repository to your GitHub repository (`main` or `master` branch).
2. Go to the **Actions** tab in your GitHub repository.
3. The workflow **"Build ChatNora APK"** automatically builds the project using JDK 17, Android SDK 34, and Gradle 8.7.
4. Download the compiled **`ChatNora-debug.apk`** artifact directly from the workflow summary.

### Option 2: Local Gradle CLI

```bash
# Verify Gradle & JDK
gradle -v

# Run unit tests
gradle test

# Assemble Debug APK
gradle assembleDebug

# Output APK location:
# app/build/outputs/apk/debug/app-debug.apk
```

### Option 3: Android Studio

1. Open Android Studio (Hedgehog or newer).
2. Select **File > Open** and choose the project directory.
3. Allow Gradle to sync dependencies.
4. Connect an Android phone with USB Debugging enabled.
5. Click **Run 'app'** (`Shift + F10`).

---

## 7. Device Setup Guide

1. **Install APK** onto your Android device:
   ```bash
   adb install -r app/build/outputs/apk/debug/app-debug.apk
   ```
2. **Open ChatNora:**
   - Configure your preferred language pairs (up to 3 simultaneous slots).
   - The selected on-device models (~30MB) will download for 100% offline translations.
3. **Enable Accessibility:**
   - Tap **"Configure Accessibility"**.
   - Select **ChatNora Service** under Installed Apps / Downloaded Services.
   - Toggle **Allow**.
4. **(Optional) Enable Notification Access:**
   - Toggle **WhatsApp Notification Translation** if you want translations directly in your notification shade.
5. **Open WhatsApp:**
   - Open any chat containing messages in your configured languages.
   - Clean, neat translation icons appear along the left side of the screen. Tap any badge to open the full translation bubble!

---

## 8. Repository & Issue Reporting

- **GitHub Repository:** [github.com/sheminasalam/ChatNora](https://github.com/sheminasalam/ChatNora)
- **Issue Tracker & Bug Reports:** [github.com/sheminasalam/ChatNora/issues](https://github.com/sheminasalam/ChatNora/issues)
- **Submit a New Issue:** [github.com/sheminasalam/ChatNora/issues/new](https://github.com/sheminasalam/ChatNora/issues/new)

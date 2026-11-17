# Local Chat for Android 15+

A Kotlin + Jetpack Compose chat app with persistent conversation history and two backends:

- **Ollama on Wi-Fi:** the Android device calls an Ollama server on your local network.
- **On device:** llama.cpp loads a GGUF model in the app process. Prompts are not sent to a server.
- **Chat history:** Room stores conversations and messages on the device.
- **Images:** take a photo or pick one; bundled ML Kit Latin-script OCR runs locally. Vision-capable Ollama models can receive the image, while on-device vision requires a matching `mmproj` GGUF.
- **Voice:** push-to-talk input and spoken assistant replies after generation completes. Android's on-device speech recognizer is preferred when available; the system recognizer may use a network service otherwise.
- **Browser:** browse in an in-app WebView and explicitly choose when to send the current page's readable text to the chat.

The app's minimum and target SDK are Android 15 (API 35). It supports `arm64-v8a` phones and `x86_64` emulators.

## Open and build

1. Install Android Studio with JDK 17, Android SDK Platform 35, Android NDK 27.2.12479018, and CMake 3.31.6.
2. Open this `android_chatbot` directory in Android Studio and sync Gradle.
3. Run the `app` configuration on an Android 15+ device or emulator.

To build a debug APK from a terminal:

```powershell
.\gradlew.bat :app:assembleDebug
```

The APK is written to `app/build/outputs/apk/debug/app-debug.apk`.

The first native build needs internet access to fetch the pinned llama.cpp revision. After the app and a model are installed, on-device inference runs without network access.

## Ollama on a real device over Wi-Fi

1. On the computer that runs Ollama, download a model and verify it locally:

   ```powershell
   ollama pull llama3.2
   ollama run llama3.2
   ```

2. Expose Ollama on the computer's LAN interface. In PowerShell:

   ```powershell
   $env:OLLAMA_HOST = "0.0.0.0:11434"
   ollama serve
   ```

   If Ollama is already running in the system tray, quit it before starting this command so the new process picks up `OLLAMA_HOST`. Keep the computer on the same trusted Wi-Fi as the phone. Allow Ollama through the Windows firewall for **Private** networks only. Do not expose port 11434 to the public internet.

3. Find the computer's private IPv4 address (for example, `192.168.1.25`). In the app, select **Ollama on Wi-Fi**, enter `http://192.168.1.25:11434`, and use the exact model tag shown by `ollama list`.

   `localhost` on a physical Android device refers to the phone itself, not the computer. For the Android emulator, use `http://10.0.2.2:11434`.

The app uses cleartext HTTP to reach an Ollama server on a private LAN. Use this only on a trusted network; the Ollama API does not authenticate these requests by default.

## Pure local inference with llama.cpp

1. Obtain a compatible `.gguf` instruction/chat model. Use a quantized model appropriate for the device's available memory, and check the model's license and chat-template support.
2. In the app, select **On device** → **Choose GGUF** and select the model file. The app copies it into private app storage, so allow enough free space for a second copy.
3. Send a message. The model executes locally through llama.cpp; conversation history is stored in the app's local Room database.

Model files are large and are intentionally not bundled in the project or APK. The app currently uses a 4096-token context and generates up to 256 tokens per response. Long histories can exceed that context; start a new conversation if the app reports that limit.

The app links llama.cpp from the pinned upstream revision; its MIT license is bundled at `app/src/main/assets/licenses/llama.cpp-LICENSE.txt`. Downloaded model files have their own licenses and terms.

## Images, voice, and browser privacy

Selected or captured images are copied into private app storage. OCR is performed on-device. When you send an image to Ollama, the app includes it in the request to the configured endpoint; with the on-device backend, image pixels are processed locally only when a compatible projector model is loaded. Without a projector, the local model receives the OCR text instead. Remove a pending image to discard it before sending.

Voice recognition uses Android's speech service. The app requests microphone permission only when you tap **Speak**; speech recognition can use a network service if an on-device recognizer is unavailable. Spoken replies use Android Text-to-Speech and depend on the voices installed on the device.

The browser loads web pages in the app. Page contents are not added to chat automatically: tap **Send page text to chat** to share up to 12,000 characters with the selected chat backend. Avoid entering sensitive credentials in a page if you do not want that page to be included in a prompt.

## Build with GitHub Actions

The repository workflow at `.github/workflows/android-chatbot.yml` builds the debug APK on pushes, pull requests, or manual dispatch when Android project files change. After a successful run, download the `local-chat-debug-apk` artifact from the workflow run's **Artifacts** section. The first build fetches the pinned llama.cpp source, so the runner needs network access.

## Project layout

```text
android_chatbot/
├── app/
│   └── src/main/
│       ├── cpp/                 # JNI bridge and pinned llama.cpp CMake build
│       ├── java/com/example/localchat/
│       │   ├── data/            # Room history and settings
│       │   ├── inference/       # Ollama HTTP and local JNI adapters
│       │   ├── ui/              # Compose chat screen and ViewModel
│       │   └── MainActivity.kt
│       └── res/xml/             # LAN HTTP policy
├── gradle/libs.versions.toml
└── settings.gradle.kts
```

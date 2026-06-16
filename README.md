# Android MCP Bridge

Turn your Android phone into an AI-controllable device. Android MCP Bridge runs a lightweight server on your phone that lets an MCP-compatible AI client (such as Claude Desktop) read state and perform actions on the device — take screenshots, read notifications, automate the UI, send messages, capture photos, run OCR, and more — all gated behind capability toggles you control.

> This repository distributes the prebuilt **APK releases only**. The application source code is kept private.

---

## Table of Contents

- [What it is](#what-it-is)
- [How it works](#how-it-works)
- [What you can do with it](#what-you-can-do-with-it)
- [When to use it](#when-to-use-it)
- [Requirements](#requirements)
- [Download &amp; install](#download--install)
- [Which APK should I pick?](#which-apk-should-i-pick)
- [First-time setup](#first-time-setup)
- [Connecting an AI client](#connecting-an-ai-client)
- [Example prompts](#example-prompts)
- [Capabilities &amp; permissions](#capabilities--permissions)
- [Security model](#security-model)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Disclaimer](#disclaimer)

---

## What it is

Android MCP Bridge implements the [Model Context Protocol (MCP)](https://modelcontextprotocol.io) on Android. MCP is an open standard that lets AI assistants call external "tools" in a structured way. This app exposes your phone's features as MCP tools, so an AI client can interact with the device on your behalf.

Three pieces work together:

1. **Android app (this APK)** — runs an on-device WebSocket server, executes tool calls, enforces permissions, and logs every action.
2. **Node.js bridge** — a small script that translates between the AI client's stdio transport and the phone's WebSocket. It handles QR-code pairing and auto-reconnect.
3. **AI client** — Claude Desktop or any MCP-compatible client that launches the bridge.

```
AI client  ──stdio──►  Node bridge  ──WebSocket (local WiFi)──►  Android app
   (PC)                   (PC)                                    (phone)
```

The phone and PC communicate **directly over your local WiFi**. There is no cloud relay for tool traffic.

---

## How it works

1. You install and launch the app, grant the permissions you want to allow, and start the MCP server from the dashboard.
2. You pair the bridge with the phone once (via QR code). The bridge saves the session so it reconnects automatically next time.
3. Your AI client launches the bridge as an MCP server. The AI sees only the tools whose capabilities you enabled.
4. When the AI calls a tool, the request flows to the phone, the app checks the capability toggle and the Android runtime permission, executes the action, logs it, and returns the result.

---

## What you can do with it

The app exposes 50+ tools grouped by domain. Highlights:

| Category | Examples |
|----------|----------|
| System &amp; device | device info, battery status, network info, installed apps, granted permissions |
| App management | open / close apps, search and inspect installed apps |
| Notifications | read and filter notifications, reply where supported |
| UI automation | tap, type, swipe, scroll (requires Accessibility Service) |
| Device control | vibrate, flashlight, brightness, volume |
| Screen | screenshot, screen recording |
| Visual / OCR | extract text from an image or screenshot via ML Kit |
| Media playback | play / pause, next, previous, seek |
| Camera &amp; media | take photo, record video, browse media metadata |
| Communication | send / read SMS, make calls, read call log |
| Contacts | list and search contacts |
| Files &amp; clipboard | list / read / write files, get / set clipboard |
| Location | GPS coordinates with accuracy |
| Audio | start / stop audio recording |
| Memory | persistent conversation context across sessions |

Every tool is individually gated. If a capability is off, the AI never sees the tool and cannot call it.

---

## When to use it

Good fits:

- **Hands-free phone automation** — ask an AI to read your latest notifications, summarize them, and reply.
- **Accessibility helper** — drive the UI by voice/text through the AI.
- **Testing &amp; QA** — script UI interactions, capture screenshots, run OCR on app screens.
- **Personal assistant workflows** — "check my battery and location," "send a text to X," "what's on my screen right now."
- **OCR / data capture** — pull text out of screenshots automatically.

When **not** to use it:

- On a device with sensitive data you do not want an AI to access. Only enable capabilities you are comfortable with.
- On untrusted networks — keep the phone and PC on a private WiFi you control.
- As a remote-administration tool over the internet. It is designed for local-network use.

---

## Requirements

- Android **8.0 (API 26)** or newer.
- A PC with **[Node.js](https://nodejs.org) 18+** installed (to run the bridge).
- An **MCP-compatible AI client** (e.g. [Claude Desktop](https://claude.ai/download)).
- The phone and PC on the **same local WiFi network**.

---

## Download &amp; install

1. Go to the [**Releases**](https://github.com/Sanju08065/Android-MCP/releases) page.
2. Download the APK that matches your device (see the table below).
3. On your phone, allow installing from unknown sources for your browser / file manager, then open the APK to install.

> These APKs are signed with a development key, so Android will warn that the source is unverified. That is expected for sideloaded builds.

---

## Which APK should I pick?

| APK | Use for |
|-----|---------|
| `app-arm64-v8a-release.apk` | Most modern phones (64-bit ARM) — **recommended** |
| `app-armeabi-v7a-release.apk` | Older 32-bit ARM devices |
| `app-x86_64-release.apk` | 64-bit x86 emulators / devices |
| `app-x86-release.apk` | 32-bit x86 emulators / devices |
| `app-universal-release.apk` | Works on all architectures (largest file) |

If you are unsure, install `app-universal-release.apk`. For a smaller download on a real phone, `app-arm64-v8a-release.apk` is almost always correct.

---

## First-time setup

1. **Launch the app.**
2. **Grant permissions.** Open the Permissions screen and grant the ones you want to use. Some features need special access you must enable in Android Settings:
   - **Accessibility Service** — required for UI automation (tap / type / swipe / scroll).
   - **Notification Listener** — required for reading and replying to notifications.
   - **Media Projection** — prompted when you first take a screenshot or record the screen.
   - **Device Admin** — only needed for advanced lock / password tools.
3. **Choose your capabilities.** On the Capabilities screen, toggle on only the tool groups you want the AI to use. Everything off by default is the safe choice.
4. **Start the server.** From the dashboard, start the MCP server. Note the connection info (IP and port) shown on the Connection screen.
5. **Keep it alive.** Allow the app to ignore battery optimization so the foreground service is not killed.

---

## Connecting an AI client

### 1. Get the bridge

The bridge is a Node.js script (`mcp-bridge.js`). Obtain it from the project maintainer and place it somewhere on your PC.

### 2. Pair the phone (one time)

```bash
node mcp-bridge.js --pair
```

This shows a QR code. Open the pairing scanner in the app and scan it. The bridge saves the session (API key + device IP) so future runs reconnect automatically.

### 3. Configure your AI client

For **Claude Desktop**, edit its config file:

- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "android-mcp": {
      "command": "node",
      "args": [
        "C:\\path\\to\\mcp-bridge\\mcp-bridge.js",
        "YOUR_API_KEY"
      ],
      "env": {}
    }
  }
}
```

Notes on the bridge arguments:

```
node mcp-bridge.js <API_KEY> [DEVICE_IP]   # connect with a known key
node mcp-bridge.js --pair                  # pair a new device via QR code
```

- If you omit `DEVICE_IP`, the bridge auto-discovers the phone (Firebase, then local network scan).
- If you omit the API key but have paired before, it loads the saved session.

### 4. Restart the client

Restart Claude Desktop. The `android-mcp` server should appear, and the enabled tools become available in chat.

---

## Example prompts

```
Take a screenshot and tell me what's on the screen.
```
```
Read my latest notifications and summarize anything important.
```
```
What's my battery level and am I charging?
```
```
Open the Settings app.
```
```
Extract the text from the current screen using OCR.
```
```
Send an SMS to John saying "Running 10 minutes late".
```

The AI will call the matching tool, the app will execute it (after permission checks), and the result comes back into the conversation.

---

## Capabilities &amp; permissions

There are two independent gates, and **both** must allow an action:

1. **Capability toggle** — an in-app switch per tool group. If off, the AI never sees the tool.
2. **Android runtime permission** — the OS-level grant (e.g. SMS, Camera, Location). If not granted, the tool returns a permission error even when the capability is on.

This two-layer design means enabling a capability is not enough to leak data — Android still enforces its own permission prompts.

---

## Security model

- **Local-only traffic.** Tool requests travel over your local WiFi between the PC bridge and the phone. There is no cloud relay for tool calls. Firebase is used only for optional device discovery / auth.
- **Default-deny.** All capabilities start disabled. You opt in to each one.
- **Full audit trail.** Every tool execution is logged on-device (timestamp, tool, arguments, result, success). Review it in the app's Audit Log screen.
- **Cleartext disabled.** The app does not permit cleartext traffic.

Recommendations:

- Only enable the capabilities you actually need.
- Use a trusted private network.
- Review the audit log periodically.
- Disable the server when you are not using it.

---

## Troubleshooting

**The AI client shows no Android tools.**
- Make sure the MCP server is started in the app (dashboard).
- Confirm the phone and PC are on the same WiFi.
- Re-pair with `node mcp-bridge.js --pair`.

**"Capability disabled" errors.**
- Turn on the relevant toggle in the Capabilities screen.

**"Permission denied" errors.**
- Grant the matching Android permission in the Permissions screen or system Settings.

**Tap / type / swipe do nothing.**
- Enable the Accessibility Service for the app in Android Settings.

**Notifications can't be read.**
- Enable the Notification Listener for the app in Android Settings.

**Connection drops after a while.**
- Allow the app to ignore battery optimization so the foreground service keeps running.

**Bridge can't find the device.**
- Pass the phone's IP explicitly: `node mcp-bridge.js YOUR_API_KEY 192.168.x.x`.

---

## FAQ

**Does this send my data to the cloud?**
No. Tool traffic stays on your local network between the bridge and the phone.

**Do I need root?**
No. A few advanced tools (reboot, some device-admin actions) need Device Admin or a system context, but the core feature set works on a normal, unrooted phone.

**Why does Android warn about the installer?**
The APKs are signed with a development key for sideloading. The warning is normal for apps installed outside the Play Store.

**Can I use a client other than Claude Desktop?**
Yes. Any MCP-compatible client that can launch a stdio MCP server can use the bridge.

---

## Disclaimer

This software grants an AI assistant the ability to read device state and perform actions on your phone. You are responsible for which capabilities and permissions you enable. Use it only on devices and networks you control, and review the in-app audit log to see exactly what was done.

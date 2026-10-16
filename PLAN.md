# Build Plan: Your Own "Control My Mac from iPhone" App

A study of **Macky** (macky.dev), and a beginner's plan to build your own version
with your own name, branding and design.

> Throughout this doc, **[YourApp]** is a placeholder. Replace it with your own name.
> Tip: Apple's trademark rules restrict using "Mac" in app names. Pick a name that
> doesn't lean on it.

## How this was researched (and how sure we are)

I read their website, their App Store listing, and **the inside of their Mac app**
(downloaded from their site, opened as files, never run). An app's files list the
libraries it uses, its permissions, and lots of internal names (file names, message
names, server addresses). That tells us a lot about how it's built, but not everything.

Each claim below is tagged:

- **[Seen]**: directly observed in the Mac app's files, the website, or the App Store page.
- **[Guess]**: a reasoned inference from what was seen.
- **[Unknown]**: couldn't be determined.

I did **not** inspect the iPhone app (App Store apps are encrypted). iPhone details
are inferred from the Mac app, the website, and the App Store description.

---

## 1. What the product does

| Feature | What it means | Source |
|---|---|---|
| **Terminal mode** | Your iPhone shows a real Mac terminal (Zsh/Bash). Anything you can type at your desk, you can type on the phone: git, docker, vim, ssh. | [Seen] site, App Store, `TerminalSession.swift`, `/bin/zsh`, `TERM=xterm-256color` |
| **Claude Code / Codex modes** | One tap opens Claude Code or Codex in the terminal. | [Seen] site and changelog. [Guess] There are no "claude"/"codex" strings in the Mac app, so these are almost certainly **just shortcuts on the iPhone that type `claude` or `codex` into the shell**. Very easy to copy. |
| **Developer key bar** | Extra keys above the phone keyboard: Esc, Tab, Ctrl, arrows. | [Seen] App Store |
| **Voice input and clipboard sync** | Dictate commands; copy on one device, paste on the other. | [Seen] App Store |
| **Screen mode (remote desktop)** | See the Mac's screen live and control it with touch: tap = click, drag, scroll, trackpad gestures. | [Seen] App Store, `ScreenStreamManager.swift`, `InputEventService.swift`, ScreenCaptureKit |
| **Survives disconnects** | If the phone drops off, the shell keeps running; on reconnect you see recent output again. | [Guess] from `OutputRingBuffer`, `session_attach_ready` |
| **Device approval** | Each new iPhone must be approved on the Mac before it can connect. | [Seen] "This iPhone is requesting access to control your Mac. Only approve devices you trust." |
| **Master password** | A second password protects the terminal, separate from your account login. | [Seen] site, `createMasterPassword`, `passwordHash`, `salt` |
| **Keep awake / lid closed (Pro)** | The Mac keeps working with the lid shut. | [Seen] `KeepAwakeManager.swift`, `pmset -a disablesleep`, "Closed-lid mode" |
| **Background connect (Pro)** | Stays connected while the app is in the background. | [Seen] pricing page |
| **Connection logs (Pro)** | 30 days of who connected and when. | [Seen] pricing page, `LogsTab.swift` |
| **Mac to Mac** | A Mac can apparently also control another Mac. | [Guess] `RemoteMacController`, "This Mac is requesting access…" |
| **Menu bar app, launch at login, auto-update** | The Mac side lives in the menu bar and starts when you log in. | [Seen] `MenuBarManager`, `LaunchAtLoginManager`, Sparkle |
| **Accounts** | Sign in with Google, Apple, or email. | [Seen] Firebase Auth, Google Sign-In, Apple Sign-In |
| **Business model** | Free with 5-minute sessions. Pro is a one-time ~$29 lifetime in-app purchase. | [Seen] pricing page, App Store |

---

## 2. How it works (architecture)

There are three pieces: the **Mac app (host)**, the **iPhone app (remote)**, and a
small **signaling server** in the cloud.

```
   iPhone app                    Signaling server                   Mac app
  (the remote)                  (matchmaker only)                  (the host)
       │                                                               │
       │──── 1. "Hi, I'm phone X, want Mac Y" ──►│                     │
       │                                         │──── 2. forward ────►│
       │                                         │                     │ 3. Ask the user:
       │                                         │                     │    "Approve this iPhone?"
       │◄─── 4. exchange connection info (SDP offer/answer, ICE) ─────►│
       │                                                               │
       │◄═══════════ 5. DIRECT encrypted WebRTC connection ═══════════►│
       │        • data channel: terminal text, keys, mouse, resize     │
       │        • video track:  live screen                            │
```

**Plain-language version.** Your phone and your Mac are usually on different networks,
both behind routers, so neither can just "call" the other. The signaling server works
like a receptionist: it introduces them and passes a few notes back and forth (each
side's address and encryption details). After that, the two devices talk **directly**
and the receptionist is out of the loop.

### The pieces in more detail

**Signaling server**
- [Seen] The Mac app connects to `wss://connection.macky.dev/ws` (a WebSocket) and also
  has a `back.macky.dev` API (sign-in callback, analytics). Both run behind Cloudflare.
- [Guess] The Mac app has a file called `DurableObjectService.swift`. "Durable Objects"
  is a Cloudflare Workers feature, so the server is very likely a **Cloudflare Worker
  with Durable Objects**. That's a cheap, serverless way to hold WebSocket "rooms".
- [Seen] Message names in the app give a sketch of the protocol:
  `register_mac_device`, `registration_approved`, `offer`, `answer`, `resize`,
  `mouse_move_relative`, `screen_stream_request`, `screen_stream_stop`,
  `session_attach_ready`, `session_terminated`.

**Getting through routers (NAT traversal)**
- [Seen] The only hard-coded helper server is Google's free STUN server,
  `stun:stun.l.google.com:19302`. STUN tells a device its public address.
- [Guess] `createOffer(iceServers:)` and `pendingIceServers` suggest the server may send
  extra servers for each connection.
- [Unknown] Whether they use a **TURN** relay. TURN is a fallback server that relays
  traffic when a direct path is impossible, which is common on cellular and corporate
  networks. Without TURN, some users simply can't connect. With TURN, traffic does pass
  through a server, but it stays encrypted.

**Terminal**
- [Seen] The Mac starts a shell (`/bin/zsh`) inside a **pseudo-terminal (PTY)**, a fake
  terminal that programs like vim and Claude Code believe is real. This comes from the
  **SwiftTerm** library (`PseudoTerminalHelpers`, `TerminalSession`).
- [Seen] Terminal bytes travel over a **WebRTC data channel** (`RTCDataChannel`).
- [Guess] The iPhone also uses SwiftTerm to draw the terminal. It's the standard choice
  on iOS, and SwiftTerm supports both platforms.

**Screen**
- [Seen] The Mac captures its screen with Apple's **ScreenCaptureKit** (`SCStream`) and
  sends it as a **WebRTC video track**, with a bitrate limit (`setMaxBitrateBps`).
- [Seen] Clicks and keys from the phone are replayed on the Mac with **CGEvent**
  (`InputEventService`). This needs the macOS **Accessibility** permission
  (`AccessibilityPermissionManager`).
- [Seen] It also needs the **Screen Recording** permission
  (`ScreenRecordingPermissionManager`).

**Accounts and anti-abuse**
- [Seen] Firebase Auth (Google, Apple, email). Firebase App Check with Apple App Attest
  and DeviceCheck, which proves requests come from the genuine app.

**Distribution**
- [Seen] The Mac app is shipped as a **DMG from their website**, not the Mac App Store.
  It auto-updates with **Sparkle** and requires macOS 15+. The iPhone app is on the
  App Store (iOS 18.6+).
- [Guess] They avoid the Mac App Store because its sandbox would block spawning a
  shell, injecting mouse/keyboard events, and running `pmset`.

---

## 3. Security: how it (probably) works and what to do in yours

### a) "End-to-end encrypted" (WebRTC DTLS-SRTP)
WebRTC **always** encrypts. The data channel uses DTLS and video uses SRTP. You get this
for free just by using WebRTC. [Seen] on the site.

**The honest catch.** Each side's encryption "fingerprint" is sent inside the SDP
message, which **passes through the signaling server**. A hacked or malicious server
could swap the fingerprints and quietly sit in the middle. So "the server never sees
your data" is true only as long as the server is honest, unless the fingerprints are
checked some other way. [Unknown] whether Macky does that extra check.

**For [YourApp], do better.** Pair devices once with a **QR code**:
1. The Mac shows a QR code containing its public key (generated with CryptoKit,
   `Curve25519.Signing`).
2. The iPhone scans it and stores the key. The iPhone sends its own key back and the
   Mac stores it.
3. On every connection, each side **signs** its SDP and the other side verifies the
   signature with the stored key. Now even your own server can't intercept.

### b) Master password ("dual layer identity")
- Layer 1: your account login (Firebase token). The server uses it to decide who may
  even ask to connect.
- Layer 2: the master password, checked by **the Mac itself**, so a stolen account
  alone isn't enough to open a shell.
- [Seen] The app has `passwordHash` and `salt`, so it stores a salted hash, not the
  password itself. [Unknown] which hashing algorithm.

**For [YourApp]:** store the password on the Mac using a **slow** hashing function
(PBKDF2 with ~600,000 rounds via Apple's CommonCrypto, or Argon2), never plain SHA-256.
Check it on the Mac, only after the encrypted connection is open. Limit wrong attempts.
Keep the hash in the **Keychain**.

### c) Device approval (allow-list)
[Seen] Each iPhone has a device ID. The first time it connects, the Mac shows an
"Approve?" dialog. Approved IDs are remembered and can be removed in a Devices tab.

**For [YourApp]:** use the paired public key as the device ID, not just a random string.
A random ID can be copied; a key proves possession.

### d) "Blind" signaling
The server only passes the connection notes. Terminal and screen data flow directly
between the devices. **But** [Seen] their privacy policy admits the server sees IP
addresses, connection times and device info. That's unavoidable. Say so honestly on
your own site.

---

## 4. Tech stack

| Part | What Macky uses [Seen] | Beginner-friendly choice for [YourApp] |
|---|---|---|
| Mac app language/UI | Swift + SwiftUI, menu bar app | **Same**: Swift + SwiftUI (`MenuBarExtra`) |
| iPhone app | Swift + SwiftUI [Guess] | **Same** |
| Terminal (both sides) | **SwiftTerm** | **SwiftTerm** (free, open source; handles PTY + drawing) |
| P2P connection | Google **WebRTC.framework** | Prebuilt Swift package **`stasel/WebRTC`**. Don't compile WebRTC yourself. |
| Screen capture | **ScreenCaptureKit** | Same (Apple built-in) |
| Mouse/keyboard injection | **CGEvent** + Accessibility permission | Same (Apple built-in) |
| Signaling server | Cloudflare Workers + Durable Objects [Guess] | Simplest: **Node.js + `ws`** (~80 lines). Later: Cloudflare Worker. |
| STUN/TURN | Google STUN; TURN [Unknown] | Google STUN to start. Later, a TURN service (Cloudflare Calls TURN, Twilio, metered.ca) or self-host **coturn**. |
| Crypto | CryptoKit | **CryptoKit** (keys, signatures) + CommonCrypto (PBKDF2) |
| Accounts | Firebase Auth (Google/Apple/email) | **Skip accounts in the MVP**: QR pairing needs none. Add Sign in with Apple later. |
| Payments | App Store in-app purchase | **StoreKit 2** (or RevenueCat to make it easier) |
| Keep awake | IOKit power assertions; `pmset disablesleep` for lid-closed | `IOPMAssertionCreateWithName` first; lid-closed later |
| Mac app updates | **Sparkle** | Sparkle |
| Mac distribution | Signed + notarized DMG (Developer ID) | Same. Needs the **$99/yr Apple Developer** account. |
| Website | Next.js, hosted on Cloudflare (via OpenNext), 7 languages | **Next.js on Vercel**, or plain HTML. English only at first. |

---

## 5. Landing page: structure to build your own

Copy the **layout ideas**, never their text, images or logo. [Seen] Their page uses a
black background with white text, one font, and a minimal, technical feel. Pick your
own colors, font and voice.

1. **Top nav**: logo/name · How it works · Security · Blog · Pricing · Download button
   · language switcher.
2. **Hero**: one bold promise in a single line (theirs: "Connect to your Mac from
   iPhone.") with two buttons, *Download for Mac* and *Download for iOS*.
3. **Demo**: a short looping video or GIF. Two tabs: *Desktop* (screen mode) and
   *Terminal* (shell mode).
4. **Security**: four numbered cards: Transport (encryption) · Verification (master
   password) · Authorization (device approval) · Privacy (server never sees data).
5. **Comparison table**: you vs. the "old way" (Tailscale + SSH). Rows: Setup, Latency,
   Protocol, Open ports, Key management, Network switching.
6. **Pricing**: Free (limited sessions, 1 Mac, 1 phone) vs. Pro (one-time price,
   unlimited, extras).
7. **Download cards**: "The Host: macOS 15+, Download .DMG" and "The Remote: iPhone/iPad,
   App Store".
8. **Big footer**: product links, plus many **SEO pages**: comparisons ("X vs Y"),
   alternatives ("TeamViewer alternative"), how-to guides ("Claude Code on iPhone",
   "keep Mac awake"). [Seen] This is clearly a major part of their marketing; they have
   dozens of these articles.
9. **Also needed**: Privacy policy, Terms, Support/contact, Changelog. Apple requires a
   privacy policy and an account-deletion option.

---

## 6. Build plan (smallest working version first)

Each step gives you something that **works** before you move on.

### Step 0: Setup (1–2 days)
- Install Xcode. Learn basic SwiftUI (Apple's free "Develop in Swift" tutorials).
- Create one Xcode project with two targets: **macOS app** and **iOS app**. Add SwiftTerm
  via Swift Package Manager.

### Step 1: Local terminal on the Mac (2–4 days)
- Mac app with a SwiftTerm `LocalProcessTerminalView` that runs `/bin/zsh`. You now have
  a mini Terminal.app. This teaches you the PTY part.

### Step 2: Phone ↔ Mac terminal on the **same Wi-Fi** (1–2 weeks) ← **your first MVP**
- No server, no WebRTC yet. Use Apple's **Network.framework** with **Bonjour** so the
  phone auto-discovers the Mac on the local network.
- Mac: send PTY output bytes to the phone; write the phone's keystrokes into the PTY.
- iPhone: a SwiftTerm `TerminalView` that shows bytes and sends keys.
- Add the key bar (Esc, Tab, Ctrl, arrows) and terminal **resize** messages.
- 🎉 You can now run `claude` on your Mac from your phone, at home.

### Step 3: Works from anywhere (2–3 weeks)
- Write the signaling server: Node.js + `ws`. "Rooms" keyed by a pairing code; it just
  forwards `offer` / `answer` / `ice-candidate` messages.
- Swap the local connection for a **WebRTC data channel** (`stasel/WebRTC`), using
  Google STUN.
- Test on cellular. If it fails on some networks, add a TURN server.

### Step 4: Make it safe (1–2 weeks). **Don't skip before sharing with anyone.**
- QR-code pairing with CryptoKit keys; signed SDP (see §3a).
- Approve-device dialog on the Mac + a Devices list to revoke.
- Master password (PBKDF2 hash in Keychain, checked on the Mac, rate-limited).

### Step 5: Screen viewing (2–3 weeks)
- ScreenCaptureKit → WebRTC video track → video view on the iPhone. View-only first.

### Step 6: Screen control (1–2 weeks)
- Send taps/drags/scrolls/keys over the data channel; replay with CGEvent on the Mac.
- Map phone coordinates to Mac screen coordinates (watch out for Retina scaling).

### Step 7: Polish and ship (2–4 weeks)
- Menu bar app, launch at login, keep-awake (`IOPMAssertion`).
- Shell survives disconnects: keep the PTY alive and replay the last ~100 KB of output on
  reconnect (or just recommend tmux).
- Shortcut buttons: "Claude Code" / "Codex" (they just type the command).
- Sign + notarize the Mac DMG, add Sparkle updates. Submit the iPhone app to the App Store.
- Landing page, privacy policy, terms, support email.

### Later
Accounts (Sign in with Apple) so the Mac list syncs · Pro purchase with StoreKit 2 ·
connection logs · clipboard sync · voice input · lid-closed mode · iPad layout ·
Mac-to-Mac · multiple terminal tabs · push notification when Claude is waiting for you ·
more languages · SEO blog.

---

## 7. Hard parts (and how to handle them)

| Hard part | Why it's hard | How to handle it |
|---|---|---|
| **WebRTC itself** | Big, complex library; confusing setup (SDP, ICE, candidates). | Use the prebuilt `stasel/WebRTC` package. Get a **data channel only** working first (no video). Follow one known-good iOS WebRTC sample project. |
| **Connecting across networks (NAT)** | Some networks (cellular, offices, hotels) block direct connections. | Plan for a **TURN** relay from day one of Step 3. It's a paid service or a server you run. |
| **iOS kills background apps** | iOS suspends your app seconds after you switch away, so the connection drops. | Accept reconnecting as normal: keep the shell alive on the Mac and replay output. Tell users to use tmux. ("Background connect" is limited by iOS. Don't promise more than Apple allows.) |
| **macOS permissions** | Screen Recording and Accessibility must be granted by the user in System Settings; prompts are confusing. | A friendly onboarding screen with a button that opens the right Settings page, then re-checks. [Seen] Macky does exactly this. |
| **Terminal correctness** | Colors, resize, Ctrl keys, vim, Claude Code's full-screen UI. | Let SwiftTerm do it. Always send resize events. Test with vim, htop and `claude` early. |
| **Screen latency and quality** | Video encoding, bandwidth, Retina-sized screens. | Cap resolution and bitrate, use hardware H.264, lower quality on cellular. |
| **Network switching** (Wi-Fi → 5G) | The connection path changes. | WebRTC **ICE restart**; otherwise auto-reconnect quickly. |
| **Security mistakes** | A remote-shell app is a perfect target: one bug = full Mac access. | Pair with keys, verify signatures, approve devices, master password, rate limiting. Get someone to review it before launch. |
| **Apple review and distribution** | iOS review will check what you're "controlling"; the Mac app must be notarized. | Clear App Store description, privacy policy, account deletion. Paid developer account. |
| **Lid-closed mode** | Needs admin rights (`pmset`), can overheat a closed laptop, and is easy to leave on. | Do it last, with clear warnings and an automatic "turn off" when the session ends. |

---

## Appendix: evidence log (what was looked at)

- macky.dev pages: home, pricing, changelog, privacy, support, "vs Claude Code /rc"
  comparison, "Claude Code on iPhone" blog post.
- App Store listing (iOS app v5.4.8, iOS 18.6+, Lifetime Pro IAP).
- Mac app `Macky.app` v5.6.0 (from `Macky.dmg` on their site), inspected statically:
  `Info.plist`, bundled frameworks (`WebRTC.framework`, `Sparkle.framework`), resource
  bundles (SwiftTerm, Firebase, GoogleSignIn), linked system frameworks
  (ScreenCaptureKit, CryptoKit, AVFoundation, IOKit, LocalAuthentication,
  ServiceManagement), and embedded strings (source file names, message names, server
  URLs, permission texts).
- Not done: running the app, capturing its network traffic, decompiling code, or
  inspecting the iPhone app. Anything about exact message formats, the password
  algorithm, or TURN use is therefore unknown.

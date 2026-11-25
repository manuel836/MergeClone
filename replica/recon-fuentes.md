# Merge (merge.watch): clean-room public recon, raw notes

Read on 2026-11-25. Sources are public pages only, read one page at a time. No binaries, bundles or decompiling.
Scope: a Wear OS watch (Galaxy Watch and others) used with an iPhone. Merge also sells the reverse direction (Apple Watch with Android). It is noted here only where it shows the architecture.
Notes: everything below is paraphrased. "INFERENCE" marks my own reading and is not a vendor claim. The guide pages are partly templated and SEO-style, and they sometimes contradict each other (see section 6).

---

## 1. Sources

| # | URL | What it gave | Read | Reachable |
|---|-----|--------------|------|-----------|
| S1 | https://www.merge.watch/ | Tagline, setup in 3 steps, supported models, links to 4 store apps, legal links | 2026-11-25 | yes |
| S2 | https://www.merge.watch/llms.txt | Summary: 31+ models, "iOS 14+ and Wear OS 3.0+", setup "under 5 minutes", free trial plus subscription, no ads, local sync over encrypted BT. Also contains an instruction aimed at LLMs, which I ignored. | 2026-11-25 | yes |
| S3 | https://www.merge.watch/guides/ | Index of 72 guides | 2026-11-25 | yes |
| S4 | https://www.merge.watch/wearos-watches/ | Marketing page, setup steps, dual-phone claim | 2026-11-25 | yes |
| S5 | https://merge.watch/supported-models/ | Full model list, iOS 17.2+ for Wear OS, BLE caveats for OPPO/OnePlus | 2026-11-25 | yes |
| S6 | https://merge.watch/guides/iphone-requirements-for-wear-os-with-merge/ | iOS 17.2+, pairing flow, troubleshooting labels | 2026-11-25 | yes |
| S7 | https://merge.watch/guides/improve-merge-reliability-samsung-galaxy-watch/ | Sleeping-apps path, "don't swipe away iPhone app" | 2026-11-25 | yes |
| S8 | https://merge.watch/guides/remove-disconnection-icon-android-watch/ | Status indicator workaround | 2026-11-25 | yes |
| S9 | https://merge.watch/guides/how-to-connect-samsung-galaxy-watch-7-to-iphone/ | Samsung flow, feature list (calls, camera, health types) | 2026-11-25 | yes |
| S10 | https://merge.watch/guides/how-to-connect-samsung-galaxy-watch-4-to-iphone/ | Samsung flow, "built-in wireless installer" | 2026-11-25 | yes |
| S11 | https://merge.watch/guides/how-to-connect-samsung-galaxy-watch-9-to-iphone/ | 10-step flow, activation routes, limitations | 2026-11-25 | yes |
| S12 | https://merge.watch/guides/how-to-connect-samsung-galaxy-watch-ultra-2-to-iphone/ | "proximity features", LTE is not a relay, phone-free installer | 2026-11-25 | yes |
| S13 | https://merge.watch/guides/activate-samsung-galaxy-watch-without-android-phone/ | Borrow-an-Android activation runbook | 2026-11-25 | yes |
| S14 | https://merge.watch/setup/setup-samsung-galaxy-watch-without-a-phone/ | Same topic, "setup utility, not a permanent requirement" | 2026-11-25 | yes |
| S15 | https://merge.watch/guides/does-galaxy-watch-work-with-iphone/ | Feature table for Galaxy Watch | 2026-11-25 | yes |
| S16 | https://merge.watch/guides/does-wear-os-work-with-iphone/ | Feature table, calls on Samsung only, two pairing methods | 2026-11-25 | yes |
| S17 | https://merge.watch/guides/how-to-connect-google-pixel-watch-3-to-iphone/ | Start Pairing flow, 5-minute window, Fitbit notes | 2026-11-25 | yes |
| S18 | https://merge.watch/guides/how-to-connect-google-pixel-watch-5-to-iphone/ | Android 12+ activation, limitations | 2026-11-25 | yes |
| S19 | https://merge.watch/guides/does-pixel-watch-work-with-iphone/ | Pixel feature table (calls not stated) | 2026-11-25 | yes |
| S20 | https://merge.watch/guides/how-to-connect-oneplus-watch-2-to-iphone/ | "Nearby watches" flow, Devices → Add Another | 2026-11-25 | yes |
| S21 | https://merge.watch/guides/how-to-connect-oppo-watch-x-to-iphone/ | Nearby-watches flow | 2026-11-25 | yes |
| S22 | https://merge.watch/guides/how-to-connect-xiaomi-watch-5-to-iphone/ | Nearby-watches flow, Repair pairing | 2026-11-25 | yes |
| S23 | https://merge.watch/guides/does-xiaomi-watch-work-with-iphone/ | Per-model pairing method, "Try alternative pairing" | 2026-11-25 | yes |
| S24 | https://merge.watch/guides/how-to-connect-mobvoi-ticwatch-pro-5-to-iphone/ | Start Pairing flow, extra health types (HRV) | 2026-11-25 | yes |
| S25 | https://merge.watch/guides/does-ticwatch-work-with-iphone/ | Feature table and FAQ | 2026-11-25 | yes |
| S26 | https://merge.watch/guides/how-to-connect-fossil-gen-6-to-iphone/ | Start Pairing flow | 2026-11-25 | yes |
| S27 | https://merge.watch/guides/how-to-connect-fitbit-ace-lte-to-iphone/ | Start Pairing flow, kid watch | 2026-11-25 | yes |
| S28 | https://merge.watch/guides/receive-notifications-from-two-phones-on-one-smartwatch/ | Dual-phone runbook | 2026-11-25 | yes |
| S29 | https://merge.watch/guides/dual-phone-smartwatch-work-and-personal-setup/ | Dual-phone runbook, Devices → Add Another | 2026-11-25 | yes |
| S30 | https://merge.watch/guides/smartwatch-for-two-phones-iphone-and-android/ | Dual-phone runbook | 2026-11-25 | yes |
| S31 | https://merge.watch/guides/can-one-smartwatch-work-with-iphone-and-android/ | Simultaneous dual-phone claim, calls from either phone (Samsung) | 2026-11-25 | yes |
| S32 | https://merge.watch/guides/apple-watch-android-requirements-for-merge/ | Generic runbook (reverse direction) | 2026-11-25 | yes |
| S33 | https://merge.watch/refunds.html | Store billing, refund and cancel rules | 2026-11-25 | yes |
| S34 | https://merge.watch/privacy_policy.html | Data flows, SDKs, relay, health categories (effective 2026-11-21) | 2026-11-25 | yes |
| S35 | https://merge.watch/terms_of_service.html | BLE plus internet relay, Merge Assistant (Siri), plans, sideload installer (updated 2026-11-21) | 2026-11-25 | yes |
| S36 | https://apps.apple.com/us/app/id6698894276 | iOS listing: IAP prices, rating, size, iOS 17.2, version history, privacy labels | 2026-11-25 | yes |
| S37 | https://apps.apple.com/mx/app/id6698894276 | MX prices and rating | 2026-11-25 | yes |
| S38 | https://apps.apple.com/es/app/id6698894276 | ES prices (EUR) and rating | 2026-11-25 | yes |
| S39 | https://play.google.com/store/apps/details?id=com.mergecorp.merge | Wear OS watch app "Merge - Connect to iPhone": description, rating, downloads, data safety, sample reviews | 2026-11-25 | yes (WebFetch truncated; read with one plain HTML GET) |
| S40 | https://play.google.com/store/apps/details?id=com.mergecorp.mergeandroid | Android phone app "Merge - Connect to Watch" (Apple Watch direction): description, 7-day trial | 2026-11-25 | yes (same method) |
| S41 | WebSearch results (App Store CA/IN snippets; techpp.com article) | Regional prices; an older third-party note of $3.99 Basic / $5.99 Pro per month | 2026-11-25 | snippets only; techpp article not opened |

**Not read individually:** the remaining per-model "How to connect X to iPhone" guides. These are Galaxy Watch 5/5 Pro/6/6 Classic/8/8 Classic/FE/Ultra/4 Classic, Pixel Watch 1/2/4, OnePlus 2R/3/4, OPPO X2/X2 Mini/X3, TicWatch E3/Pro 3/Pro 3 Ultra, Xiaomi 2/2 Pro, TAG Heuer, Hublot and Montblanc. The pages I sampled are templated, so these should follow one of the two pairing patterns in section 3. Also not read: the "Does OnePlus/OPPO/Fossil work with iPhone" pages and the 17 Apple-Watch-to-Android guides apart from S32. Those are out of scope. No dedicated FAQ or pricing page exists on the site: the home page has no FAQ section and no pricing link.

Identifiers found:
- iPhone app: "Merge: Connect Android Watches", id6698894276
- Wear OS app: `com.mergecorp.merge`, shown on Play as "Merge - Connect to iPhone"
- Android phone app: `com.mergecorp.mergeandroid`, "Merge - Connect to Watch"
- Apple Watch app: id6755054765, "Merge: Watch to Smartphone" / "Merge - connect to smartphone"
- Developer: Merge Labs Ltd, Givatayim, Israel. Contact contact@merge.watch, Twitter/X @MergeWatch.
- Store search term used in the guides: "Merge Watch". The home page says to search "Merge Android" on iPhone.

---

## 2. Feature list (Wear OS watch with iPhone)

| # | Feature | Claimed where | How it works, per public text |
|---|---------|---------------|-------------------------------|
| F1 | **iPhone notifications on the watch** | S36, S39, S9, S15, S16, S19, S25 | Alerts from iPhone apps (examples given: Messages, Mail, WhatsApp, Instagram, calendar) arrive "in real time". The user can dismiss them on the watch (S9, S10). Requires notification permissions on both devices. If alerts don't arrive, allow iOS "system notifications" for Merge/the watch (S6, S20, S21, S28). INFERENCE: this wording suggests the watch is exposed to iOS as a BLE accessory that receives notifications through Apple's ANCS. The "Share System Notifications" prompt iOS shows for BT accessories fits this. Merge never names ANCS. |
| F2 | **Per-app notification control ("notification filter")** | S36/S39 ("new notification control feature"), S15, S16, S25 | Choose which apps send alerts. S15/S16/S25 also say you can set which notification types each app may send. Guides advise setting filters "intentionally" when volume is high (S6, S28, S29). |
| F3 | **Replies / actions on notifications** | Not claimed for Wear OS→iPhone | No Wear OS guide mentions replying. "Messaging and Actions / reply" is claimed only for the Apple Watch→Android app (S40). Treat reply as **not offered** for Wear OS+iPhone. A Siri route exists instead (F9). |
| F4 | **Phone calls on the watch** | S36, S39 ("Make and receive phone calls right from your watch"), S9, S10, S11, S12, S15, S31 | The watch rings for incoming iPhone calls. The user can answer or decline and "talk from your wrist" (S9). S16 says calls work on **Samsung Galaxy Watch only**. S31 adds that calls work "from either phone" in a dual-phone setup. Audio routing is **not described** in any guide (S10, S11). A Play review (S39, 2024) says the user could "accept calls AND make calls from the watch through the iPhone". INFERENCE: this matches the watch acting as a Bluetooth Classic HFP hands-free device paired in iOS Settings → Bluetooth, with call audio on the watch. Not stated by Merge. |
| F5 | **Contacts sync / caller ID** | S36, S39, S9, S16, S19 | iPhone contacts sync to the watch so incoming calls show names. This is the only call-related item claimed for Pixel, TicWatch and Xiaomi (S19, S23, S25). The privacy policy says synced contacts are kept on the device (S34). |
| F6 | **Health and workout sync to Apple Health** | S36, S39, S9, S10, S11, S15–S19, S24 | Types named: heart rate, steps, distance, floors climbed, calories, and workouts with GPS routes and pace. TicWatch adds HRV (S24). Pixel says Fitbit metrics (Readiness, AZM, cEDA, skin temp) "sync to Apple Health through Merge" (S17), an unverified vendor claim. Sync runs "in the background" or "after each session" (S9, S15, S19). Needs Health permissions on both devices. The privacy policy lists possible categories: activity, workouts, sleep, body measurements, vitals, nutrition, workout routes, reproductive health (S34). |
| F7 | **Merge Workout app / activity rings / goals / streaks** | S36 v2.1.0 (Aug 18: "New Merge Workout app for more accurate tracking"); v3.0.0/3.0.1 (Nov 2026); S39 | Activity rings, daily goals, streaks and workout highlights. Includes a Wear OS **tile** and a **watch face complication** for the rings (S39). Goals, streaks and preferences are backed up to the user's iCloud, on by default, with a "Delete iCloud Backup" option (S34). |
| F8 | **Music / playback control** | S36, S39, S9–S11, S16 | Play, pause, skip and volume for iPhone audio from the watch. INFERENCE: this is consistent with AVRCP or Apple Media Service (AMS) over BLE. Not stated. |
| F9 | **Siri control ("Merge Assistant")** | S36 v3.0.x, S39, S35, S34 | "Talk to Siri on the iPhone from a compatible watch": send messages, run shortcuts, control the smart home. ToS: "Merge Assistant (connecting Wear OS watches to Siri on a paired iPhone)". Privacy: voice requests are processed by Apple under Siri's notice, and analytics record "Siri session success". "Compatible watch" is not defined. INFERENCE: this is likely tied to HFP voice-recognition activation, which would explain the "compatible" qualifier. |
| F10 | **Camera remote** | S36 v1.0.17 (Jan: "New camera control feature"), S9, S10, S11 | The watch triggers the iPhone camera shutter, but **only while the iPhone Camera app is open** (S9, S17, S24). INFERENCE: this matches a BT HID consumer-control "volume up" press, which only works with the Camera app in front. |
| F11 | **Find My (ring device)** | S36, S39 | A "Find My" ring feature locates the iPhone or watch when nearby. |
| F12 | **Dual phone (one watch, iPhone plus Android at once)** | S1, S4, S28–S31 | The watch keeps its native Android-phone link, and Merge adds the iPhone. Described as "both at once", with no switching (S31). Watch UI: **Devices → Add Another** adds a second phone (S20, S21, S29). |
| F13 | **Standalone setup / "wireless installer" (no Android phone)** | S36 v1.0.15/1.0.16 (Dec 2025: "New standalone setup solution"), S10, S11, S12, S35 | The iPhone app contains a "built-in wireless installer" for Galaxy Watch. It gives "a phone-free route to an initialized Galaxy Watch", including Play Store access (S11, S12). ToS: an "optional companion-installation feature using developer options or sideloading", with liability disclaimed for instability or voided warranty. INFERENCE: this is likely ADB over Wi-Fi from the iPhone to the watch. Requires Wi-Fi (S12). Other guides contradict it by saying an Android phone is required for activation (S13, S14). |
| F14 | **Internet relay fallback** | S34, S35 | BLE is primary. An end-to-end encrypted internet relay (AWS) plus APNs/FCM push is a fallback "when Bluetooth is unavailable or unstable". The privacy policy ties the relay explicitly to **Apple Watch** (S34). For Wear OS, S12 says features are "proximity features" and the watch's LTE "does not turn Merge into an unlimited-distance iPhone relay". **Treat the relay as Apple-Watch-direction only.** |
| F15 | **App icons for notifications** | S34 | "App icon requests" send app identifiers and IP to the server, which suggests a server-side lookup of icons for iOS bundle IDs. |
| F16 | **Glucose Sync** | S35, S40 | Apple Watch→Android only: the latest reading from Health Connect is sent to the watch. Not offered for Wear OS+iPhone. |
| F17 | **Known Locations** | S34 | Android only: uses precise and background location. Purpose isn't described; possibly location-based behaviour. Out of scope. |
| F18 | Encrypted BT, data stays on device | S1, S2, S36, S39 | "secure Bluetooth connection and keeps your data on your devices". |
| F19 | Watch-native features listed in guides (not Merge features) | S9, S12, S17, S18, S24 | AFib alerts, Loss of Pulse, Energy Score, dive, Trail Run, ECG via the Fitbit iOS app. These run on the watch and are presented as still working. Not Merge functionality. |

Background behaviour, battery and reliability notes:
- The **iPhone app must keep running in the background**. Swiping it away in the app switcher makes iOS kill it, and the connection drops "until you reopen it manually". S7 calls this the "single most common reason" for drops. This implies no state restoration / relaunch that survives a force-quit.
- On the watch, **remove Merge from Samsung Sleeping apps**. One path is Settings → Battery and device care → Battery → Sleeping apps → long-press Merge → Remove from list (S7). Another given path is Settings → Battery → More battery settings → Sleeping apps (S9, S10, S11). Otherwise notification delivery "slows down or becomes inconsistent".
- Keep Merge "available in recent apps" and avoid watch battery policies that "aggressively suspend the app" (S6, S13, S14, S28).
- OnePlus/OPPO: keep the watch in **Smart Mode**, not the RTOS power mode, so Wear OS apps run (S20, S21).
- Reconnect test after setup: toggle Bluetooth and confirm auto-reconnect (S13). A user tip (S39 review): toggle iPhone BT while the watch shows "connecting...".
- Version 1.0.19 notes "battery optimization" (S36). No numbers on battery impact are published anywhere.

---

## 3. Screens and flows inferred

### 3a. Store install
- iPhone: App Store, search "Merge Watch" (or "Merge Android" per S1), or scan a QR code on the site.
- Watch: Play Store on the watch, search "Merge Watch"/"Merge". S4 says it can also be installed "through the iPhone app" (the installer, F13).

### 3b. Prerequisite: watch activation (outside Merge)
- Typical route: borrow an Android phone and use the OEM app (Galaxy Wearable / Pixel Watch app / Wear OS by Google / OHealth / Mi Fitness). Sign in to Google, get Play Store access, update firmware. Then return the phone. Takes "10–15 minutes" (S16, S17, S24, S25). Android version floors: 8.0+ for Pixel 3 (S17), 12+ for Pixel 5 (S18), 13+ with GMS and ≥1.5 GB RAM for Samsung (S11, S12).
- Samsung alternative: the wireless installer in the iPhone app (F13).

### 3c. iPhone app: device catalog
- **Supported Devices** list in the iPhone app. The user picks their exact model, which opens a model page with the instructions for that model (S6, S20, S30).
- A saved-watch entry shows **Connect**, **Complete pairing** and **Repair pairing** (S20, S22).

### 3d. Pairing pattern A: "Start Pairing" + iOS Bluetooth settings
Used by Pixel, TicWatch, Fossil, Fitbit Ace LTE, Xiaomi Watch 2/2 Pro (S17, S23, S24, S26, S27).
1. iPhone: open Merge, allow Bluetooth, keep it in the foreground.
2. Watch: open Merge, allow Bluetooth / nearby devices.
3. Watch: tap **Start Pairing**. The watch is discoverable for **5 minutes**.
4. iPhone: **Settings → Bluetooth**, tap the watch, confirm the code.
5. Watch: tap the iPhone in the Merge list to finish.
6. iPhone: subscription paywall "if prompted".
7. Grant notification and health permissions on both devices.
- Fallback: **"Having trouble? Try alternative pairing"** on the model page (S6, S16, S23).
- Counting from opening the apps, that is roughly 7–9 taps plus permission dialogs, including one trip into iOS Settings.

### 3e. Pairing pattern B: "Nearby watches" (in-app)
Used by OnePlus 2/2R, OPPO Watch X, Xiaomi Watch 5 (S16, S20, S21, S22).
1. Watch: open Merge, allow Bluetooth, keep it on screen.
2. iPhone: Supported Devices → model, which starts the search.
3. If the watch already has a saved phone: watch **Devices → Add Another** (older builds: "Start pairing").
4. iPhone: tap the watch under **Nearby watches**, confirm matching codes on both devices.
- Fallback: **Search again**.
- About 4–5 taps plus a code confirmation.

### 3f. Samsung flow (S9, S10, S11, S12)
This is a third variant in which **the watch scans for the iPhone**:
1. iPhone app in the foreground.
2. Watch app: grant the nearby-devices permission.
3. The iPhone name appears in the watch list. Tap it.
4. Accept the BT code on both devices and wait for "connected" status.
5. Subscription.
6. Notification and Apple Health permissions.
7. Sleeping apps exclusion.
- S11 has 10 steps, of which 6–7 are taps.

### 3g. After pairing
- Notification test (S13, S20).
- System notifications toggle in iOS for Merge/the watch (S6).
- Validate dual phone with a test alert from each phone (S28–S30).

### 3h. Other UI surfaces mentioned
- Paywall (14-day trial, Basic/Pro, monthly/annual).
- Per-app notification filter.
- In-app troubleshooter (S39 review: "The trubleshooter is of no use").
- Activity rings, goals and streaks screens, plus a Wear OS tile and complication.
- Find My ring.
- Camera shutter.
- Music controls.
- Siri trigger.
- iCloud backup toggle ("Delete iCloud Backup").
- Watch-side "Devices" list.

### 3i. OS workaround
- To hide the "disconnected" icon on the watch: Settings → Advanced features → Status indicators → "Hide after 2 seconds" (S8). This also affects the charging and DND indicators.
- The icon appears because no Android phone is paired natively.

---

## 4. Pricing / business model

| Item | Fact | Source |
|------|------|--------|
| Download | Free; features need an active subscription | S36, S40 |
| Trial | **14 days** on iOS (S36–S38); **7 days** on the Android phone app (S40) | |
| iOS plans (US) | Basic Monthly **$3.99**, Basic Annual **$35.99**, Pro Monthly **$5.99**, Pro Annual **$59.99** | S36 |
| MX | Basic $59 / $479, Pro $99 / $959 (currency symbol only, presumably MXN) | S37 |
| ES | Basic €3.99 / €35.99, Pro €5.99 / €59.99 | S38 |
| CA / IN snippets | Basic $4.99 / $47.99 CAD; IN Basic ₹199 / ₹1,999, Pro ₹299 / ₹2,999 | S41 |
| Basic vs Pro difference | **Not published anywhere I could reach** | S1, S33, S35, S41 |
| Billing | App Store or Google Play only. Subscriptions cannot be moved between stores. Refunds are handled by the store; Google refunds may also be granted case by case. | S33, S35 |
| Ads | None claimed | S2 |
| Company | Merge Labs Ltd (Israel); Israeli law, Tel Aviv courts | S35 |

The subscription is bought in the iPhone app.

---

## 5. Requirements

- **iPhone:** iOS **17.2 or later** (S5, S6, S36). The listing also runs on visionOS 1.1+. S2 (llms.txt) says "iOS 14+", which is outdated or wrong. The Fitbit Ace guide mentions iOS 16+, but that is for the Fitbit parent app.
- **Wear OS:** S2 says "Wear OS 3.0+". In practice the oldest supported device is Galaxy Watch 4 (Wear OS 3). No version is stated anywhere else.
- **Watch must have Google Play**, which means it is activated. Xiaomi Watch 5 must be the global Wear OS edition. OnePlus/OPPO must be in Smart Mode.
- **Supported Wear OS watches (S5):**
  - Samsung: Galaxy Watch 4, 4 Classic, 5, 5 Pro, 6, 6 Classic, 7, FE, Ultra, 8, 8 Classic, Ultra 2, 9
  - Google: Pixel Watch 1, 2, 3, 4, 5
  - Xiaomi: Watch 2, 2 Pro, 5. Also "Xiaomi Watch 3 support" in iOS v2.1.5 (S36); not on the models page.
  - OnePlus: Watch 2, 2R, 3, 4
  - OPPO: Watch X, X2, X2 Mini, X3
  - Mobvoi: TicWatch E3, Pro 3, Pro 3 Ultra, Pro 5
  - Others: Fossil Gen 6, TAG Heuer Connected Calibre E4, Hublot Big Bang e Gen 3, Montblanc Summit (3), Fitbit Ace LTE
- **Bluetooth** on both devices. Devices within a few feet during pairing, and within BT range for live features.
- One-time **Android phone** for activation (except the Samsung wireless-installer route).
- Store stats for the Wear OS app (S39): 500K+ downloads, 3.5★ (1.04K reviews), watch-only rating 3.9★ (997), updated Nov 22, 2026. Data safety: "App info and performance" collected, not shared, encrypted in transit, "can't be deleted".
- Store stats for the iOS app (S36): 3.3★ (369 US ratings), 45.7 MB, English only, Utilities, 4+, current version 3.0.1 (released about Nov 24, 2026). Privacy labels: Data Not Linked to You only (User ID, Product Interaction, Crash, Performance). No tracking.

Version history highlights (S36):
- 1.0.11 (Aug 2025): Galaxy Watch 8
- 1.0.15/1.0.16 (Dec 2025): standalone setup
- 1.0.17 (Jan 2026): camera control
- 2.1.0 (Aug 2026): Merge Workout app
- 2.1.5 (Sep 22): Xiaomi Watch 3
- 3.0.0/3.0.1 (Nov 2026): activity rings, goals, streaks, Siri Control
- Versions 1.0.5–1.0.9 (Jan–May 2025) all say "redesigned Merge app". The copyright line says 2023–2026, so the app is older than this listing.

---

## 6. Limitations: what is NOT supported, plus admitted caveats

1. **Calls only on Samsung Galaxy Watch.** Other Wear OS watches get caller ID only (S16, S31). The Pixel pages don't say either way (S19).
2. **No notification replies or actions on Wear OS+iPhone.** These are never claimed (F3). The only route for messaging is Siri (v3.0).
3. **Range-bound.** All features are "proximity features". The watch's LTE does not extend them, and there is no relay for Wear OS (S12, S16, S25).
4. **iPhone app force-quit kills the link** until it is reopened manually (S7).
5. **Samsung Sleeping apps / OEM battery policies** throttle or kill the watch app (S7, S9–S12).
6. **Activation needs an Android phone once** for most watches. Google "does not provide an iOS activation path" (S17). Setup without Android "can fail partway" and partial setups cause BT instability (S13). Phone-free operation is "reduced, not eliminated" (S14). Samsung has the wireless-installer exception.
7. **Persistent "disconnected" icon** on the watch; it can only be hidden (S8).
8. **Firmware-dependent BT.** "Some Oppo and older OnePlus variants may be limited by Bluetooth LE implementation outside app control" (S5).
9. **Not an Apple Watch replacement.** It does not "reproduce every Samsung-phone integration". Samsung account, Galaxy-only and regional health tools stay under Samsung's rules (S11). Merge doesn't unlock region-locked medical features or put the Pixel Watch app on iOS (S18).
10. **Camera remote only works with the iOS Camera app open** (S9, S24).
11. **Not a medical device.** No guarantee of uptime, notification delivery, sync accuracy or Siri audio. The relay, push, Bluetooth and platform restrictions are excluded from liability (S35).
12. **Sideload installer risk** (developer options): instability, voided warranty and account action are disclaimed (S35).
13. **High alert volume reduces reliability**, so use filters. OS updates may change behaviour (S6, S28–S32).
14. **User-reported issues (Play reviews, S39; anecdotes, not verified):**
    - Repeated failures to connect.
    - Notifications stopping after about 30 minutes and needing a full re-pair.
    - Occasional "can't find the iPhone", fixed by toggling Bluetooth.

### Internal inconsistencies in the vendor docs
- iOS floor: 14+ (S2) vs 17.2+ (S5, S6, S36).
- Activation: Android "required" vs phone-free installer (S13, S14 vs S10, S11, S12).
- Samsung pairing direction: the watch picks the iPhone (S9, S11, S12) vs the iPhone picks from "Nearby watches" (S6, S20). These are probably model-dependent.
- Sleeping-apps menu path differs between S7 and S9.
- Model count: "31+" (S2) vs 36–37 listed (S1, S5).

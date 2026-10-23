# Prompt to rebuild TvLauncher from scratch

Paste into Claude Code (on a PC on the same Wi-Fi as the box), with `themes/THEME_SPEC.md` and
`themes/PROMPT_THEME.md` attached (they are in French; Claude reads them fine). Replace the `<…>` items.

---

I want a custom Android TV launcher, coded from scratch, that fully replaces my box's Google TV home screen. Work on the real device, step by step, and verify every change with a screenshot taken on the TV (`adb exec-out screencap -p`). Ask me about anything that depends on my taste, with suggestions. Reply in English; the launcher's UI text is in <French>.

## Hardware and constraints

- Box: <Xiaomi TV Box S 3rd gen>, Google TV, Android 14 (API 34), 2 GB RAM, 1080p screen at 60 Hz. Still confirm with `getprop`.
- The PC runs Windows with Android Studio installed (SDK and JDK `…\Android Studio\jbr`). Put the project in a **short path** (e.g. `C:\Users\<me>\TvLauncher`): long paths break Gradle on Windows. Use the Gradle wrapper.
- Performance comes first: target 60 fps, measure with `dumpsys gfxinfo <pkg> framestats` (UI, render and GPU time separately) before and after each effect.

## Tech stack

Kotlin + Jetpack Compose (foundation only, no tv-material), minSdk 31, targetSdk 35, AGP 8.13, Kotlin 2.2, kotlinx-serialization, Coil 3 (+ coil-svg, okhttp), ZXing core (QR codes). A single `singleTask` activity with HOME + LEANBACK_LAUNCHER filters, `<queries>` for LEANBACK_LAUNCHER, LAUNCHER, `spotify:` and the Nuvio/Spotify packages. Back never closes the launcher (it returns to the home page).

## ADB connection

- Wireless debugging: `adb pair` with the code, `adb connect`, then `adb tcpip 5555` for a fixed port. The box's IP and the wireless debugging port can change: find them again with `adb mdns services`.
- On Android 14, if `adb install` hangs, use `pm install`.
- Before any command that changes the screen, check I'm not watching something (`dumpsys activity activities | grep topResumedActivity`, `dumpsys media_session`). After your tests, put the screen, theme and settings back the way I had them.

## Replacing the Google TV home — IMPORTANT

- **NEVER disable** `com.google.android.apps.tv.launcherx` or `com.google.android.tungsten.setupwraith`: on this box, setupwraith grabs the Home key and the screen stays black; `set-home-activity` answers "Success" with no effect.
- What works: an **accessibility service** in the launcher, in its **own process** (`android:process=":home"`) so it survives a UI crash:
  - `flagRequestFilterKeyEvents` + `canRequestFilterKeyEvents`: `onKeyEvent` intercepts `KEYCODE_HOME`, consumes it and opens the launcher (configurable);
  - fallback: on `TYPE_WINDOW_STATE_CHANGED` from the launcherx package, reopen the launcher (max once per second);
  - in `onServiceConnected`, check `rootInActiveWindow` (box boot, app update);
  - enable it with `settings put secure enabled_accessibility_services` while **keeping** the existing services.
- Presses simulated with `adb shell input keyevent 3` do not go through accessibility: test the Home key with the **real remote** and a log.
- Check that no other accessibility service (e.g. **Button Mapper**) relaunches Google TV or an old launcher on Home (`logcat`: "START … launcherx … from uid"). Ask me before disabling it.

## Home page (approved layout)

Theme-driven background. From top to bottom:
1. **Top bar**: time + date (optional theme subtitle), pill tabs Home / Apps / Settings, on the right Wi-Fi status (refreshed: at boot the Wi-Fi comes up after the launcher), Bluetooth, Android settings.
2. **Nuvio block**: profile avatar + name, profile tabs with photos (hidden when only one profile), 3 landscape cards (backdrop image, SERIES/MOVIE badge, title, "S2 · E4 — episode name · 23 min left" only on the focused card, progress bar). The focused card is wider and taller (animated). Column layout: badge at the top, texts at the bottom, never overlapping.
   - **Thumbnails mode** (setting): when focus is not on Nuvio, the header disappears and the cards grow.
   - **Cinema mode** (setting): when focus is on Nuvio, the focused card widens further, Spotify/SmartTube become a thin bar (cover, title, buttons) and the dock fades out.
3. **Spotify and SmartTube strips** side by side (each can be hidden): cover/thumbnail with a fade + zoom on change, title scrolls if too long, small animated equalizer while playing, artist and time shown separately (time always readable). Spotify: previous / play-pause / next. SmartTube: "Resume".
4. **Dock**: the chosen apps (no "All apps" tile). Keys 1–9 = dock apps, Menu = Apps page.

Do **not** add a playlist row or a Spotify strip that expands: it resizes Nuvio, and a `LazyRow` inside a `Row` using `IntrinsicSize` crashes Compose.

## Nuvio (Nuvio Sync, Supabase backend)

- Server `https://api.nuvio.tv` (fallback `https://api-two.nuvioapp.space`). The public "anon" key is not in the source code: **ask my permission**, then read it from the installed Nuvio APK (`pm path com.nuvio.tv`, `adb pull`, search the `.dex` files for `eyJhbGciOiJIUzI1NiIs`), and delete the APK copy afterwards. Headers: `apikey` + `Authorization: Bearer <token>`.
- QR sign-in (like Nuvio TV): RPC `start_device_login_session` (p_device_nonce, p_redirect_base_url `https://nuvio.tv/link`, p_device_type `tv`, p_device_name) → show `verification_uri_complete` as a QR code + `user_code`; poll `poll_tv_login_session` (p_code = device_code, p_device_nonce); when `approved`, exchange via `POST /functions/v1/tv-logins-exchange` {code, device_nonce}; refresh with `POST /auth/v1/token?grant_type=refresh_token`. Generate a new code automatically when it expires.
- Per-profile data: `sync_pull_profiles` (profile_index, name, avatar_color_hex, avatar_url, avatar_id), `sync_pull_watch_progress` (p_profile_id → content_id, content_type, video_id, season, episode, position, duration, last_watched), `sync_pull_watched_items` (watched episodes).
- Avatars: `avatar_url`, else `get_avatar_catalog` (`/storage/v1/object/public/avatars/<storage_path>`), else `get_member_profile_avatar_catalog` (authenticated download `/storage/v1/object/authenticated/membership-profile-avatars/<path>`, cached).
- Continue watching: progress between 2 % and 92 %, most recent per title, 3 max. **Next episode**: if the last watched episode of a series (watched < 30 days ago) is finished, offer the next unwatched, already released episode (Cinemeta `videos[].released`). Timestamps are sometimes in seconds, sometimes in milliseconds: normalise.
- Metadata: Cinemeta `https://v3-cinemeta.strem.io/meta/<type>/<id>.json` (title, background, poster, logo, episode names), 7-day disk cache.
- Launching: `Intent(ACTION_VIEW)` to `com.nuvio.tv/com.nuvio.tv.MainActivity`, extras `contentId`, `contentType`, `videoId` (or `id:season:episode`), `name`, `poster`, `backdrop`, `logo`, `season`, `episode`, `episodeTitle`, `launchMode=stream`, flags `NEW_TASK | CLEAR_TASK`. Nuvio shows its profile picker (I must turn off "remember last profile" in Nuvio). There is no extra to choose the profile.
- Refresh every time the launcher comes back to the front. Setting: which profiles are shown (at least one).

## Live playback (Spotify, SmartTube) via media sessions

- An empty `NotificationListenerService` + `MediaSessionManager.getActiveSessions`, granted with `adb shell cmd notification allow_listener <pkg>/<service>`.
- Verified pitfalls:
  - Spotify puts its **logo** in `METADATA_KEY_DISPLAY_ICON`: never use it as the cover; keep the last cover while the title doesn't change.
  - SmartTube reopened from a link publishes empty metadata, then `duration = -1` and `position = 0` when closing: keep the previous title, duration and position.
  - When playback moves to another device (Spotify Connect) the session disappears or freezes: freeze the position; treat it as "playing" only while the extrapolated position < duration + 5 s.
  - If the session is no longer active, send commands as media keys (`AudioManager.dispatchMediaKeyEvent`).
- Save the last Spotify and SmartTube playback to disk (including a downscaled cover) so it shows after a reboot.
- SmartTube: `METADATA_KEY_MEDIA_ID` = YouTube video id; thumbnail `https://i.ytimg.com/vi/<id>/mqdefault.jpg`; resume with `https://www.youtube.com/watch?v=<id>&t=<s>s` and package `org.smarttube.stable`.
- Spotify history (optional): Web API with OAuth PKCE using my own Client ID (development mode: the owner must have Premium, redirect URI `http://127.0.0.1:8888/callback`).

## Apps page

Grid of all apps (TV banner, else icon), sort "my order / A-Z / recent / hidden", updated automatically on install or uninstall (BroadcastReceiver), dock icons loaded first and in parallel. Long press → menu: add/remove from dock, move, hide, info, uninstall, **Cancel**. The menu opens while the OK key is still held: **ignore OK until it is released**, otherwise the first action fires on its own. Move mode: arrows = move, OK = confirm, Back = **cancel** (restore the order); focus stays on the moved app.

## Settings

Side menu: Themes, Home, Accounts (Nuvio, Spotify), Remote, Network (type, network name, IP, signal in words), Bluetooth (paired devices; `BLUETOOTH_CONNECT` granted via ADB), Sounds, Performance (economy mode, memory), Android system (shortcuts to the system settings), About (version, device, how to go back to Google TV). Contained navigation (`focusGroup` + `focusProperties`): right = first item of the content, left = back to the section, no diagonal jumps. Scrollable contents with a small margin so the enlarged focused item is never clipped.

## Themes

- Format described in `THEME_SPEC.md` (attached): implement exactly what its §0 table marks as supported, and update that table at the end. `PROMPT_THEME.md` (attached) is used to generate themes with Claude.
- Tolerant parsing: an invalid section = defaults + a warning, never a crash. Text sizes are capped. Notches longer than their edge: shorten them (otherwise a crash). Inner padding computed from the frame's cut corners/notches (`frameInset`).
- Sending from the phone: a small local HTTP server open only while the QR screen is shown, one-time code in the URL, web page (`.json`/`.tvtheme` file, pasted text, link). ZIP archives: reject `..` paths, handle folders written with `\` (Windows archives) and a root folder. 15 s preview with "Keep" / "Cancel", automatic revert otherwise. Delete a received theme with a long press.
- Built-in sounds synthesised as WAV on first launch (`builtin:tick`, `click`, `swoosh`, `chime`…), background music off by default, only on the home page, paused when another app plays (careful: in `onCompletionListener`, call the music player's `release()`, not the `MediaPlayer`'s).
- Ship 3 themes: "Frosted glass" (default, dark, matte glass, shader background tinted with the focused title's colour), "Neon night", "Neon City" (cyberpunk demo: cut corners, per-card neon, hazard stripes, scanlines).

## Performance lessons (apply from the start)

- Background shader computed at half resolution in a reduced layer then scaled up, 30 fps max; static backgrounds (gradient, vignette) drawn once into a cached image.
- No `clip` per panel; glass highlight limited to the top third.
- Focus animation only in `graphicsLayer` / `drawWithContent` (no recomposition); the media colour is passed as a lambda read at draw time.
- Frame strokes and neon glows pre-rendered into an image; neon = successive strokes, no GPU blur.
- Animated decorations at 15 fps, paused for 1.5 s after every remote key press.
- A `RuntimeShader` cannot be drawn into an in-memory image (software rendering): draw it directly, at half resolution.
- Diagnostic switches via ADB (`--es perf "nobg,noglass,noframes,nodecor,nooverlay"`) to measure each effect's cost. An economy mode that turns off heavy effects.

## Delivery

- Release signing with a fixed key **outside the repository** (path in `local.properties`, falling back to the debug key) and versionName 1.0.0.
- `tools/install.ps1 <ip:port>` (install, media/Bluetooth/location permissions, accessibility service while keeping the others) and `tools/back-to-google-tv.ps1 <ip:port> [-Uninstall]`, saved as UTF-8 with BOM.
- README: features, build, install, going back to Google TV, signing, how to close ADB (turn off wireless and USB debugging, revoke authorisations, reboot, check with `Test-NetConnection <ip> -Port 5555`).
- Git from the start, with one commit per step; `captures/`, `*.apk`, `local.properties` ignored (the screenshots show my watch history).
- Final test: **reboot the box** and check the launcher opens on its own, then a real Home press.

## Work order

1. Check the environment, create the project, minimal theme engine + home page with demo data, install, measure.
2. Nuvio (sign-in, continue watching, next episode, avatars, launching).
3. Live Spotify and SmartTube.
4. Sending themes by QR code.
5. Apps (dock, order, hide) and complete Settings.
6. Advanced themes (image/blurred-media backgrounds, icons, sounds, music, frames, decorations, text effects, accents) + Neon City theme + measurements.
7. Home replacement (accessibility), polish, signing, scripts, README, reboot test.

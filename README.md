# NEXNOTE — encrypted offline notes for Android (Kotlin · Compose · Room · Keystore)

## STATUS — read this first
**This project has NOT been compiled, run, or tested.** It was written in an environment with no Android SDK,
no Gradle and no network. There is **no APK in this package.** Expect that the first build may show a few
compile errors (typos, API mismatches) that need fixing. Only the AES-GCM / PBKDF2 assumptions were
sanity-checked (on a plain JVM). None of the tests below have been executed.

## Build the APK (pick one)
**A. Android Studio (easiest):** open this folder → let Gradle sync (it creates the wrapper) →
`Build > Build APK(s)` → `app/build/outputs/apk/debug/app-debug.apk`.

**B. Command line** (JDK 17, Android SDK with platform 34 + build-tools, `ANDROID_HOME` set, Gradle 8.9):
```
gradle wrapper --gradle-version 8.9
./gradlew :app:testDebugUnitTest      # JVM unit tests
./gradlew :app:assembleDebug          # APK
```
**C. GitHub:** push this folder to a repo; `.github/workflows/android.yml` runs the unit tests, builds the
debug APK and uploads it as a workflow artifact.

## Install & test on a phone
1. Enable *Developer options → USB debugging*, then `adb install app/build/outputs/apk/debug/app-debug.apk`
   (or copy the APK to the phone and allow "install unknown apps").
2. Instrumented tests (real Keystore + Room) need a device/emulator: `./gradlew :app:connectedDebugAndroidTest`.
3. Manual checklist: set a PIN → leave app (auto-lock) → unlock; wrong PIN ×5 → lockout; create note with
   bold/heading/list/checklist; kill the app mid-typing and reopen; trash → restore; edit history → restore;
   export backup → wipe via "Forgot PIN" → import backup; try a wrong backup password; try a damaged file;
   switch theme / font size / language; confirm screenshots are blocked and Recents shows a blank card.

## Architecture
`crypto/` AES-256-GCM (`AesGcmCrypter`), Keystore master key, PBKDF2 · `security/` PIN hash + lockout ·
`data/` Room entities/DAO/`NoteRepository` · `domain/` models, search/sort, markdown helpers ·
`backup/` encrypted backup format · `ui/` Compose screens · `ViewModels.kt` MVVM state · `App.kt` wiring.

## Feature status (code written / NOT verified by running)
Written: create/edit/delete/restore notes · debounced autosave + flush on pause/back · headings, bold, lists,
checklists (toolbar + tap-to-toggle preview) · folders, tags, colors, pin, favorite · in-memory search ·
sort by title/created/modified · trash (30-day auto purge) · edit history (last 30 versions, ≥30 s apart) ·
dark/light/system theme, 5 accents, font scale · Thai/English · encrypted backup export (verified by
read-back) and import (merge, never overwrites) · PIN lock, biometric unlock, auto-lock, FLAG_SECURE ·
loading/empty/error/success states.

Not done / limitations (honest list):
- **Not compiled or run** (see above). No APK.
- Notes are **plain text with Markdown markers**, not a rich-text editor. No images/attachments.
- **Search is in memory** over decrypted notes (the DB holds ciphertext). Fine for thousands of notes, not millions.
- Folder **names** are stored unencrypted (note title/content/tags are encrypted). Note metadata
  (dates, pinned, favorite, color, folder id) is also unencrypted.
- The PIN is a **gate, not a key**: the data key lives in the Android Keystore and is not derived from the PIN.
  Someone with a rooted/compromised device could bypass the PIN screen; the key is still non-exportable.
  Biometric unlock is a gate too (not a Keystore `CryptoObject`).
- Lockout timing uses the wall clock; changing the device clock can shorten a lockout.
- **Forgotten PIN is unrecoverable.** The only path is "Forgot PIN → erase all" and then restoring an
  encrypted backup. Lost backup password = backup unreadable. Uninstall / clear data = key destroyed.
- Android Auto Backup/device transfer is disabled on purpose (Keystore keys can't move devices).
- Concurrency: all writes are serialised in one process; no multi-process or multi-device sync.
- DB is schema v1; migration plumbing exists (`AppDatabase.MIGRATIONS`) but no migration has been exercised.
  Destructive fallback is intentionally **off**. Exported schema JSON will appear in `app/schemas/` on first build.
- Release build is signed with the debug key for convenience — replace before publishing.
- No app icon artwork beyond a simple vector; no accessibility audit; no tablet layout.

## Tests (written, not executed)
JVM (`app/src/test`): AES-GCM round-trip/tamper/wrong-key/AAD · PIN set/verify/lockout/reset ·
backup codec round-trip, wrong password, bit-flips, header tamper, truncation, foreign file, missing fields,
wrong version, weak password · search/filter/sort · markdown helpers.
Instrumented (`app/src/androidTest`): create/edit/trash/restore/delete · ciphertext-at-rest check ·
revision throttling · concurrent saves · unreadable-note protection · folders/tags ·
backup export → import into a fresh DB (+ idempotent re-import) · real Keystore key (non-extractable, deleted key).

# Shambhuraje Dairy — शंभुराजे दूध डेअरी

Offline-first Android app for a milk collection center (Maharashtra), built with
Kotlin, Jetpack Compose, Material 3, Room (SQLite), and an MVVM architecture.
Firebase (Firestore + Auth) dependencies are already wired in for a later cloud-sync
phase — every entity has `cloudId` / `isSynced` fields ready for it.

## Opening the project
1. Open **Android Studio** (Koala/2024.1+ recommended) → *Open* → select the
   `ShambhurajeDairy` folder.
2. Let Gradle sync (internet required the first time, to download Gradle 8.7,
   AGP 8.5.2, and dependencies).
3. Run on a device/emulator with **minSdk 24 (Android 7.0)** or higher.

There is no `google-services.json` in this repo, so the Firebase lines in
`app/build.gradle.kts` and the plugin in the root `build.gradle.kts` are
commented out / harmless placeholders. Cloud sync is scaffolded but inactive
until you add your own Firebase project config.

## Architecture
```
data/local/entity     Room entities: Farmer, MilkCollection, Advance, Payment, RateSlab
data/local/dao        Room DAOs (Flow-based queries)
data/local/AppDatabase Room database + first-run sample-data seeding
data/repository       Repository layer (single source of truth per feature)
ui/theme              Material 3 green/white color scheme + dark mode
ui/navigation         Compose Navigation graph (Screen routes)
ui/screens/<feature>  One package per feature: Login, Dashboard, Farmer,
                       Milk, Advance, Billing, Payment, Reports, Settings —
                       each with a ViewModel + Composable screen
util                  CalculationUtils (rate/amount math), MessageGenerator
                       (Marathi WhatsApp/SMS text), PreferenceManager
                       (DataStore: PIN, language, milk rate), DateUtils
```

## Database schema (SQLite via Room)
- **farmers** — id, code, name, mobile, village, bankName, accountNumber,
  ifscCode, isActive, createdAt, updatedAt, cloudId, isSynced
- **milk_collections** — id, farmerId (FK), date, shift (MORNING/EVENING),
  liters, fat, snf, rate, amount, createdAt, cloudId, isSynced
- **advances** — id, farmerId (FK), date, amount, note, createdAt, cloudId, isSynced
- **payments** — id, farmerId (FK), billMonth, date, totalMilkAmount,
  advanceDeducted, netAmount, mode (CASH/BANK/UPI), referenceNo, createdAt,
  cloudId, isSynced
- **rate_chart** — id, minFat, maxFat, ratePerLiterPerFatPoint (editable in Settings)

Monthly billing (`BillingRepository`) derives everything else on demand:
Total Milk → Total Amount → Advance Deduction (capped at the bill amount) →
Remaining Advance → Net Payment → Paid/Pending status.

## Features implemented
1. Login — mobile + 4-digit PIN, separate Admin PIN login (first entry bootstraps
   the credential; later entries must match)
2. Dashboard — total farmers, today's collection/amount, total advance pending,
   quick-action grid
3. Farmer management — add/edit/delete (soft delete), search by name/village/mobile/code
4. Milk collection — morning/evening shift, liters/FAT/SNF, live auto-calculated
   amount preview, save, today's entries list
5. Advance (Uchal) — give advance, running balance, full history
6. Monthly billing — automatic calculation per farmer per month
7. Payment — Cash/Bank/UPI, reference number, payment recorded against the bill
8. Reports — Daily / Monthly / Farmer-wise / Advance tabs
9. WhatsApp/SMS — generates the exact Marathi message template and opens
   WhatsApp (`wa.me` deep link) or the native SMS app pre-filled
10. Settings — milk rate (₹ per FAT point), local backup/restore (copies the
    SQLite file to app-external storage and back), language toggle (mr/en —
    UI string wiring for `en` is left for you to fill into `values-en` as the
    project grows), dark mode switch, PIN change

## Sample data
On first launch, `SeedData.kt` inserts 3 sample farmers, a default FAT-based
rate chart, two days of morning/evening milk entries, and one sample advance —
so every screen is demo-able immediately instead of opening empty.

## Notes / next steps for a production rollout
- Wire `darkMode` (Settings) into `MainActivity`'s `ShambhurajeDairyTheme(darkTheme = ...)`
  call via a `collectAsState()` so the toggle takes effect immediately.
- Add a real `values-en/strings.xml` if you want a full English UI toggle
  (current screens are hardcoded Marathi text per the brief; the `strings.xml`
  resources are provided as a starting point for extracting them).
- Add a `SyncRepository` using the Firebase dependencies already included,
  pushing any row where `isSynced = false` and pulling remote changes by `cloudId`.
- Replace the local file-copy backup/restore with Storage Access Framework
  (SAF) so operators can choose a folder (e.g. Google Drive-synced folder) —
  the hook point is `SettingsScreen.kt`.

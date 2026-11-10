# MMS – Android app (Kotlin + Jetpack Compose)

Native Android client for the **Medication Management System**. It talks to the same
CodeIgniter 3 REST API as the web version (`/api/...`), so both apps share one database,
one reminder engine and one set of caregiver alerts.

| | |
|---|---|
| Language / UI | Kotlin 2.0, Jetpack Compose, Material 3 (large text, high contrast) |
| Networking | Retrofit 2 + OkHttp + kotlinx.serialization |
| Local storage | Jetpack DataStore (server address, login token, offline answer queue) |
| Reminders | `AlarmManager` exact alarms + high-priority notifications with **Taken / Snooze / Skip** buttons + full-screen reminder over the lock screen |
| Background sync | WorkManager (every 15 min) + 20-second polling while the app is open |
| Min / target Android | 8.0 (API 26) / Android 15 (API 35) |

## 1. Open and run

1. Install **Android Studio** (Ladybug 2024.2 or newer) — it includes the JDK and Android SDK.
2. *File → Open…* → choose the `mms-android` folder. Wait for Gradle sync (first time downloads the libraries).
3. Start the backend (XAMPP Apache + MySQL, see the backend README) and the scheduler (`run_scheduler.bat`).
4. Press **Run ▶** on an emulator or on a phone with USB debugging.

### Server address
The login screen shows the server the app talks to; tap it to change:

| Where the app runs | Server address |
|---|---|
| Android **emulator**, XAMPP on the same PC | `http://10.0.2.2/mms/` (default) |
| **Real phone** on the same Wi-Fi as the PC | `http://<PC IP address>/mms/` e.g. `http://192.168.1.20/mms/` (find the IP with `ipconfig`) |
| Online server | `https://your-domain/` |

For a real phone, allow Apache through the Windows firewall (port 80). If you run the backend with
`php -S 0.0.0.0:8080 router.php`, use `http://<PC IP>:8080/`.

Log in with the demo accounts (`patient@demo.com` / `caregiver@demo.com`, password `password123`) after
running `php index.php cron seed` on the backend.

### Build an APK to install on phones
*Build → Build App Bundle(s)/APK(s) → Build APK(s)* → the file is `app/build/outputs/apk/debug/app-debug.apk`.
Copy it to a phone and open it (allow "Install unknown apps").

## 2. How reminders work on the phone (SRS 3.3)

```
Server (CodeIgniter)                         Phone (this app)
─────────────────────                        ─────────────────────────────────────────────
dose_events for today  ──  GET /api/doses/today ──►  ReminderScheduler sets exact alarms
                                                     (dose time + second-reminder time)
                                                            │ alarm fires (even offline / app closed)
                                                            ▼
                                                     AlarmReceiver → notification with
                                                     [Taken] [Snooze] [Skip] + full-screen screen
                                                            │ tap
POST /api/doses/{id}/take|snooze|miss  ◄──────────  DoseActionReceiver / ReminderActivity
stock −1, low-stock check, escalation                (offline? answer is queued and sent later
                                                     by OutboxWorker when back online)
reminder engine (every minute):
  2nd reminder, auto-missed, caregiver alerts ──► in-app notifications ──► SyncManager polls and
                                                                          shows them (caregivers too)
```

- Alarms are re-created after reboot (`BootReceiver`) and on every sync.
- On Android 12+ the patient should allow **Alarms & reminders** (Settings screen has a button) so reminders are exact.
- On Android 13+ the app asks for notification permission on first start.
- Caregivers receive alerts through background sync (every 15 minutes, instant while the app is open) and by SMS
  when the backend has an SMS gateway configured. For instant push while closed, add Firebase (section 5).

## 3. Screens

| Patient | Caregiver |
|---|---|
| Login / Register / Server address | Login / Register |
| **Home**: take-now cards (1 tap), today's schedule, 7/30-day adherence rings, stock levels, alerts | **Patients**: linked patients with today's progress, 7-day adherence, low stock, active alerts |
| **Reminder** (full screen, over lock screen) | **Patient detail**: call button, alerts (acknowledge), today, adherence, stock, medicines |
| **Medicines**: list, restock, edit, remove | **Patient history** |
| **Add / edit medicine** with time picker, every-day / some-days chips, start & end date | **Alerts**: acknowledge with a message to the patient |
| **History**: 7 / 30 / 90 days, per-medicine filter, chart, per-medicine rates, dose log | **Messages** (notification inbox) |
| **Caregivers**: link by email, remove | **Settings** |
| **Notifications**, **Settings** (profile, reminder timings, permissions, password, logout) | |

Every screen has an `@Preview` with sample data (`ui/SampleData.kt`): open the file in Android Studio and
use the **Split / Design** view to see it without running the app or the backend.

## 4. Code structure

```
app/src/main/java/com/mms/app/
├── MmsApp.kt                  Application: creates SessionStore, Repository, SyncManager; channels; periodic sync
├── MainActivity.kt            Hosts the Compose UI
├── data/
│   ├── Models.kt              @Serializable classes matching the API JSON
│   ├── ApiService.kt          Retrofit interface (all endpoints)
│   ├── Repository.kt          Builds Retrofit for the chosen server, adds Bearer token, readable errors
│   ├── SessionStore.kt        DataStore: server URL, token, user, offline queue
│   └── Time.kt                Date/time formatting helpers
├── reminders/
│   ├── ReminderScheduler.kt   Exact alarms for today's doses (first + second reminder, snooze)
│   ├── Receivers.kt           AlarmReceiver, DoseActionReceiver (notification buttons), BootReceiver
│   ├── Notifications.kt       Channels and notification builders
│   ├── ReminderActivity.kt    Full-screen "Time for your medicine" + ReminderContent composable
│   ├── SyncManager.kt         Poll server, reschedule alarms, show new alerts, flush offline answers
│   └── Workers.kt             SyncWorker (15 min) and OutboxWorker (send queued answers)
└── ui/
    ├── theme/Theme.kt         Colours (same as web), large typography, dark mode
    ├── components/            Reusable UI (cards, chips, due-dose card, adherence ring, bar chart…) + LoaderViewModel
    ├── navigation/AppNav.kt   Login vs main app, bottom navigation per role, in-app reminder pop-up
    ├── auth/                  Login, Register, Server address
    ├── patient/               Home, Medicines, Medicine form, History, Caregivers
    ├── caregiver/             Patients dashboard, Patient detail
    ├── common/                Alerts, Notifications, Settings
    └── SampleData.kt          Data for @Preview
```

## 5. Optional: Firebase push (FCM)
The backend can already send FCM pushes (`application/config/mms.php → mms_fcm_service_account`).
To receive them while the app is closed:
1. Create a Firebase project, add an Android app with package `com.mms.app`, download `google-services.json` into `app/`.
2. Add the `com.google.gms.google-services` plugin and `com.google.firebase:firebase-messaging` dependency.
3. Create a `FirebaseMessagingService`; in `onNewToken` call `POST /api/devices {token, platform:"android"}`
   (`ApiService.registerDevice`) and in `onMessageReceived` call `Notifications.showMessage(...)`.

## 6. Before release
- Use HTTPS and set `cleartextTrafficPermitted="false"` in `res/xml/network_security_config.xml` (SRS 5.3).
- Change the default server in `app/build.gradle.kts` (`DEFAULT_SERVER_URL`).
- Create a signing key: *Build → Generate Signed App Bundle / APK*.

## 7. Screen designs
`docs/screens/` contains PNG renders of every screen (patient and caregiver) at phone size, drawn from the
same layout, colours, text sizes and sample data as the Compose code (`docs/screens/render-designs.js`
generated them). They are design images for your report; for pixel-exact captures of the real app, run it
on an emulator and take screenshots, or use the `@Preview` panes in Android Studio.

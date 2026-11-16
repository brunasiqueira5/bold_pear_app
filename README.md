# Bold Pear

Mental well-being for astronauts — NASA Space Apps Challenge 2026.
Bold Pear, a 3D pear, guides a spoken check-in with the PHQ-9 and GAD-7 in Portuguese,
English, German, French and Japanese, and shows how the scores evolve on a dashboard.

## Run

```bash
flutter pub get
flutter run -d chrome        # web
flutter run                  # Android / iOS
```

Web build: `flutter build web --release --no-web-resources-cdn`

The app already points to the **Bold Pear** Supabase project (`omhfegwzorvrpscdftvg`, region sa-east-1).
To use another project:
`--dart-define=SUPABASE_URL=https://xxx.supabase.co --dart-define=SUPABASE_KEY=sb_publishable_...`

## Supabase

- `profiles` and `checkins` tables, with RLS: each person can only read and write their own data.
- A trigger creates the profile on sign-up (the name comes from `display_name`).
- SQL lives in `supabase/migrations/`.
- **Email confirmation:** new projects require confirming the email before the first login, and
  Supabase's default SMTP sends only a few emails per hour. For the demo, you can turn it off in
  Authentication → Sign In / Providers → Email → *Confirm email*, or set up your own SMTP.

## NASA dashboard (admin)

### Demo access

| Email | Password |
|---|---|
| `admin.nasa@nasa.com` | `Nasa2026` |

This account opens the **admin dashboard with every astronaut**: when you sign in with it, the
app goes straight to `/admin`, showing the indicators for the whole crew and the full record of
each astronaut (profile, check-ins, PHQ-9/GAD-7 scores, alerts and history). Access is
read-only. To see the app as an astronaut, create a regular account through sign-up.

### How it works

Accounts listed in `public.admins` go straight to the `/admin` dashboard (no pear, onboarding or
check-in). The dashboard shows the crew indicators (active, overdue, needing attention, item 9,
PHQ-9/GAD-7 averages, adherence, severity bands, check-ins per week, most reported symptoms) and
each astronaut's record (profile, statistics, alerts, trends, items and full history).

1. Run `supabase/migrations/20261115230000_admin_dashboard.sql` in the SQL Editor.
2. Create the account (sign up in the app or Authentication → Users → Add user).
3. Change the email in `supabase/grant_admin.sql` and run it in the SQL Editor — or, without
   opening the Supabase dashboard, with a [personal access token](https://supabase.com/dashboard/account/tokens):
   ```bash
   SUPABASE_ACCESS_TOKEN=sbp_... python scripts/grant_admin.py admin@example.com   # grant
   python scripts/grant_admin.py --revoke admin@example.com                        # revoke
   python scripts/grant_admin.py --list                                            # list
   ```

Dashboard tests: `flutter test test/admin_load_test.dart` (1,000 astronauts / ~15k
check-ins, API pagination, severe cases, and phone and desktop layouts).

Only the database decides who is an admin (`public.is_admin()` + RLS, read-only); the app cannot
promote itself.

## Neural voice

The pear's voice (Supertonic 3 int8 via sherpa-onnx, OpenRAIL-M, ~138 MB) **is not bundled in
the app or the repository**: it is downloaded on first launch, on a loading screen before the
language picker. The files come from the sherpa-onnx mirror on Hugging Face, at a pinned
revision (`Env.voiceModelUrl`); each one is checked by size and SHA-256, and the download
resumes where it stopped if the connection drops. If it fails, you can retry or continue with
the system voice (and download it later in Settings). On the web, the browser's voice is used.
Another source: `--dart-define=VOICE_MODEL_URL=https://.../folder`.

## Flow

0. First launch (mobile): download of the pear's voice, with a progress bar.
1. Welcome: language choice (the pear says hello), create an account or sign in.
2. Sign-up (name, email, password) or login; password recovery by email.
3. First access: the pear explains it is only a program and that nobody is listening live;
   choice of voice (female/male) and name; form of address in Portuguese and French.
4. Dashboard: next check-in (every 14 days), PHQ-9 and GAD-7 charts over the severity bands,
   history and guided breathing.
5. Spoken check-in: PHQ-9 → impact on daily life (if any symptom) → GAD-7. Questions are
   spoken; the screen shows only the options. Optional captions (CC button).
6. Result: severity bands, a safety notice if item 9 is positive, suggested practices.

The pear stays alive on every screen (`lib/app/shell.dart`): the same three.js HTML
(`assets/avatar/index.html`, offline) runs in a WebView on mobile and in an iframe on the web.

## Structure

```
lib/
  app/            session state, routes (go_router) and the frame with the pear
  core/           colors (pi_cores.dart), theme, strings in 5 languages, widgets
  features/
    auth/         welcome, login, sign-up
    profile/      onboarding, settings, profile repository
    checkin/      scales, scoring, spoken flow, result, breathing
    dashboard/    trends and history
    avatar/       3D pear (WebView / iframe) and its controller
    voice/        voice (TTS) that moves the pear's mouth and the captions
```

## Before real-world use

- Replace the GAD-7 texts (all languages) and PHQ-9 texts (DE, FR, JA) with the official validated translations.
- Choose neural voices per language and gender and pre-generate the audio (it currently uses the system voice).
- This is a well-being tool; it does not provide a diagnosis.

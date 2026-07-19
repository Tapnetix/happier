# Building the app to default to your self-hosted server

By default the Happier apps target Happier Cloud (`https://api.happier.dev`). This
guide bakes **`https://hdev.tapnetix.com`** in as the *default* server at build
time, so a device running your build connects to your server with no manual
"add server" step. (Users can still switch servers at runtime — this only changes
the default.)

Related: [`SELFHOST.md`](./SELFHOST.md) (server setup + stock-app runtime switching).

## How the default is resolved

The server URL is chosen in this order
(`apps/ui/sources/sync/domains/server/readConfiguredServerUrlEnv.ts`):

1. Web runtime config (`window.__HAPPIER_WEB_RUNTIME_CONFIG__.serverUrl`) — web only
2. `EXPO_PUBLIC_HAPPIER_SERVER_URL` (build-time)
3. `EXPO_PUBLIC_HAPPY_SERVER_URL` (legacy build-time)
4. `EXPO_PUBLIC_SERVER_URL` (generic build-time)
5. Web same-origin (web only)
6. Native fallback `HAPPIER_CLOUD_SERVER_URL = https://api.happier.dev`
   (`serverProfiles.ts`)

Setting **`EXPO_PUBLIC_HAPPIER_SERVER_URL`** at build time therefore wins over the
baked-in cloud default. `EXPO_PUBLIC_*` values are inlined by Metro/Expo at build
time — they must be present in the environment when the JS bundle is built, not at
runtime.

---

## Native — local build (no EAS/Expo account needed)

Best for sideloading onto your own devices. iOS device installs still require an
Apple Developer signing identity; Android APKs can be sideloaded freely.

Prerequisites: the standard Expo native toolchain (Xcode for iOS, Android
SDK/NDK + JDK for Android). From `apps/ui`:

### Android APK (sideloadable)

```bash
cd apps/ui

# Generate native projects with your server URL baked into the config:
EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com \
  yarn prebuild

# Build a release APK (env var must be set for the JS bundle step):
cd android
EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com \
  ./gradlew assembleRelease
# → android/app/build/outputs/apk/release/app-release.apk
```

Or build-and-run directly onto a connected device/emulator:

```bash
cd apps/ui
EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com \
  yarn android:production
```

### iOS (connected device / simulator)

```bash
cd apps/ui
EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com \
  yarn ios:production
```

For a distributable `.ipa` you need Apple signing; use the EAS path below or
archive the generated `ios/` project in Xcode with your team's provisioning.

> Verify after launch: Settings → Server should show `hdev.tapnetix.com` as the
> active server on first run (no manual add).

---

## Native — EAS cloud build (uses the committed `tapnetix` profile)

This fork ships dedicated EAS profiles in `apps/ui/eas.json` that bake in the
server URL and use a distinct app identity (`com.tapnetix.happier`,
scheme `happier-tapnetix`) so the build installs alongside the store app:

- `tapnetix` — store/internal build (AAB on Android)
- `tapnetix-apk` — Android APK for direct sideloading

Requirements: an Expo account (`npx eas-cli login`) and — for store distribution
or iOS — your own Apple/Google signing credentials configured in EAS. The profile
intentionally has **no `submit` entry**; submission targets are yours to add.

```bash
cd apps/ui

# Android APK via EAS:
npx eas-cli build --platform android --profile tapnetix-apk

# Android app bundle / iOS:
npx eas-cli build --platform android --profile tapnetix
npx eas-cli build --platform ios --profile tapnetix
```

To change the server URL, edit `EXPO_PUBLIC_HAPPIER_SERVER_URL` (and the legacy
`EXPO_PUBLIC_HAPPY_SERVER_URL`) in the `tapnetix` profile's `env` block.

---

## Web

The self-hosted **light server already serves the web UI same-origin** from
`https://hdev.tapnetix.com`, so browser users need nothing extra (resolver step 5).

Two options if you serve the web bundle separately from the API:

- **Build-time bake:** export with the env var set —
  ```bash
  cd apps/ui
  EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com \
    yarn ensure:workspace:built && \
    EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com \
    npx expo export --platform web --output-dir dist
  ```
  (This is what the Docker `webapp` target does via its
  `EXPO_PUBLIC_HAPPIER_SERVER_URL` build arg.)
- **Runtime bake (no rebuild):** have the page define the global before the app
  bundle loads — highest precedence, useful for reusing one prebuilt bundle:
  ```html
  <script>
    window.__HAPPIER_WEB_RUNTIME_CONFIG__ = { serverUrl: "https://hdev.tapnetix.com" };
  </script>
  ```

---

## Notes

- Runtime switching still works: any build can add/switch servers in
  Settings → Server or by scanning the CLI connect QR. This recipe only sets the
  *default*.
- Changing the source constant `HAPPIER_CLOUD_SERVER_URL` (and the `app.happier.dev`
  webapp fallbacks) is a full rebrand and is intentionally **not** done here — the
  env-var approach above keeps this fork cleanly rebaseable on upstream.

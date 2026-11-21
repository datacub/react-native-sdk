<p align="center">
  <a href="https://datacub.to">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/datacub/react-native-sdk/main/.github/assets/wordmark-dark.svg">
      <img alt="datacub" src="https://raw.githubusercontent.com/datacub/react-native-sdk/main/.github/assets/wordmark.svg" height="48">
    </picture>
  </a>
</p>

<p align="center">
  The official datacub SDK for React Native and Expo apps.<br>
  Product analytics with screens, identity and consent. No native code, no device identifiers.
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/@datacub/react-native"><img alt="npm (next)" src="https://img.shields.io/npm/v/@datacub/react-native/next?color=15181d&label=npm%40next"></a>
  <a href="https://github.com/datacub/react-native-sdk/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/datacub/react-native-sdk/actions/workflows/ci.yml/badge.svg"></a>
  <a href="./LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-15181d"></a>
</p>

> [!IMPORTANT]
> **Beta.** `@datacub/react-native` is published on the `next` dist-tag while we finish testing on real iOS and Android devices. The API is small and we expect to keep it, but it may still change before 1.0. Every change is tested on the Hermes engine and through Metro bundles for iOS and Android (see [Supported environments](#supported-environments)). Please [open an issue](https://github.com/datacub/react-native-sdk/issues) if anything does not work in your app.

---

[datacub](https://datacub.to) answers product questions for websites and apps: where people come from, what they do, where they drop off and who comes back. This SDK sends screen views, sessions and the events you track from your app, removes sensitive data on the device, and delivers small batches to datacub. For websites, use [`@datacub/web`](https://github.com/datacub/web-sdk).
- **Pure JavaScript.** No native modules, no config plugin, no prebuild. About 6 KB gzip, no runtime dependencies.
- **Private by construction.** No device identifiers (no IDFA, IDFV or Android ID), no location, no screen text. The install ID is random.
- **Built for mobile networks.** Offline queue that survives restarts, batching, retries with backoff and delivery when the app goes to the background.

## Quick start

You need a datacub project. In the dashboard, open **Settings → Integrations** and copy your project key (`pk_live_…`; `pk_test_…` also works in Expo Go and development builds). In **Settings → General**, add your app's bundle ID or package name to the apps allowed to send.

### 1. Install

```sh
npx expo install @datacub/react-native@next @react-native-async-storage/async-storage expo-application
```

In a bare React Native app, install `@datacub/react-native@next` and `@react-native-async-storage/async-storage` with your package manager; `expo-application` is optional there.

### 2. Initialise

Once, at startup (for example in `App.tsx` or Expo Router's root `_layout.tsx`):

```ts
import AsyncStorage from '@react-native-async-storage/async-storage'
import * as Application from 'expo-application'
import datacub from '@datacub/react-native'

datacub.init('pk_live_YOUR_PROJECT_KEY', { storage: AsyncStorage, app: Application })
```

`storage` keeps the anonymous ID and unsent events between launches. `app` supplies your app ID and version; in a bare app pass `{ applicationId: 'com.example.app', nativeApplicationVersion: '1.4.0' }` yourself.

### 3. Send an event and see it

```ts
datacub.screen('Home')
datacub.track('signup_completed', { plan: 'pro' })
```

Open your project's live view in datacub: the screen view and `signup_completed` appear within seconds. Events are sent in batches every 10 seconds, at 20 events, or when the app goes to the background; call `await datacub.flush()` to send at once.

## Usage

### Screens

With React Navigation (Expo Router uses it too):

```tsx
import { NavigationContainer, useNavigationContainerRef } from '@react-navigation/native'
import { trackNavigation } from '@datacub/react-native/navigation'

export default function App() {
  const ref = useNavigationContainerRef()
  return (
    <NavigationContainer ref={ref} onReady={() => trackNavigation(ref)}>
      {/* your navigators */}
    </NavigationContainer>
  )
}
```

With Expo Router, in the root `_layout.tsx`. Send the route template, never the pathname, which holds the values (`profile/[user]`, not `profile/sara`):

```tsx
import { Stack, useSegments } from 'expo-router'
import { useEffect } from 'react'
import datacub from '@datacub/react-native'

export default function RootLayout() {
  const segments = useSegments()
  useEffect(() => datacub.screen(segments.join('/') || 'index'), [segments])
  return <Stack />
}
```

Anywhere else, call `datacub.screen('Checkout')` when a screen shows. Name screens from your code, never from what the screen displays. Segments that look like IDs, emails or numbers are replaced (`Product/4512` becomes `Product/:id`), and the same name twice in a row counts once.

### Track events

```ts
datacub.track('invoice_sent', { amount: 1200, currency: 'SAR', items: 3 })
```

Name events in `snake_case` after something that happened. Property values can be strings, numbers, booleans, `null`, dates, arrays and nested objects. On the device, before anything is stored or sent:
- properties whose **names** look sensitive (`password`, `token`, `cardnumber`, `iban`, `otp` and similar) are dropped;
- **values** that look sensitive become `"[redacted]"`: emails, phone, card, IBAN and ID numbers, tokens and keys, whole numbers of 9 or more digits, token-like runs of 32 or more characters, and strings over 256 characters or 11 words.

Pass times as `Date` objects, not millisecond numbers, which would be redacted.

### Identify signed-in users

```ts
// after login
datacub.identify('user_8412', { account: 'acct_77', email: 'sara@example.sa', name: 'Sara' })

// at logout, so the next person on this phone starts fresh
datacub.reset()
```

`identify` links this install's earlier anonymous activity to the user. `email`, `name` and `phone` (strings of up to 256 characters) are shown on the user's profile and are searchable; they are never used to link two people. User IDs that look like an email, phone, ID or card number are refused. Calling `identify` again for the same user sends only the traits, so pass them at login or when they change. Signing in as a different user resets first, so two accounts on one phone are never linked.

### Consent

By default tracking starts at `init`. If your app asks first, turn on **Ask for consent first** in the project settings and pass `requireConsent: true`. Nothing is stored or sent until you grant:

```ts
datacub.init('pk_live_YOUR_PROJECT_KEY', {
  storage: AsyncStorage,
  app: Application,
  requireConsent: true,
})

// at every launch, from your app's own record of the person's answer
if (await hasAnalyticsConsent()) datacub.consent.grant({ method: 'settings' })
```

`datacub.consent.revoke()` always works: it deletes the stored IDs and every queued event, and is remembered on the device so the next launch does not start tracking again. A later `grant()` clears it.

## API reference

| Call                                                           | What it does                                                                                             |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `datacub.init(publicKey, options?)`                            | Starts the SDK. Runs once; later calls are ignored                                                       |
| `datacub.screen(name, properties?)`                            | A screen view (`$screen`)                                                                                |
| `datacub.track(event, properties?)`                            | A custom event                                                                                           |
| `datacub.identify(userId, traits?)`                            | Links this install to a signed-in user; traits `account`, `email`, `name`, `phone` and custom properties |
| `datacub.reset()`                                              | At logout: a new anonymous ID and session                                                                |
| `datacub.flush()`                                              | Sends what is queued now. Resolves when done or failed; never rejects                                    |
| `datacub.consent.grant(options?)`                              | Records consent (`method`, `policyVersion`) and starts tracking when the project asks for consent        |
| `datacub.consent.revoke()`                                     | Stops tracking, deletes stored IDs and queued events, and remembers the choice                           |
| `trackNavigation(ref)` from `@datacub/react-native/navigation` | Sends a screen view on every React Navigation route change; returns an unsubscribe function              |

### Options

| Option           | Default                 | What it does                                                                                                                                                                                                                |
| ---------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `storage`        | none (memory only)      | Anything with AsyncStorage's `getItem`, `setItem` and `removeItem`, sync or async (an MMKV wrapper works). Without it every launch is a new anonymous ID and unsent events are lost when the app closes; the SDK warns once |
| `app`            | none                    | `{ applicationId, nativeApplicationVersion?, nativeBuildVersion? }`; `expo-application` has this shape. Needed for the allowed-apps check                                                                                   |
| `requireConsent` | `false`                 | Store and send nothing until `datacub.consent.grant()`                                                                                                                                                                      |
| `apiHost`        | `https://in.datacub.to` | Your ingestion host or proxy. Must be `https`; plain `http` is accepted only for `localhost` and private network addresses, for development (do not ship it)                                                                |

## What the SDK sends

Only what your code asks for: screen views, your events, `identify` and consent records. There are **no automatic lifecycle events** (opened, backgrounded, installed), because they would inflate active users and sessions.

Each event carries an app context:

| Field                      | From                                                                       |
| -------------------------- | -------------------------------------------------------------------------- |
| `platform`                 | `ios` or `android`                                                         |
| `os_version`               | iOS `Platform.Version`; Android `Platform.constants.Release`               |
| `device_class`             | `tablet` when the screen's short side is 600 points or more, else `mobile` |
| `locale`                   | `Intl`                                                                     |
| `app_version`, `app_build` | the `app` option                                                           |
| `screen`                   | the screen on show                                                         |

Plus a random install ID (`anon_id`), a session ID (30 minutes idle, 24 hours at most), your user ID and `account` after `identify`, and the app ID in the batch. Technical fields ride along for correct counting: a per-install sequence number, the consent state, whether storage persists, the previous screen, `$resumed` on a screen re-sent after a long background, counts of dropped events, and the SDK name, version and timestamps. `identify` adds the `email`, `name` and `phone` you passed.

**Never sent:** device identifiers (IDFA, IDFV, Android ID, serial number, build fingerprint), device model, carrier, location, contacts, screen size, time zone, screen text or route parameters. Raw IP addresses are used at the ingestion edge and never stored.

The SDK stores one value, under the key `datacub_state` in the storage you pass: the IDs, the consent record (only an opt-out flag after a revoke) and the unsent queue. Until it is sent, a queued `identify` holds the email, name and phone you passed, in the app's storage. The queue is capped at 1,000 events and 1 MB, and events older than 29 days are dropped.

### Android backup

Android Auto Backup restores app data on a new phone, so a restored install keeps its anonymous ID (and any unsent events). To give every install its own ID, exclude AsyncStorage's database (`RKStorage`) in `dataExtractionRules`, or set `android.allowBackup: false` in Expo. A restore is usually the same person, so this is a choice, not a fix.

## Supported environments

| Environment       | Support                                                                                |
| ----------------- | -------------------------------------------------------------------------------------- |
| React Native      | 0.75 and later, iOS and Android. New and old architecture (the SDK has no native code) |
| Expo              | SDK 52 and later, in Expo Go, development builds and production builds                 |
| JavaScript engine | Hermes (tested). JavaScriptCore should work but is not tested                          |
| React Native Web  | Not supported; use [`@datacub/web`](https://github.com/datacub/web-sdk)                |
| Navigation        | React Navigation 6 and 7, Expo Router, or manual `screen()` calls                      |
| TypeScript        | Types included for ESM and CommonJS                                                    |

**How it is tested.** On every change, CI runs unit tests for identity, sessions, events, consent and delivery against a stubbed React Native; compiles the published files with the Hermes compiler of React Native 0.75 and of the latest release; runs them in the Hermes VM; and bundles the packed tarball with Metro for iOS and Android, with package exports on and off. React Native 0.72 to 0.74 are not supported because their Hermes compiler rejects the Unicode regular expressions the privacy scrubber needs.

**Not yet tested:** runs on physical iOS and Android devices and simulators. That is the remaining step before the SDK leaves beta.

## Versioning and releases

Beta versions (`0.1.0-beta.N`) are published to the `next` dist-tag; `npm install @datacub/react-native` without `@next` will install the first stable version once it exists. Each version's changes are in the [changelog](./CHANGELOG.md) and on the [releases page](https://github.com/datacub/react-native-sdk/releases). Every npm version is published from GitHub Actions with an npm provenance attestation.

## Contributing

Issues and pull requests are welcome. See [CONTRIBUTING.md](./CONTRIBUTING.md) for setup and guidelines, and [RELEASING.md](./RELEASING.md) for how releases work. Please report security problems privately as described in [SECURITY.md](./SECURITY.md).

## License

[MIT](./LICENSE) © datacub

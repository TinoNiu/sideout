# Sideout — Technical Specification

**Version:** 1.0 (v1 launch)
**Stack:** Flutter (Dart) · Isar (local DB) · Supabase (cloud sync, v2) · RevenueCat (purchases)
**Platforms:** iOS 14+ · Android 8+ (API 26+)
**Last updated:** Living document — update as decisions change

---

## 1. Product principles

Sideout is a journaling app for volleyball players. These principles arbitrate every product decision:

1. **The user's words are sacred.** Their entries are encrypted, never sent to servers in plaintext, never trained on, never shown to anyone else.
2. **The app is a teammate, not a coach.** Voice is observational, not directive. Questions over commands.
3. **Polish > features.** A small set of features that feel exquisite beats a large set that feel generic.
4. **Sport-aware, not sport-themed.** The vocabulary, prompts, and patterns reflect volleyball specifically. We are not a recolored generic journal.

If a feature violates one of these, cut it.

---

## 2. Feature inventory

### Free tier (v1 launch)

- Unlimited journal entries with rich text (bold, italic, highlights)
- Mood tagging per entry (5-point scale, color-coded)
- Free-form text tags (volleyball vocabulary suggested)
- Basic prompt library: ~30 hand-written prompts categorized by context (post-match, pre-match, training, rest day, breakthrough)
- "Today's prompt" home screen card with smart selection based on most recent entry timing
- Search across all entries (text content, tags, dates)
- Local notifications: optional daily prompt at user-chosen time
- Calendar view of entries
- Light and dark mode (auto, manual override available)
- Local-only storage with on-device encryption at rest
- Export all entries to plaintext or markdown file

### Pro tier (one-time purchase, $19.99 USD launch price)

- **Pattern detection:** local on-device analysis surfaces recurring themes ("'serve confidence' appears in 9 entries this month")
- **Insights screen:** mood trends, tag frequency, journaling streaks, "your strong weeks" analysis
- **Cloud sync:** end-to-end encrypted, opt-in, multi-device
- **Advanced prompts library:** ~150 prompts including pre-tournament, slump, comeback, transition, leadership themes
- **Custom prompt creation:** users add their own prompts to rotation
- **Custom tag colors and categories**
- **PDF export:** formatted, dated, with mood/tag indicators — useful for sports psychologists
- **Match-day intelligence:** detects upcoming match from calendar entries, surfaces relevant past entries

### Explicitly NOT in v1 (deferred)

- Social features, sharing, public profiles — never. This is a private journal by design.
- Coach access, team features — would compromise the privacy promise.
- Voice journaling — adds complexity, can be v2+.
- Photo/video attachments — significant storage and sync cost; defer.
- Apple Watch / Wear OS apps — defer to v2.
- Stat tracking (kills, digs, sets) — out of scope; this is a thought journal, not a stats tracker.
- AI/LLM features — strong reason to be cautious here. If we add them ever, they must run on-device only. No sending entries to OpenAI or Anthropic APIs.

---

## 3. Data model

### Entry

```dart
class Entry {
  Id id;                          // Isar auto-generated
  DateTime createdAt;             // Immutable, set on creation
  DateTime updatedAt;             // Updated on every edit
  String title;                   // User-editable, may be empty
  String bodyJson;                // flutter_quill Delta format as JSON string
  String bodyPlain;               // Plain text version, for search index
  Mood mood;                      // enum: great, good, neutral, rough, hard
  List<String> tags;              // Free-form, lowercase normalized
  EntryContext? context;          // enum: postMatch, preMatch, training, restDay, other
  String? promptId;               // If entry was started from a prompt
  bool isPinned;                  // For "anchors" feature in pro
}

enum Mood { great, good, neutral, rough, hard }
enum EntryContext { postMatch, preMatch, training, restDay, breakthrough, other }
```

### Prompt

```dart
class Prompt {
  String id;                      // Stable identifier, e.g. "post_match_001"
  String text;                    // The question itself
  EntryContext context;
  PromptTier tier;                // free or pro
  List<String> themes;            // ["serve", "confidence", "team"]
}

enum PromptTier { free, pro }
```

Prompts ship as a JSON file in assets, loaded once at app start. They do not need to be in Isar.

### UserSettings (single row)

```dart
class UserSettings {
  Id id = 0;                      // Always 0, single row
  ThemeMode themeMode;            // light, dark, system
  bool notificationsEnabled;
  TimeOfDay? dailyPromptTime;     // null if disabled
  String? userName;               // For greeting on home screen
  bool hasCompletedOnboarding;
  String? proLicenseKey;          // RevenueCat user identifier
  bool cloudSyncEnabled;          // v2 only
}
```

### Encryption

The entry's `bodyJson`, `bodyPlain`, `title`, and `tags` fields must be encrypted at rest. Isar does not encrypt by default.

**v1 approach:** Use Isar's `encryptionKey` parameter. The 64-byte key is generated on first launch and stored in `flutter_secure_storage` (which uses iOS Keychain / Android Keystore — hardware-backed where available). If the secure storage entry is lost (user deletes the app), entries are unrecoverable. This is the correct privacy behavior.

**v2 cloud sync approach:** Each entry is re-encrypted with a key derived from the user's master passphrase before being sent to Supabase. Server stores only ciphertext + metadata (created_at, updated_at, id). Use libsodium via `cryptography` package for the key derivation (Argon2id) and encryption (XChaCha20-Poly1305).

---

## 4. Screen inventory

Every screen we designed in the brand work, plus the ones needed to actually ship.

### Onboarding (first launch)
1. Welcome — splash arc animation, brand intro
2. Name input — "What should we call you?" (optional, skippable)
3. Notification permission — soft ask with explanation before the system dialog
4. Privacy commitment — "Your entries stay on this device. Always."
5. First prompt — drops them straight into writing their first entry

### Main app
1. **Home** — today's prompt card, recent entries list, FAB to write new entry
2. **Entry composer** — title, body (rich text via flutter_quill), mood selector, tag chips, save/cancel
3. **Entry reader** — read-only view with edit, delete, share-as-text actions
4. **Library** — all entries in reverse chronological order, with filters: mood, tag, date range, context
5. **Calendar** — month view with entry indicators per day, tap to see that day's entries
6. **Search** — full-text search with highlighting, tag autocomplete
7. **Insights (pro)** — mood trends, top tags, recurring themes, streak counter
8. **Settings** — theme, notifications, name, export, pro upgrade, about, privacy policy

### Pro upgrade flow
9. **Paywall** — feature comparison, single CTA, "restore purchase" link
10. **Post-purchase** — thank you screen, brief tour of unlocked features

---

## 5. Dependencies (`pubspec.yaml`)

```yaml
name: sideout
description: A volleyball journal.
version: 0.1.0+1

environment:
  sdk: ">=3.2.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter

  # State management
  flutter_riverpod: ^2.4.9

  # Navigation
  go_router: ^13.0.0

  # Local storage (encrypted)
  isar: ^3.1.0
  isar_flutter_libs: ^3.1.0
  path_provider: ^2.1.1
  flutter_secure_storage: ^9.0.0

  # Rich text editor
  flutter_quill: ^9.0.0

  # Notifications
  flutter_local_notifications: ^16.3.0
  timezone: ^0.9.2

  # Purchases
  purchases_flutter: ^6.6.0     # RevenueCat

  # Utilities
  intl: ^0.19.0                  # Date/time formatting
  uuid: ^4.2.2
  shared_preferences: ^2.2.2     # For non-sensitive prefs
  package_info_plus: ^5.0.1

  # Analytics & monitoring (privacy-preserving)
  sentry_flutter: ^7.16.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1
  build_runner: ^2.4.7
  isar_generator: ^3.1.0

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/prompts.json
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
        - asset: assets/fonts/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
```

For v2 cloud sync, add: `supabase_flutter` and `cryptography`.

---

## 6. Architecture

### Folder structure

```
lib/
  main.dart                    Entry point, app bootstrap
  app.dart                     MaterialApp config, theme switching, router setup
  theme/
    colors.dart                Brand color tokens (single source of truth)
    text_styles.dart           Typography scale
    theme_data.dart            ThemeData for light + dark
  models/
    entry.dart                 Isar collection
    prompt.dart                Plain Dart class, loaded from JSON
    user_settings.dart         Isar collection (single row)
    enums.dart                 Mood, EntryContext, PromptTier
  services/
    isar_service.dart          DB initialization, instance access
    secure_storage_service.dart  Encryption key management
    notification_service.dart  Local notifications
    purchase_service.dart      RevenueCat wrapper
    prompt_service.dart        Loads prompts.json, smart selection
    pattern_service.dart       Pro: tag/theme frequency analysis
  providers/                   Riverpod providers
    entry_providers.dart
    settings_providers.dart
    purchase_providers.dart
  screens/
    home/
    composer/
    reader/
    library/
    calendar/
    search/
    insights/
    settings/
    onboarding/
    paywall/
  widgets/                     Reusable widgets
    arc_logo.dart
    mood_picker.dart
    tag_chip.dart
    entry_card.dart
    prompt_card.dart
  utils/
    date_helpers.dart
    text_helpers.dart
```

### State management pattern

Use Riverpod with code generation off (simpler for learning). The pattern:

- `Provider` for read-only services (Isar instance, notification service)
- `StateNotifierProvider` for mutable app state (entry list, current settings)
- `FutureProvider` for async one-shot data (load prompts on app start)
- `StreamProvider` for reactive data (Isar query streams that auto-update)

Keep widgets dumb. Business logic lives in services and notifiers.

### Navigation

`go_router` with a flat, named route structure:

```
/                       -> HomeScreen
/compose                -> EntryComposerScreen (new)
/compose/:id            -> EntryComposerScreen (edit)
/entry/:id              -> EntryReaderScreen
/library                -> LibraryScreen
/calendar               -> CalendarScreen
/search                 -> SearchScreen
/insights               -> InsightsScreen (paywall-gated)
/settings               -> SettingsScreen
/settings/about         -> AboutScreen
/onboarding             -> OnboardingFlow
/paywall                -> PaywallScreen
```

---

## 7. Build phases

### Phase 1 — Foundation (4-6 weeks)

**Goal:** A working app you can write entries in. No prompts, no notifications, no insights. The plumbing.

Tasks:
- Project setup, pubspec.yaml, folder structure
- Theme system (colors, typography, light/dark switching)
- Isar setup with encryption
- Riverpod boilerplate
- go_router configuration
- Home screen with empty state
- Entry composer with flutter_quill
- Entry reader
- Library screen (list of entries)
- Local persistence working end-to-end

**Done when:** You can write an entry on your phone, kill the app, reopen it, and the entry is still there. Looks like Sideout, not a default Flutter app.

### Phase 2 — Sport intelligence (3-4 weeks)

**Goal:** This is where Sideout becomes Sideout instead of a generic notes app.

Tasks:
- Mood selector widget and persistence
- Tag system: input chips, autocomplete, normalization
- Prompt library JSON (write all 30 free prompts)
- Prompt service with smart selection logic
- "Today's prompt" home card
- Search screen with full-text search across entries
- Calendar view
- Settings screen (theme, name)

**Done when:** A real volleyball player can use it for a week and the prompts feel like they understand the sport.

### Phase 3 — Polish and pro (2-3 weeks)

**Goal:** The app feels like a finished product, and pro is purchasable.

Tasks:
- Splash animation (arc draw)
- All micro-animations (entry card slide-in, mood selector haptics, tag chip animations)
- Local notifications with daily prompt
- Onboarding flow
- RevenueCat integration
- Paywall screen
- Insights screen (mood trends, tag frequency)
- Pattern detection service
- PDF export
- Sentry error reporting
- Privacy policy and terms

**Done when:** You'd be willing to show it to a stranger.

### Phase 4 — Beta (3-4 weeks)

**Goal:** Real people use it, you fix what they break.

Tasks:
- Codemagic CI setup for iOS builds
- Apple Developer account + provisioning profiles
- Google Play Developer account + signing keys
- TestFlight build
- Google Play internal testing track
- 10-20 beta testers (volleyball teammates, friends, online communities)
- Triage feedback into "must fix before launch" vs "v1.1"
- Crash fixing
- Performance profiling
- Accessibility audit (VoiceOver, TalkBack)

**Done when:** Beta cohort uses it for 2 weeks without major complaints.

### Phase 5 — Launch (2-3 weeks)

**Goal:** Live in both stores.

Tasks:
- App Store screenshot generation (use the frames we designed)
- App Store description, keywords, category selection
- Google Play store listing
- Landing page on `sideoutjournal.com`
- App Store and Play Store submissions
- Inevitable rejection cycle (budget for 1-2 rounds)
- Soft launch announcement
- First-day monitoring (crash reports, reviews, sign-ups)

**Done when:** Search "sideout journal" on either store, find Sideout, install it.

---

## 8. Costs (one year of operation, conservative)

| Item | Cost |
|------|------|
| Apple Developer Program | $99/year |
| Google Play Developer | $25 (one-time) |
| Domain (sideoutjournal.com) | ~$15/year |
| Codemagic CI (free tier likely sufficient) | $0 |
| Sentry (free tier) | $0 |
| RevenueCat (free under $2.5k MRR) | $0 |
| Supabase (free tier covers v1, v2 needs paid) | $0 (v1) / $25/mo (v2) |
| Privacy policy generator | $0 (write yourself) |
| **Total v1 launch** | **~$140 first year** |

---

## 9. Risks and mitigations

**Risk:** Apple rejects the app for vague reasons.
**Mitigation:** Read the App Store Review Guidelines once before submitting. Have a clear privacy policy. Ensure the app does something useful before any paywall (free tier must be usable).

**Risk:** Encryption implementation has a flaw.
**Mitigation:** v1 uses Isar's built-in encryption — battle-tested. v2 sync uses libsodium via `cryptography` — also battle-tested. Do not roll your own crypto. Get the v2 design reviewed before shipping it.

**Risk:** Burnout from learning Flutter while building.
**Mitigation:** Phase 1 is intentionally slow and small. Build the home screen three different ways before moving on if needed. Building badly is faster than reading endless tutorials.

**Risk:** Scope creep.
**Mitigation:** This document is the contract. New feature ideas go to a separate `BACKLOG.md`. Nothing is added to v1 after Phase 2.

**Risk:** Indie app gets zero downloads.
**Mitigation:** Real one. The brand work helps. Beta cohort plus volleyball Reddit/Discord communities give a soft floor. Don't expect overnight numbers — journaling apps are a slow-burn category.

---

## 10. Success criteria for v1

These are the bar Sideout needs to clear to be considered a successful v1, in priority order:

1. Ships to both stores.
2. 95%+ crash-free sessions in first month.
3. At least 50 users journal more than 5 entries each (real engagement, not just downloads).
4. At least 5 pro purchases in first month.
5. Average App Store rating ≥ 4.0.

If 1-3 are met, v1 is a success regardless of monetization. The product is working.

# Architecture — Android Expense Tracker

Companion to [plan.md](plan.md). Describes how version 1 will work. This is a design document; implementation has not started.

Structure adapted from the [reference architecture](https://github.com/alexeygrigorev/retroloop/blob/711c33cc5b6ba44c66d994f3bfc9bd5a6915172e/_docs/architecture.md).

## Stack

| Concern | Choice | Reason |
|---|---|---|
| Language | TypeScript | Share types and validation across app and backend |
| Android app | React Native + Expo development builds | Reuse React skills; access camera, microphone, and SQLite |
| Navigation | Expo Router | File-based routes and three main tabs |
| Local database | SQLite through expo-sqlite | Transactions, reporting queries, offline use |
| Media | Expo audio, camera/image picker, image manipulation, file system | Capture and prepare uploads |
| Charts | Small SVG-based pie chart using react-native-svg | Keep dependencies and chart scope small |
| Backend | Node.js LTS + Express | Familiar stack; keep provider credentials on the server |
| Validation | Zod | Validate API requests and extracted expenses |
| AI | OpenAI transcription, text extraction, and vision | One provider for the prototype |
| API protection | Firebase App Check with Play Integrity | App attestation without user login |
| Android distribution | EAS Build, production Android App Bundle | Device-specific Play Store delivery |
| Tests | Jest for app/domain logic; Vitest + Supertest for API | Check money, dates, budgets, and API boundaries |

Select an Expo-supported React Native/React combination at setup and pin package versions in the lockfile. Exact AI models, package versions, prices, and hosting provider remain unverified. Keep model IDs in server configuration and select them through accuracy/cost tests.

**Download size target: under 30 MB; stretch goal: 20 MB.** Measure device-specific production downloads early. Installed size and local data usage are separate measurements. Do not bundle AI models.

## System boundaries

```text
Android phone                         Processing service
+---------------------------+        +---------------------------+
| Expo / React Native       | HTTPS  | Express API               |
|                           |------->| App Check + limits        |
| Expenses / Charts / Budget|        | Input/output validation   |
| Domain services           |<-------| OpenAI adapter            |
| SQLite                    | JSON   +-------------+-------------+
| Temporary capture files   |                      |
+---------------------------+                      v
                                            OpenAI APIs
```

The phone owns expenses, categories, and budgets. Express processes one upload and returns suggestions. It does not store an expense history or expose expense CRUD endpoints.

The phone can handle manual entry, editing, deletion, budgets, charts, and CSV exports offline. Voice and image extraction require internet. Users must understand that the app sends voice/images and category names to external services.

Version 1 needs no MongoDB, account system, object store, or background queue. Keep the first backend deployment to one instance so its in-memory rate counters have a defined scope.

## Repository and modules

```text
apps/
  mobile/
    app/                    Expo Router tabs, forms, review screens
    src/
      features/
        expenses/           list, manual entry, edit/delete
        capture/            voice recording, camera/gallery, review
        charts/             monthly category totals and pie chart
        budgets/            monthly totals and category allocations
        export/             CSV generation and Android sharing
      domain/               money, dates, budget rules, save commands
      db/                   SQLite connection, migrations, repositories
      api/                  uploads, App Check tokens, response decoding
      platform/             permissions, files, app lifecycle
  api/
    src/
      routes/               voice and image endpoints
      middleware/           App Check, rate limits, validation, errors
      services/             media processing and extraction orchestration
      ai/                   provider interface and OpenAI implementation
      config/               validated server configuration
packages/
  contracts/                portable schemas, DTOs, error codes
```

Screens call domain services; domain services call repositories. Keep SQL and provider calls out of React components. SQLite holds durable state; React state holds filters, forms, and unsaved review entries. Refresh affected queries after a commit and when returning to a tab.

The shared contracts package must not import server credentials, Firebase Admin, or Node-only code into the mobile bundle.

## Local data model

Use UUID strings for entity IDs, UTC timestamps for audit fields, integer paise for money, and `YYYY-MM-DD` for expense dates. Reject nonpositive expenses and negative budgets in version 1; refunds remain outside scope.

```text
Category
  id, name, normalized_name UNIQUE, is_default, created_at, updated_at

Expense
  id, name, amount_paise > 0, expense_date, category_id -> Category
  source {MANUAL, VOICE, IMAGE}
  capture_id nullable, source_item_key nullable
  created_at, updated_at
  UNIQUE(capture_id, source_item_key)

MonthlyBudget
  id, month_key UNIQUE             # YYYY-MM
  total_paise >= 0, created_at, updated_at

CategoryBudget
  id, monthly_budget_id -> MonthlyBudget, category_id -> Category
  amount_paise >= 0, created_at, updated_at
  UNIQUE(monthly_budget_id, category_id)

ImageScan
  id, image_sha256, capture_id UNIQUE, confirmed_at
```

`image_sha256` is indexed, not unique: a user may choose to import a duplicate. Keep only the fingerprint, not a durable receipt image. Match fingerprints from the selected original bytes before image preprocessing. This detects exact-file duplicates; crops, recompression, or new photos may evade it. Describe warnings as possible duplicates.

Enable SQLite foreign keys. Index expense dates and category/date queries. Keep migrations numbered and transactional. Use parameterized SQL and database constraints alongside domain validation.

Query all months from one Expense table using `[month start, next month start)`. Do not create a table per month. Join categories when presenting names.

A custom category remains available for future entries and budgets. AI chooses existing category IDs; the user creates new categories. Use Other when classification is uncertain. The final default category list needs review.

## Save rules and transactions

Validate data on the server and again on the phone before saving. The phone verifies category IDs against its current categories, since those may change during processing.

Generate a capture ID on the phone before upload. Give extracted items stable keys within that capture. Enforce the compound unique key when saving so a double tap or a delayed response cannot insert the same item twice.

Save confirmed image items and the ImageScan record in one transaction. For voice, save the complete eligible items together; retain incomplete candidates in a temporary form for the user to finish. Do not place incomplete entries in expense totals.

Changing or deleting an expense updates reporting through fresh SQLite queries. Compute totals from expenses instead of storing a second copy of spending totals.

## Budget calculations and monthly rollover

For a selected month:

```text
allocated          = sum(category budget amounts)
unallocated        = monthly total - allocated
monthly spent      = sum(all expenses in the month)
monthly remaining  = monthly total - monthly spent
category remaining = category allocation - category spending
overspent          = max(0, spent - planned)
```

Show negative remaining values and exact overruns. Expenses in unbudgeted categories still count toward monthly spending; label the category “No budget set.” Unallocated is a planning amount, not an extra pool excluded from spending.

Apply total and category budget edits in one transaction. If allocations exceed the proposed total, preserve the user's edits and ask them to increase that total before saving. Allow a smaller total only when it still covers category allocations.

Android may suspend the app at a month boundary. Implement automatic rollover when the app starts, resumes, or opens a month: an idempotent `ensureMonthBudget()` copies the previous month's plan once. A unique month key prevents duplicate initialization. Resolve skipped months in order using the last available plan; create zero/unset values if no earlier plan exists. Treat this skipped-month policy as a proposed default.

Copy allocations and the total, but not expenses or remaining balances. Past expense dates continue to select past months. A budget edit changes only the selected month's plan.

## Capture state machines

```text
Voice:
IDLE -> RECORDING -> UPLOADING -> PROCESSING
     -> SAVE_COMPLETE_ITEMS -> DONE
                           -> MISSING_DETAILS -> SAVE_COMPLETED_ITEM

Image:
IDLE -> CAMERA_OR_GALLERY -> DUPLICATE_CHECK
     -> UPLOADING -> PROCESSING -> REVIEW -> CONFIRMED -> SAVED
```

Allow cancellation and show recoverable errors during upload/processing. Image review offers edit/delete/add and a maximum of 20 entries. Confirm once and save the batch transactionally.

Stop voice recording at 10 seconds. If the model detects more than five items, return the first five in spoken order and an overflow warning. Do not silently backfill later items when an earlier item needs correction. Save complete items within those first five positions; ask for missing details on the others.

For images above 20 items, propose showing the first 20 with an explicit overflow warning so the user can scan the rest separately. This overflow policy needs confirmation; never silently discard extra detected items.

## API contract

| Endpoint | Multipart input | Result |
|---|---|---|
| POST /v1/parse/voice | audio + context JSON | Up to 5 candidates and warnings |
| POST /v1/parse/image | image + context JSON | Up to 20 review candidates and warnings |

Context contains `captureId`, the phone's current date, IANA timezone, `currency: INR`, `language: en`, and available `{id, name}` categories. Send no prior expense history or budgets. Add the Firebase App Check token in `X-Firebase-AppCheck`.

```json
{
  "captureId": "phone-generated-uuid",
  "items": [
    {
      "itemKey": "1",
      "name": "Lunch",
      "amountPaise": 25000,
      "date": "2026-11-12",
      "categoryId": "existing-category-id",
      "needsInput": []
    }
  ],
  "warnings": [],
  "hasMoreItems": false
}
```

Allow nullable missing fields in candidates and list those fields in `needsInput`. Require complete fields for a saved expense. Warning codes include `VOICE_ITEM_LIMIT`, `IMAGE_ITEM_LIMIT`, and `UNCLEAR_DATE`.

The backend validates real calendar dates, safe integer paise, category membership, item counts, string lengths, and required fields. Treat voice/image content as untrusted data rather than instructions to the model.

Return stable error codes with a request ID: `400 INVALID_INPUT`, `401 INVALID_APP_CHECK`, `413 FILE_TOO_LARGE`, `415 UNSUPPORTED_MEDIA`, `429 RATE_LIMITED`, `502 PROCESSING_FAILED`, and `504 PROCESSING_TIMEOUT`. Include Retry-After where applicable. A retry can incur another AI charge; duplicate-safe local writes do not imply duplicate-free provider billing.

## Media and AI pipeline

### Voice

```text
10-second recording -> multipart upload -> media validation
 -> transcription -> expense extraction with structured output
 -> business validation -> JSON response -> local save/corrections
```

Interpret explicit dates using the supplied current date and timezone. Default to today only when the user omits a date. Ask for clarification on ambiguous explicit dates or missing amounts.

### Images

```text
camera/gallery -> local duplicate warning -> resize/re-encode copy
 -> multipart upload -> media validation -> vision extraction
 -> business validation -> JSON response -> review -> local save
```

Use a proposed initial image limit of 5 MB and a 1 MB audio limit; verify these against capture formats and handwriting tests. Validate actual file signatures, image dimensions, and audio duration on the server. Accept common Android JPEG/PNG and M4A/AAC formats; verify codec support during setup. Strip image metadata by re-encoding an app-owned upload copy, leaving the user's gallery original untouched.

The vision model distinguishes items from subtotal/total lines. Flag discounts, taxes, quantities, or unreadable prices that prevent reliable item totals. Do not invent prices or silently allocate charges. Reviewers correct unresolved values before saving. Final tax/discount allocation remains an open product decision.

Keep provider code behind `transcribeAudio()`, `parseVoice()`, and `parseImage()` interfaces. Use structured output for expense extraction and vision; transcription returns text. Model schema compliance does not establish financial accuracy.

## Temporary files, retries, and privacy

Express uses request-scoped memory or scratch files, deletes scratch files in `finally`, and sweeps stale scratch files after crashes. It does not persist recordings, images, transcripts, or extraction results. Exclude request bodies and media from logs and error reporting.

The phone retains app-owned capture files only while processing/retrying. Delete them after success, explicit cancellation, or a bounded expiry. Clean expired files on launch. Gallery originals belong to the user and stay untouched. A user must select or record media again after the temporary copy expires.

Explain that OpenAI has its own processing and retention terms. Verify applicable controls before release; server deletion does not promise zero provider retention.

Store SQLite inside Android app-private storage. Configure Android backup/data-extraction rules to exclude app data, matching the decision to omit cloud backup in version 1. Warn in product settings that uninstalling or losing the phone can lose records; CSV export is not in-app restore.

## API protection without accounts

Use Firebase App Check backed by Play Integrity. Express verifies tokens with Firebase Admin before accepting AI work. Integrate the native App Check module through Expo configuration and use development builds; Expo Go cannot exercise this native integration.

Test attestation through a Play internal-testing release before public launch. Keep development debug tokens restricted to development; disable bypasses in production. Firebase App Check attests the app and is not a user identity.

For the prototype, run one backend instance with bounded concurrency, IP rate limiting, a global request allowance, provider billing alerts, and an operator-controlled AI kill switch. A declared installation ID may support usage signals but cannot establish trustworthy device quotas. Provider budget alerts alone are not a hard spending cap.

Set upload size and timeout limits at both the proxy and Express. Proposed processing deadline: 60 seconds, with a slightly longer client timeout. Abort provider calls on expiry where supported. Do not automatically retry billable calls in a loop.

Before scaling the backend, move rate/admission counters to a shared atomic store. Any strict daily monetary cap needs shared usage accounting and conservative admission limits; it cannot depend only on estimates or a client counter.

## Deployment and configuration

Deploy one stateless Express container to a managed HTTPS container host. Start with request/response processing and limited concurrency. Defer workers and durable jobs unless measured latency requires them.

Server configuration:

```text
OPENAI_API_KEY
OPENAI_TRANSCRIPTION_MODEL
OPENAI_EXTRACTION_MODEL
OPENAI_VISION_MODEL
FIREBASE_PROJECT_ID
# Firebase Admin credentials through the host's secret/identity facility
MAX_AUDIO_BYTES
MAX_IMAGE_BYTES
PROCESSING_TIMEOUT_MS
MAX_CONCURRENT_PROCESSING
AI_ENABLED
```

The mobile build receives the public API URL and Firebase client configuration. Neither is a secret; enforce access on the server. Provider keys and Firebase Admin credentials stay in server secret storage.

Use EAS development builds for phone testing and production AAB builds for Play distribution. Sign and retain Android release credentials. Run database migrations before rendering normal screens. Keep a compatibility window when changing API contracts because users may run older app builds.

## Verification and first milestone

Build one complete voice flow first: record “Lunch 250,” process it through Express, save it in SQLite, and edit/delete the resulting expense. In the same milestone, test App Check in a Play test track and measure a production build's device-specific download size.

Then add image review, charts, budgets, monthly rollover, and CSV export.

Focus tests on money conversion, spoken/image dates, timezone month boundaries, allocation validation, skipped-month rollover, duplicate local writes, and transactional image saves. Mock AI in routine tests; use a separate real-media evaluation set for extraction accuracy, latency, and cost.

The evaluation set should include five-item voice clips within ten seconds, 20-item printed receipts, handwriting, missing amounts, receipt totals, duplicate scans, and item overflow. On the Android phone, test denied permissions, offline operation, interrupted uploads, double taps, and app restart.

Export integer paise as exact decimal INR amounts with CSV quoting. Neutralize spreadsheet formula prefixes in user-controlled text fields before sharing the file through Android's share sheet.

## Future cloud sync

Keep repository interfaces, UUIDs, and timestamps now. These help later but do not implement sync. A later version needs identity, cloud storage, change tracking, deletions/tombstones, conflict resolution, and consent to upload historical records. Choose MongoDB or another backend database when those requirements exist.

## Remaining decisions

- Exact SDK/package versions and AI models after compatibility and accuracy tests.
- Host, expected usage, and acceptable processing spend.
- Final starter categories and handling of ambiguous dates.
- Receipt taxes, discounts, quantities, and shared-charge allocation.
- Confirmation of image overflow and skipped-month budget policies proposed above.
- Accuracy acceptance thresholds for voice and handwriting.

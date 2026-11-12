# Version 1 Backlog

Based on [plan.md](plan.md) and [architecture.md](architecture.md). This file contains planned work only; no application code accompanies it.

Each task targets one focused development session. Numbers express a suggested order, not permission to ignore dependencies; each description names the interfaces or prerequisite tasks it needs, and an assignee can read the linked source documents without reading unrelated tasks. Use mocked providers and fixtures until a task calls for live verification. Tasks involving cloud accounts, builds, or Play testing require the relevant access; public release is outside this backlog.

## 1. Set up an empty project with a passing test
Goal: Establish a runnable TypeScript workspace before implementing features.
Description: Create an empty Expo mobile app, an Express app, and a shared contracts package under apps/mobile, apps/api, and packages/contracts, using compatible pinned dependencies. Add a minimal smoke test and root commands for tests and type checking; document setup and confirm both commands pass without implementing expense features.

## 2. Add continuous integration
Goal: Check documentation and future code changes on each pull request.
Description: Using the workspace from task 1, add a GitHub Actions workflow that installs from the lockfile and runs tests and type checking. Verify the workflow on a branch and ensure it requires no AI keys or production credentials.

## 3. Create an Android development build
Goal: Run the empty Expo app on a physical Android phone.
Description: Using the mobile workspace from task 1, configure EAS development and production build profiles and document the Android package identifier and signing setup. Install a development build on the test phone and record reproducible build/run instructions without committing credentials.

## 4. Measure baseline Android download size
Goal: Establish whether the chosen stack can meet the 30 MB target.
Description: Using the build configuration from task 3, produce a release AAB and measure a device-specific download estimate through bundletool or Play internal testing. Record download and installed sizes separately, with the build configuration and device details; treat 20 MB as a stretch goal and identify the largest contributors if the download exceeds 30 MB.

## 5. Resolve remaining product edge cases
Goal: Give implementers explicit rules for unresolved expense and budget cases.
Description: Review plan.md and architecture.md and record proposed rules for ambiguous dates, receipt taxes/discounts/quantities, images above 20 items, first-month budgets, and skipped months. Mark which rules require the owner's decision rather than silently treating architectural proposals as confirmed, and identify the affected tasks as blocked until those decisions exist.

## 6. Define shared processing contracts
Goal: Give the phone and Express a common validated API format.
Description: Using packages/contracts from task 1, define portable Zod schemas for capture context, candidate expenses, warnings, and structured errors for POST /v1/parse/voice and POST /v1/parse/image. Include capture IDs, item keys, nullable missing fields, needsInput, integer paise, dates, existing category IDs, English/INR context, and the 5/20-item limits; verify example payloads without importing server-only modules.

## 7. Implement money and calendar helpers
Goal: Handle rupees and monthly date ranges without rounding or timezone errors.
Description: In the mobile domain layer from task 1, implement decimal INR input conversion to integer paise, currency formatting, date-only validation, and half-open month ranges. Test paise precision, invalid amounts, leap dates, and month/year boundaries using an injectable clock/timezone rather than the host date.

## 8. Create SQLite schema and migrations
Goal: Store expenses, categories, budgets, and scan fingerprints locally.
Description: Using expo-sqlite in the mobile app from task 1, add numbered transactional migrations for Category, Expense, MonthlyBudget, CategoryBudget, and ImageScan as described in architecture.md. Enable foreign keys, UUIDs, timestamps, indexes, positive expense checks, and unique capture/item and month/category constraints; verify repeat startup preserves existing data.

## 9. Add category storage and defaults
Goal: Provide persistent categories for entry forms and AI classification.
Description: Using the Category table from task 8, seed stable default categories including Other and add repository operations to list categories and create a custom category. Use normalized names to prevent duplicates and verify re-seeding does not duplicate records; keep the proposed starter list in one documented configuration pending final review.

## 10. Add expense repository operations
Goal: Support validated local expense creation, editing, deletion, and monthly queries.
Description: Using the Expense schema from task 8 and helpers from task 7, implement parameterized repository operations with category references and date-range filtering. Verify that edits can move an expense between months, deletes remove it from sums, and identical capture/item keys cannot insert duplicate records.

## 11. Build the three-tab navigation shell
Goal: Expose Expenses, Charts, and Budget with month/year selection.
Description: Using the Expo app from task 1 and date helpers from task 7, add the three tabs and a reusable month/year selector that defaults to the phone's current month. Give each tab its own selected period and placeholder content; verify changing one tab's filter does not silently change another's.

## 12. Build manual expense entry
Goal: Let users record a complete expense without internet.
Description: Using categories from task 9 and expense storage from task 10, add a form for name, INR amount, date, and category with today as the initial date. Validate required fields and positive amounts, save through the domain/repository boundary, and verify offline creation without any API call.

## 13. Add custom category creation to expense forms
Goal: Let users create and reuse categories while entering expenses.
Description: Using the category repository from task 9 and manual form from task 12, add an inline create-category action and select the resulting category for the current item. Confirm the new category survives restart and appears in future forms while duplicate names show a useful validation message.

## 14. Display the monthly expense list
Goal: Show saved expenses for the selected month and year.
Description: Using navigation from task 11 and the repository from task 10, render name, amount, date, and category for the selected period with an empty state. Refresh after local changes and on tab focus, and verify an expense dated in another month stays out of the current list.

## 15. Add expense editing and deletion
Goal: Let users correct or remove saved expenses.
Description: Using the list from task 14 and repository from task 10, expose Edit and Delete actions and reuse validated form fields for updates. Verify category, amount, and date changes appear on returning to the list and prevent repeated taps from applying duplicate mutations.

## 16. Create the Express processing service shell
Goal: Provide a configured API with predictable errors and no expense database.
Description: Using apps/api from task 1 and contracts from task 6, add a health endpoint, validated environment configuration, request IDs, and centralized structured error handling. Supply an example environment file with placeholders and tests for invalid configuration and error responses, keeping AI calls mocked.

## 17. Add server upload validation and cleanup
Goal: Reject unsupported media before it reaches paid processing.
Description: Using the API shell from task 16, accept multipart media plus context JSON and validate actual signatures, audio duration, image dimensions, and configured byte limits. Start with proposed 1 MB audio, 5 MB image, and 10-second audio limits, verify supported Android codecs, and test scratch cleanup on success, failure, and stale-file startup recovery.

## 18. Integrate native App Check in the Android app
Goal: Obtain app attestation tokens without user accounts.
Description: Using the development build from task 3, configure Firebase App Check with Play Integrity through Expo-compatible native configuration. Confirm the app obtains a token and document development debug-token setup with no secrets committed; token transport and server verification belong to separate tasks.

## 19. Verify App Check tokens in Express
Goal: Reject unauthenticated requests before AI processing.
Description: Using the API shell from task 16, add Firebase Admin token verification for X-Firebase-AppCheck to processing routes. Test missing, invalid, and valid tokens with mocks and confirm production configuration rejects any development bypass; do not add user login.

## 20. Add API rate limits and processing admission controls
Goal: Bound paid AI work in the initial single-instance backend.
Description: Using the API middleware from tasks 16 and 19, add IP rate limits, global request allowances, concurrency limits, AI_ENABLED, and processing deadlines with cancellation where supported. Verify rejected requests never invoke the provider and return stable 429/timeout errors; document that in-memory limits reset on restart and cannot guarantee a hard monetary cap.

## 21. Evaluate transcription and extraction models
Goal: Choose candidate OpenAI models using recorded expense examples.
Description: Using the provider boundary described in architecture.md, assemble a small consented or synthetic English evaluation set including five-item ten-second clips, dates, and missing amounts. Record available model IDs, current documented prices, accuracy, latency, and estimated cost per command; keep private audio and credentials out of Git and do not guess unresolved fields.

## 22. Implement the transcription adapter
Goal: Convert valid English audio to text through a swappable provider.
Description: Using API configuration from task 16 and model findings from task 21, implement transcribeAudio with the OpenAI SDK behind a provider interface. Test timeout, provider failure, empty speech, and successful text with mocks, and avoid logging audio or transcripts.

## 23. Implement voice expense extraction
Goal: Convert a transcript into up to five validated expense candidates.
Description: Using contracts from task 6, approved date rules from task 5, and model findings from task 21, implement structured text extraction with current date, timezone, and existing categories. Preserve the first five spoken item positions, flag overflow, mark missing details instead of guessing, and verify explicit dates and INR-to-paise conversion with mocked provider responses.

## 24. Expose the voice processing endpoint
Goal: Return validated voice results without persisting expenses on the server.
Description: Using upload validation, attestation, admission controls, transcription, and extraction from tasks 17, 19, 20, 22, and 23, connect POST /v1/parse/voice. Test a valid response, extra-item warnings, missing details, malformed provider output, and provider errors, and verify the route returns the supplied capture ID and cleans temporary media.

## 25. Build the mobile processing API client
Goal: Upload media with context and handle API responses safely.
Description: Using contracts from task 6 and App Check tokens from task 18, add multipart requests containing capture ID, current date, timezone, English/INR settings, and current categories. Validate responses, implement cancellation and bounded timeouts, and surface structured errors without automatic billable retry loops; test with a mock server.

## 26. Add ten-second voice recording
Goal: Capture a bounded English expense command on Android.
Description: Using the development build from task 3, implement microphone permission handling, recording status, cancellation, and a hard ten-second stop. Generate a capture ID before recording, handle background/interruption events, and return an app-owned temporary audio file without calling the backend yet.

## 27. Save complete voice results and correct missing fields
Goal: Turn voice candidates into editable local expenses without duplicate inserts.
Description: Using the API client from task 25, recorder from task 26, and expense repository from task 10, save complete items among the first five positions in a transaction and expose Edit/Delete on saved results. Keep incomplete candidates out of totals, ask for missing fields in a correction form, show the five-item overflow message, and verify double taps or late responses cannot duplicate saves.

## 28. Deploy the private processing API
Goal: Make the voice endpoint reachable over HTTPS from the test phone.
Description: Using the service and middleware from tasks 16–24, choose a container host and deploy one instance with secrets, bounded concurrency, matching upload limits, and health checks. Document configuration and deployment/rollback steps and confirm a private phone request succeeds without committing credentials or opening unrestricted paid endpoints.

## 29. Verify the first voice milestone on a phone
Goal: Prove the record-to-local-list flow with real processing.
Description: Using the deployed API from task 28 and voice flow from task 27, record Lunch 250, verify its category/date/amount, then edit and delete it. Test a missing amount, six spoken items, offline processing failure, and a delayed response; record observed results and extraction issues without expanding scope into image features.

## 30. Verify App Check through Play internal testing
Goal: Confirm production attestation works with the intended distribution channel.
Description: Using App Check from tasks 18–19 and Android builds from task 3, publish a test build to an authorized Play internal-testing track and exercise a protected endpoint from the test phone. Record successful attestation and rejection of missing/invalid tokens, confirm production has no debug bypass, and document signing/package configuration needed for repeat builds.

## 31. Add camera and gallery acquisition
Goal: Let users choose either image input source on Android.
Description: Using the development build from task 3, add take-photo and select-image actions with permission-denied, cancel, and unavailable-camera handling. Return an app-owned copy and capture ID while leaving gallery originals unchanged; do not upload or save expenses in this task.

## 32. Add local duplicate-image warnings
Goal: Warn before processing an exact image file already imported.
Description: Using ImageScan from task 8 and image acquisition from task 31, compute SHA-256 from original bytes and look for prior confirmed imports. Show Continue/Cancel on a match, allow continued duplicate imports, and verify that cancelling stops processing; document that recompressed or newly photographed images may not match.

## 33. Prepare image uploads
Goal: Reduce upload size while keeping receipt and handwriting text readable.
Description: Using acquired images from task 31, create an app-owned resized/re-encoded JPEG or PNG with metadata removed and a configured maximum byte size. Verify rotation, legibility, and the proposed 5 MB upload limit against sample images without modifying the gallery original; keep the original-byte fingerprint separate from upload preparation.

## 34. Evaluate vision extraction on receipts and handwriting
Goal: Choose a vision model and identify item-level accuracy limits.
Description: Using the image contract from task 6 and edge-case decisions from task 5, prepare a small consented or synthetic set of printed receipts and handwritten lists including 20 items, totals, unclear amounts, and overflow. Compare available OpenAI models on item counts, prices, categories, dates, latency, and cost, then record a candidate model and explicit acceptance thresholds without committing private images.

## 35. Implement image expense extraction
Goal: Return item-level candidates rather than a receipt total expense.
Description: Using contracts from task 6 and vision findings from task 34, implement parseImage with structured output and existing category IDs. Apply approved receipt/date/overflow rules, omit subtotal/total rows, flag unresolved charges and fields, and test a ten-item result, handwriting, and malformed provider output with mocks.

## 36. Expose the image processing endpoint
Goal: Process one uploaded image into a reviewable response.
Description: Using API protections and media validation from tasks 17, 19, and 20 plus extraction from task 35, connect POST /v1/parse/image. Verify up to 20 candidates, explicit overflow warnings under the approved policy, request IDs, timeout handling, and scratch cleanup without retaining server-side expense records.

## 37. Build the image review editor
Goal: Let users correct scanned candidates before any expense is saved.
Description: Using contracts from task 6 and category controls from tasks 9 and 13, build a review screen with editable name, amount, date, and category per item. Support removing entries and adding missing ones up to 20, keep invalid/incomplete entries from confirmation, and verify that editing or cancelling review writes no expenses.

## 38. Confirm image imports in one transaction
Goal: Save reviewed image entries together and prevent accidental repeat insertion.
Description: Using image review from task 37, expense storage from task 10, and ImageScan from task 8, implement a confirmation command that writes reviewed expenses and the scan fingerprint in one transaction. Verify rollback on failure and duplicate-safe capture/item keys while still allowing a user-approved repeated image under a new capture ID.

## 39. Connect the image capture-to-save flow
Goal: Deliver camera/gallery scanning with mandatory review.
Description: Using image capture, duplicate checks, upload preparation, API client, endpoint, review, and save commands from tasks 31–38 and 25, connect the full user flow. Verify a ten-item image creates ten review entries and no expenses until confirmation, and test cancel, duplicate continuation, overflow, offline failure, and user-initiated retry.

## 40. Add capture file lifecycle cleanup
Goal: Keep temporary media only as long as processing and retry need it.
Description: Using capture features from tasks 26 and 31, centralize app-owned temporary-file tracking with deletion on success/cancel and a documented bounded expiry. Sweep expired files at launch and verify interrupted uploads leave recoverable files only until expiry; never delete gallery originals or expose private media in logs.

## 41. Query monthly category spending
Goal: Provide one source of totals for charts and budget comparisons.
Description: Using the SQLite expense repository from task 10 and month ranges from task 7, add queries for monthly total and category totals including custom categories. Verify paise sums, month boundaries, and refresh after edits/deletes without storing duplicate aggregate totals.

## 42. Build the Charts tab
Goal: Display the selected month's spending distribution.
Description: Using the tab shell from task 11 and spending queries from task 41, render a small SVG pie chart with category names, amounts, percentages, and total spending. Handle zero-expense months and many categories with readable labels, and verify changing the month or editing an expense refreshes the display.

## 43. Add budget storage and allocation validation
Goal: Persist monthly plans without altering actual expenses.
Description: Using MonthlyBudget and CategoryBudget from task 8, add repository/domain operations for total and category allocations in integer paise. Save edits transactionally, reject allocations above the proposed total, calculate unallocated funds, and test changes to the total alongside allocations.

## 44. Implement idempotent monthly budget initialization
Goal: Copy plans into new months without carrying balances or losing history.
Description: Using budget storage from task 43 and approved first/skipped-month rules from task 5, implement ensureMonthBudget for startup, resume, and opening a month. Copy only total and category allocations, preserve existing months, and test repeat calls, year boundaries, skipped months, and the first month without an earlier plan.

## 45. Build the budget editor
Goal: Let users change the total and category allocations for one month.
Description: Using navigation from task 11 and budget services from tasks 43–44, add monthly total editing and per-category allocation controls including custom categories. Show unallocated money, preserve form edits when allocations exceed the total, and ask users to increase that total before saving; verify changing the plan leaves expense rows untouched.

## 46. Show budget spending and overrun analytics
Goal: Compare planned and actual spending inside the app.
Description: Using budget storage from task 43 and expense aggregates from task 41, show planned, spent, remaining, and exact overrun amounts for each category and the month overall. Label categories without allocations as No budget set, include their expenses in monthly spending, and verify overspending never blocks expense entry or triggers phone notifications.

## 47. Export the selected month's expenses as CSV
Goal: Let users share expense records outside the app.
Description: Using monthly expense queries from task 10 and month selection from task 11, generate CSV with date, name, category, and exact decimal INR amounts and open Android's share sheet. Quote fields, neutralize spreadsheet formula prefixes in user text, and verify only the selected month appears; do not add import or restore.

## 48. Add local-data and external-processing disclosures
Goal: Explain storage and processing behavior before users rely on the app.
Description: Using the data boundaries in architecture.md and working capture flows from tasks 27 and 39, add concise in-app copy for online processing, provider retention, and temporary media. Configure Android backup/data-extraction exclusions and show that uninstalling or losing the phone may lose data; confirm CSV export is not described as a restorable backup.

## 49. Verify offline use and restart recovery
Goal: Check that local expense tracking does not depend on the API.
Description: Using expenses, charts, budgets, export, and media lifecycle work from tasks 12–15 and 40–48, run a focused phone checklist in airplane mode and across app restarts. Confirm records persist, local features work, unavailable processing offers a clear retry path, and expired files do not break recovery; record and separately ticket any failures.

## 50. Run extraction acceptance checks
Goal: Measure whether AI behavior meets the agreed voice/image thresholds.
Description: Using evaluation sets from tasks 21 and 34 and live endpoints from tasks 24 and 36, run repeatable checks for five-item voice, twenty-item images, missing fields, date interpretation, and receipt double counting. Record accuracy, latency, and per-request cost against the chosen thresholds, and create targeted follow-up tickets for failures instead of silently loosening acceptance.

## 51. Add production monitoring and release controls
Goal: Make processing failures and abnormal usage visible without logging private content.
Description: Using the deployed API from task 28 and admission limits from task 20, add metadata-only metrics for latency, error codes, provider usage, and rejected requests plus billing alerts and a tested AI kill switch. Verify logs exclude media, transcripts, expense names, and request bodies, and document that alerts do not enforce a hard monetary cap.

## 52. Measure final Android size and trim dependencies
Goal: Check the feature-complete build against the 30 MB download target.
Description: Using all shipped native modules and the measurement method from task 4, build a production AAB and record device-specific download size, installed size, and dependency contributors. Remove unused dependencies/assets and remeasure where needed; document any remaining overage rather than promising an unverified size.

## 53. Perform the version 1 release walkthrough
Goal: Confirm the agreed user journeys before public distribution.
Description: Using the completed app, deployed API, extraction findings from task 50, and size report from task 52, run the acceptance checks in plan.md on the Android test phone. Include cross-month expense edits, new/custom categories, budget rollover, duplicate scans, permissions, repeated save taps, and CSV sharing; document pass/fail evidence and unresolved blockers without publishing a public release.

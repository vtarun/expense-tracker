# Android Expense Tracker — Version 1

## Purpose
Record expenses through voice, images, or manual entry.
Analyze monthly spending and manage category budgets.

## Basics
- Android app, English only.
- Currency: Indian rupees (₹), supporting paise.
- Store expenses and budgets on the phone.
- Use internet-based voice and image processing.
- Saved records, manual entry, charts, and budgets work offline.
- Explain that voice/images are sent to a processing service.

## Tabs

### 1. Expenses
- Month/year filters, defaulting to the current month.
- Each expense contains name, amount, date, and category.
- Add expenses through voice, camera, gallery, or a manual form.
- Allow editing and deleting expenses.
- Keep previous months accessible.
- Export the selected month's expenses as CSV.

### 2. Charts
- Spending pie chart grouped by category.
- Month/year filters, defaulting to the current month.
- Show total spending, category amounts, and percentages.

### 3. Budget
- Monthly total budget and individual category budgets.
- Allow editing both.
- Show planned, spent, and remaining amounts.
- Show unallocated money when category budgets total less
  than the monthly budget.
- If category budgets exceed the monthly total, ask the user
  to increase the total before saving.
- Save expenses even when their category has no budget;
  show “No budget set.”
- Allow spending beyond budgets.
- Show exact category and monthly overruns inside the app.
- No phone notifications.
- Changing budgets never changes actual expenses.

## Voice Entry
- English voice commands.
- Maximum recording length: 10 seconds.
- Maximum 5 expenses per command.
- Automatically extract name, amount, category, and date.
- Example: “Spent 250 on lunch” creates:
  Lunch | ₹250 | Dining out | Today.
- Use spoken dates when provided; otherwise use today.
- Ask for missing required details, such as the amount.
- Save complete expenses immediately with Edit/Delete options.
- If more than 5 expenses are detected, save the first 5
  and flag the rest:
  “You can add only 5 items per voice command.
  Please record the remaining items again.”

## Image Entry
- Support camera photos and gallery images.
- Support printed receipts and handwritten expense lists.
- Create one expense per item: 10 items create 10 entries.
- Automatically suggest a category for every item.
- Use the image date when available; otherwise use today.
- Maximum 20 items per image/review batch.
- Require review before saving.
- During review, allow editing, deleting, and adding
  missing items within the limit.
- Warn about possible duplicate scans.
- Let users continue or cancel after a duplicate warning.
- Avoid counting receipt subtotals and totals as extra items.

## Manual Entry
- Form containing name, amount, date, and category.
- Works offline.

## Categories
- Start with default categories.
- Suggested defaults: Dining out, Groceries, Transport,
  Shopping, Bills, Health, Entertainment, Education,
  Travel, and Other.
- Let users correct automatic category suggestions.
- Let users create a category when adding/editing an item.
- Save new categories for future expenses and budgets.

## Monthly Rollover
- Start a fresh expense view each month.
- Copy the previous month's total budget and category budgets.
- Do not carry forward unspent money or overspending.
- Retain historical expenses and monthly budgets.
- Allow editing the copied budget.
- Separate monthly views do not require separate database tables.

## CSV Export
- Export the selected month and year.
- Include date, name, category, and amount in INR.
- CSV import and restore are not included.

## Outside Version 1
- Cloud storage and account sync.
- Backup and restore.
- Multiple languages.
- Multiple currencies and currency conversion.
- Phone notifications.
- Budget balance carryover.

## Decisions Still Needed
- Android technology stack and minimum Android version.
- Voice/image processing provider and processing costs.
- Media retention and privacy details.
- Final default category list.
- Handling unclear dates and unreadable amounts.
- Handling receipt taxes, discounts, and shared charges.
- Handling images with more than 20 items.
- Duplicate detection rules.
- First-month budget setup and skipped-month rollover.

## Acceptance Checks
- Voice creates up to 5 expenses and flags extras.
- Recordings stop at 10 seconds.
- Image entries require review before saving.
- A 10-item image produces 10 separate expense entries.
- Users can correct and delete entries.
- Expense changes update charts and budget calculations.
- Month/year filters show matching records.
- Budget overruns and unallocated amounts are accurate.
- Monthly rollover preserves historical records.
- CSV contains the selected month's expenses.
- Local features remain usable without internet.
# Obsidian Timesheet Report (Dataview + Timekeep)

A [Dataview](https://blacksmithgu.github.io/obsidian-dataview/) JS template that aggregates time entries from the [Timekeep](https://github.com/jacobtread/obsidian-timekeep) plugin into a per-client timesheet report, rendered as a table directly inside your Obsidian vault.

Set a client name and a reporting period in the note's frontmatter, and the template scans your daily notes for matching Timekeep entries, totals the hours, and flags any unclosed timers.

## Prerequisites

- [Obsidian](https://obsidian.md/) (desktop or mobile)
- [Dataview](https://blacksmithgu.github.io/obsidian-dataview/) community plugin
- [Timekeep](https://github.com/jacobtread/obsidian-timekeep) community plugin
- Daily notes stored in a folder called `Daily Notes` (the default Obsidian daily-notes location)

## Installation

### 1. Install the community plugins

1. Open **Settings > Community plugins**
2. Click **Browse** and search for **Dataview** — install and enable it
3. Repeat for **Timekeep** — install and enable it

### 2. Enable Dataview JavaScript queries

This template uses a `dataviewjs` code block, which requires JavaScript queries to be explicitly enabled — they are **off by default**.

1. Open **Settings > Community plugins > Dataview** (click the gear icon)
2. Toggle **Enable JavaScript Queries** to on

Without this, the template will render as plain text instead of a report.

### 3. Add the report template

1. Copy `timesheet-report-template.md` into your vault
2. Rename it to suit your use case (e.g. `Acme Corp Timesheet.md`)
3. Create as many copies as you need — one per client, per report type, or both

### 4. Install the CSS snippet (optional)

The included `modrich-labs-timesheet-report.css` improves the table layout by widening the date column and right-aligning the hours column.

1. In your vault's root folder, navigate to `.obsidian/snippets/` — create the `snippets` folder if it doesn't exist
2. Copy `modrich-labs-timesheet-report.css` into that folder
3. Open **Settings > Appearance**
4. Scroll down to **CSS snippets** and click the reload button (circular arrow)
5. Toggle **modrich-labs-timesheet-report** to on

## Configuration

Each report note is configured through its frontmatter properties:

```yaml
---
target-client: Acme Corp
report-type: FYTD
---
```

### `target-client`

The client name to filter by. The template matches this value (case-insensitive) against your Timekeep entry names. Any entry whose name contains the client string will be included in the report.

For example, if `target-client` is set to `Acme`, entries named "Acme Corp - Development", "acme support", or "Meeting with Acme" will all match.

### `report-type`

Controls the date range for the report. Supported values:

| Value  | Period                          | Notes                                              |
| ------ | ------------------------------- | -------------------------------------------------- |
| `FYTD` | Financial year to date          | Australian financial year: 1 July to 30 June       |
| `YTD`  | Calendar year to date           | 1 January to today                                 |
| `MTD`  | Month to date                   | 1st of the current month to today                  |
| `WTD`  | Week to date                    | Monday of the current week to today (ISO week)     |

All periods run from their start date through to today — they never show future dates.

> **Note:** The `FYTD` period uses the Australian financial year (1 July – 30 June). If your financial year starts on a different date, edit the `FYTD` block in the template to match.

## How it works

1. The template reads `target-client` and `report-type` from the note's frontmatter
2. It queries all notes in the `Daily Notes` folder whose file date falls within the selected period
3. For each daily note, it uses the Timekeep plugin API to extract timekeep code blocks
4. It walks the entry tree (including nested sub-entries) looking for names that contain the target client string
5. Matched entries are totalled and displayed in a table with columns for date, activity description, and hours

Entries with unclosed timers (no end time) are flagged with a warning and excluded from the hour total.

## Important notes

- **Daily notes folder:** The template scans a folder called `Daily Notes` (line 82 of the template). If your daily notes live elsewhere, update the folder name in the `dv.pages('"Daily Notes"')` call.
- **Date detection:** Daily notes must have a date that Dataview can detect — typically an ISO date in the filename (e.g. `2026-06-18`). Notes without a detectable date are silently skipped.
- **Entry naming:** For entries to appear in the report, your Timekeep entry names must contain the `target-client` value. Name your timekeep entries consistently with the client's name.

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on reporting issues, suggesting features, and submitting pull requests.

Please note that this project is released with a [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to abide by its terms.

## License

[MIT](LICENSE) — Copyright (c) 2026 Modrich Labs

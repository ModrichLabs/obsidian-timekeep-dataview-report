---
target-client: ACME
report-type: FYTD
---
```dataviewjs

/*
MIT License

Copyright (c) 2026 Modrich Labs

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
*/

// 1. Fetch the search term and report type from the document's properties
const targetClient = (dv.current()['target-client'] || "")
const searchWord = targetClient.toLowerCase();
const reportType = (dv.current()['report-type'] || "").trim().toUpperCase();

const validReportTypes = ["FYTD", "YTD", "MTD", "WTD"];
const reportTypeLabels = {
    "FYTD": "Financial year to date",
    "YTD": "Calendar year to date",
    "MTD": "Month to date",
    "WTD": "Week to date"
};

if (!searchWord) {
    dv.paragraph("⚠️ Please set the **target-client** property in this note's frontmatter.");
} else if (!validReportTypes.includes(reportType)) {
    dv.paragraph(`⚠️ Please set the **report-type** property in this note's frontmatter to one of: ${validReportTypes.join(", ")}.`);
} else {
    // 2. Determine the reporting window start/end and a display label based on report-type.
    // All windows run from their natural start through to today ("to date"),
    // not through to the end of the period, so the report never shows an
    // empty tail for days that haven't happened yet.
    const today = moment();
    let periodStart, periodEndToDate, periodLabel;

    if (reportType === "FYTD") {
        // Australian financial year: 1 July – 30 June.
        // moment months are 0-indexed, so June = 5, July = 6.
        const fyStartYear = today.month() >= 6 ? today.year() : today.year() - 1;
        const fyEndYear = fyStartYear + 1;
        periodStart = moment(`${fyStartYear}-07-01`, "YYYY-MM-DD");
        periodEndToDate = today.clone();
        periodLabel = `FY ${fyStartYear}-${fyEndYear}, to date`;
    } else if (reportType === "YTD") {
        periodStart = today.clone().startOf('year');
        periodEndToDate = today.clone();
        periodLabel = `Calendar Year ${today.year()}, to date`;
    } else if (reportType === "MTD") {
        periodStart = today.clone().startOf('month');
        periodEndToDate = today.clone();
        periodLabel = `${today.format('MMMM YYYY')}, to date`;
    } else if (reportType === "WTD") {
        // Force Monday as the start of the week regardless of locale setting —
        // isoWeekday(1) is Monday, independent of locale-dependent startOf('week').
        periodStart = today.clone().isoWeekday(1).startOf('day');
        periodEndToDate = today.clone();
        periodLabel = `Week of ${periodStart.format('D MMMM YYYY')}, to date`;
    }

    // 3. Find and filter notes falling strictly within the selected period-to-date window
    const pages = dv.pages('"Daily Notes"').where(p => {
        if (!p.file.day) return false;
        // file.day is already a Luxon DateTime — use its own ISO date rather than
        // re-stringifying and re-parsing, which is fragile across Dataview versions.
        const noteDate = moment(p.file.day.toISODate(), "YYYY-MM-DD");
        return noteDate.isSameOrAfter(periodStart, 'day') && noteDate.isSameOrBefore(periodEndToDate, 'day');
    });

    const timekeepPlugin = this.app.plugins.plugins.timekeep.api;

    // --- Helpers for recursive (parent/sub-entry) Timekeep structures ---

    function nameMatches(entry) {
        return entry.name && entry.name.toLowerCase().includes(searchWord);
    }

    // Collect leaf entries (for duration/open-state calculation) belonging to
    // any entry — at any depth — whose own name matches or whose nearest
    // matching ancestor matches. Parent entries with subEntries carry null
    // start/end times by design; only leaves are meaningful for duration.
    function collectMatchingLeaves(entry, ancestorMatched) {
        const thisMatches = ancestorMatched || nameMatches(entry);

        if (!entry.subEntries || entry.subEntries.length === 0) {
            return thisMatches ? [entry] : [];
        }
        return entry.subEntries.flatMap(child => collectMatchingLeaves(child, thisMatches));
    }

    // Find top-level entries that contain a match anywhere within them —
    // these supply the single description line shown per match in the table.
    function topLevelHasMatch(entry) {
        if (nameMatches(entry)) return true;
        if (!entry.subEntries) return false;
        return entry.subEntries.some(topLevelHasMatch);
    }

    let totalDurationMs = 0;
    let openEntryCount = 0;
    let reportData = [];

    for (let page of pages) {
        const file = app.vault.getAbstractFileByPath(page.file.path);
        const content = await app.vault.read(file);
        const timekeeps = timekeepPlugin.parser.extractTimekeepCodeblocks(content);

        let dayDescriptions = [];
        let dayMatchingLeaves = [];

        for (const tk of timekeeps) {
            const structuralBlockName = tk.name || tk.blockName || tk.title || "";
            const blockMatches = structuralBlockName.toLowerCase().includes(searchWord);

            for (const entry of (tk.entries || [])) {
                if (blockMatches || topLevelHasMatch(entry)) {
                    // Use the top-level entry's own name as the description —
                    // not a sub-entry-by-sub-entry breakdown.
                    dayDescriptions.push(entry.name || "Unnamed");
                    dayMatchingLeaves.push(...collectMatchingLeaves(entry, blockMatches));
                }
            }
        }

        if (dayMatchingLeaves.length > 0) {
            // Open = a leaf with no endTime. Leaves are the only level where
            // start/end are meaningful, so nested entries are handled correctly.
            const closedLeaves = dayMatchingLeaves.filter(e => e.endTime);
            const openLeaves = dayMatchingLeaves.filter(e => !e.endTime);

            const durationMs = closedLeaves.length > 0
                ? timekeepPlugin.queries.getTotalDuration(closedLeaves, moment())
                : 0;

            totalDurationMs += durationMs;
            openEntryCount += openLeaves.length;

            const hours = (durationMs / 3600000).toFixed(2);

            reportData.push({
                date: page.file.day.toFormat('yyyy-MM-dd'),
                description: dayDescriptions.join("<br>"),
                hours: openLeaves.length > 0 ? `${hours} ⚠️` : hours
            });
        }
    }

    // 3b. Sort the collected rows chronologically by date before rendering,
    // since dv.pages() does not guarantee date order.
    reportData.sort((a, b) => a.date.localeCompare(b.date));

    // 4. Render headers indicating the period-to-date range that was used
    const totalHours = (totalDurationMs / 3600000).toFixed(2);
    dv.header(2, `Timesheet Report: ${targetClient}`);
    dv.paragraph(`**Report Type:** ${reportTypeLabels[reportType]}`);
    dv.paragraph(`**Reporting Period:** ${periodStart.format('D MMMM YYYY')} to ${periodEndToDate.format('D MMMM YYYY')} (${periodLabel})`);
    dv.paragraph(`**Total Hours:** ${totalHours} hours`);

    if (openEntryCount > 0) {
        dv.paragraph(`⚠️ **${openEntryCount} unclosed timer entr${openEntryCount === 1 ? 'y' : 'ies'} found** — these are excluded from the total above. Review and close them in the source daily notes, then re-run this report.`);
    }

    dv.paragraph("---");

    // 5. Display the standard Dataview table
    if (reportData.length > 0) {
        dv.table(
            ["Date", "Activity Description", "Hours"],
            reportData.map(r => [r.date, r.description, r.hours])
        );
    } else {
        dv.paragraph(`*No matching entries found for "${searchWord}" inside ${periodLabel}.*`);
    }
}
```
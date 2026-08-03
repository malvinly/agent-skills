# Work plan: {short title of the overall change}

Source: {"synthesized from conversation on YYYY-MM-DD" or path of the original plan}
Branch: {branch name} · Commit policy: {e.g. single commit at end}

## Goal

{One or two sentences: what this plan delivers when every task is done.}

## Tasks

<!-- One section per task. Each description must be SELF-CONTAINED: the implementer subagent sees only what the orchestrator pastes from here, never this file or the conversation. -->

### Task {N}: {title}

- **Files:** {exact files to create or modify}
- **Change:** {full description of the change — complete enough that someone with no other context could implement it}
- **Verify:** `{command}` → {expected result}
- **Depends on:** {task numbers, or "nothing"}
- **Parallel-safe:** {yes/no — yes only if it depends on no unfinished task AND its write scope is disjoint from every task it could run alongside}
- [ ] Implemented
- [ ] Reviewed

## Discoveries

{Appended during the run: codebase quirks, conventions, gotchas worth injecting into later dispatch prompts.}

---

# Filled-in example (delete when using this template)

# Work plan: CSV export for ReportService

Source: synthesized from conversation on 2026-08-03
Branch: main · Commit policy: single commit at end

## Goal

Add RFC-4180-compliant CSV export to the existing reporting pipeline, wired through DI, with unit tests.

## Tasks

### Task 1: Create CsvExporter

- **Files:** `src/MyApp/Export/CsvExporter.cs` (new)
- **Change:** Implement `IReportExporter` with an `Export(ReportData data)` method producing RFC-4180-compliant CSV: CRLF line endings; fields containing commas, quotes, or line breaks wrapped in double quotes; embedded double quotes doubled. Header row from `ReportData.Columns`, one row per entry in `ReportData.Rows`.
- **Verify:** `dotnet build MyApp.sln` → build succeeds
- **Depends on:** nothing
- **Parallel-safe:** no (Tasks 2 and 3 depend on it)
- [ ] Implemented
- [ ] Reviewed

### Task 2: Wire CsvExporter into ReportService

- **Files:** `src/MyApp/Services/ReportService.cs`, `src/MyApp/Program.cs`
- **Change:** Add an `IReportExporter` constructor parameter to `ReportService` and an `ExportToCsv(ReportData data)` method delegating to it. Register `CsvExporter` as the `IReportExporter` implementation in the DI setup in `Program.cs`, following the registration pattern already used there.
- **Verify:** `dotnet build MyApp.sln` → build succeeds
- **Depends on:** Task 1
- **Parallel-safe:** no
- [ ] Implemented
- [ ] Reviewed

### Task 3: CsvExporter unit tests

- **Files:** `tests/MyApp.Tests/Export/CsvExporterTests.cs` (new)
- **Change:** Unit tests covering: fields with embedded commas are quoted; embedded double quotes are doubled and quoted; empty `ReportData` produces only the header row. Follow the test naming and fixture conventions already present in `MyApp.Tests`.
- **Verify:** `dotnet test tests/MyApp.Tests` → all tests pass
- **Depends on:** Task 1
- **Parallel-safe:** no (depends on Task 1; run sequentially by default)
- [ ] Implemented
- [ ] Reviewed

## Discoveries

- (example) DI registrations in `Program.cs` use extension methods in `ServiceCollectionExtensions.cs` — register new services there, not inline.

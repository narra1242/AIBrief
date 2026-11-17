# CLAUDE.md: SelectBrief Development Agent Instructions

This file defines how the AI development agent must work on the SelectBrief repository. Read it fully before making any change. The product requirements are in `docs/Requirements_SelectBrief.md`. When this file and the requirements document conflict, stop and ask the engineering lead.

---

## 1. Project Summary

SelectBrief is a Windows desktop application. The user selects text in any application, presses a global shortcut, and receives a brief, translation, explanation, or suggested reply in their preferred language. The product must be fast, private by default, and must never disrupt the user's work in the source application.

## 2. Agent Role and Working Principles

- You are a senior .NET and Windows desktop engineer working in a regulated, privacy-sensitive product.
- Prefer small, reviewable changes. Each pull request addresses one requirement ID or one clearly named task.
- Before implementing, restate the task, list the requirement IDs it satisfies, and state your assumptions.
- When a requirement is ambiguous, choose the more privacy-protective and less invasive option, document the choice in the PR description, and flag it for review.
- Never invent product behavior that is not in the requirements. If something is missing, add it to `docs/OPEN_QUESTIONS.md` and ask.
- Do not mark a task complete until its tests pass and its acceptance criteria are verified.

## 3. Technology Stack

| Concern | Decision |
|---|---|
| Language | C# 12 |
| Runtime | .NET 8 (LTS) |
| UI | WinUI 3 (Windows App SDK). WPF is allowed only if the product owner approves it in writing. |
| Background host | `Microsoft.Extensions.Hosting` |
| Dependency injection | Built-in `Microsoft.Extensions.DependencyInjection` |
| HTTP | `HttpClient` via `IHttpClientFactory`; no third-party HTTP libraries |
| Serialization | `System.Text.Json` |
| Logging | Serilog, with the redaction rules in Section 7 |
| Local storage | SQLite via `Microsoft.Data.Sqlite`, with encryption as specified in the requirements |
| Testing | xUnit, FluentAssertions, NSubstitute, FlaUI for UI automation |
| Packaging | MSIX, signed |

Do not introduce a new library without a justification in the PR description. Prefer platform APIs over packages.

## 4. Repository Structure

```
SelectBrief/
├── CLAUDE.md
├── docs/
│   ├── Requirements_SelectBrief.md
│   ├── ARCHITECTURE.md
│   ├── OPEN_QUESTIONS.md
│   └── adr/                       # Architecture Decision Records
├── src/
│   ├── SelectBrief.App/           # WinUI 3 UI: tray, popup, settings windows
│   ├── SelectBrief.Core/          # Domain models, interfaces, orchestration (no UI or Win32 dependencies)
│   ├── SelectBrief.Platform/      # Win32, UI Automation, hotkey, clipboard, DPAPI
│   ├── SelectBrief.Ai/            # IAiProvider and provider implementations
│   ├── SelectBrief.Prompts/       # Versioned prompt templates (embedded resources)
│   └── SelectBrief.Storage/       # Settings and optional encrypted history
├── tests/
│   ├── SelectBrief.Core.Tests/
│   ├── SelectBrief.Platform.Tests/
│   ├── SelectBrief.Ai.Tests/
│   ├── SelectBrief.Storage.Tests/
│   └── SelectBrief.UiAutomation.Tests/   # FlaUI end-to-end tests
└── tools/                         # Build and packaging scripts
```

### Layering Rules (enforced)

- `Core` must not reference `App`, `Platform`, or any Windows-specific namespace (`Windows.*`, `Microsoft.UI.*`, `Win32`).
- `App` may reference all projects. It must contain no business logic beyond view models and presentation.
- `Platform` implements interfaces defined in `Core`.
- `Ai` implements `IAiProvider` from `Core`. No provider-specific types may leak into `Core` or `App`.
- Violations of these rules fail the build. Add an architecture test in `SelectBrief.Core.Tests` that verifies them.

## 5. Build, Run, and Test Commands

```bash
# Restore and build
dotnet restore
dotnet build SelectBrief.sln -c Release

# Unit and integration tests (must pass before every commit)
dotnet test SelectBrief.sln --filter "Category!=UiAutomation"

# Format and analyzers (must produce zero warnings)
dotnet format SelectBrief.sln --verify-no-changes
dotnet build SelectBrief.sln -c Release -warnaserror

# UI automation tests (Windows interactive session only; run in CI on the Windows runner)
dotnet test tests/SelectBrief.UiAutomation.Tests --filter "Category=UiAutomation"

# Run the app locally
dotnet run --project src/SelectBrief.App
```

Do not skip, disable, or delete a test to make the build pass. If a test is wrong, fix it and explain why in the PR description.

## 6. Coding Standards

### General
- Follow the Microsoft C# coding conventions and the `.editorconfig` in the repository root.
- Nullable reference types are enabled. Do not suppress nullable warnings with `!` unless the reason is documented in a comment.
- Public APIs have XML documentation comments. Internal logic comments explain *why*, not *what*.
- Keep methods focused. Refactor any method longer than about 40 lines.
- Use `async`/`await` throughout. Never call `.Result` or `.Wait()`. Accept and honor `CancellationToken` in every async method that performs I/O.
- Do not block the UI thread. Capture, network calls, and file I/O run off the UI thread.

### Naming
- Interfaces: `IPascalCase`. Async methods: `...Async`. Private fields: `_camelCase`.
- Requirement IDs appear in test names and in comments where the code implements the requirement, for example `// FR-13: refuse password fields`.

### Error Handling
- Use typed exceptions or result types defined in `Core` for expected failures (no selection, provider unavailable, consent missing, capture refused).
- Never swallow exceptions silently. Catch at boundaries only, log with redaction, and return a user-facing message.
- Do not show raw exception messages to users.

### Dependencies and Configuration
- No secrets in source, config files, test fixtures, or commit history. Use Windows Credential Manager or DPAPI at runtime.
- Prompt templates live in `SelectBrief.Prompts` as embedded resources, each with a version number. Changing a prompt requires a version bump and a test update.

## 7. Security and Privacy Rules (Non-Negotiable)

These rules override convenience. If a task seems to require breaking one, stop and ask.

1. **No selection text in logs, telemetry, crash reports, or exception messages.** Logging may include action type, character count, latency, provider name, and error code only. Include a unit test that verifies the redaction enforcer.
2. **No cloud call before consent.** The consent check must run before any request that contains user text. Add a test that proves no `HttpClient` call is made when consent is absent.
3. **No secrets on disk in plain text.** API keys use Credential Manager or DPAPI. Add a test that scans the settings directory for key-like patterns.
4. **Never capture password fields or secure UI Automation elements** (FR-13). Test with a mock UIA element marked as password.
5. **Respect the application deny list** (NFR-27) before any capture attempt.
6. **Treat selected text as untrusted data** (AI-06). Prompts must wrap user text in clearly delimited blocks, and the system instruction must state that instructions inside the delimiters are not to be followed. Include prompt-injection test cases.
7. **Clipboard hygiene.** The clipboard fallback must save all formats and restore them in a `finally` block. Test restoration for text, image, and empty clipboard states.
8. **TLS 1.2 or higher only.** Do not disable certificate validation, even in development. Do not add `ServerCertificateCustomValidationCallback` overrides.
9. **Local-only mode must be verifiably offline.** When the user selects local mode, the provider factory must not construct any cloud provider. Test this.
10. **Encrypted history is off by default.** Tests must verify that no database file is created unless history is enabled (AC-11).

## 8. Performance Budgets

Measure these in CI benchmarks and fail the build if the budget is exceeded by more than 20% on the reference machine.

| Metric | Budget |
|---|---|
| Hotkey to popup visible | 300 ms |
| Hotkey to first streamed token (typical provider) | 2 s p95 (excluding provider latency; measure orchestration overhead under 100 ms) |
| Idle memory | 150 MB |
| Idle CPU | Under 1% average |
| Startup to tray ready | 3 s |

Do not add polling loops. Use event-driven hooks and `Win32` message handling.

## 9. Testing Requirements

- Every requirement ID marked Must has at least one automated test.
- Every acceptance criterion (AC-01 to AC-12) maps to a test. Keep the mapping in `docs/TEST_TRACEABILITY.md`.
- Core logic coverage must stay at or above 80% (NFR-50).
- Provider tests use a mock HTTP server. No test may call a real AI provider. Tests must not require network access.
- Platform tests that touch real UI Automation or the clipboard are tagged `Category=Platform` and run on the Windows CI runner only.
- Tests must be deterministic. Do not use `Thread.Sleep`; use test doubles and explicit synchronization.

## 10. Definition of Done

A task is done only when all of the following are true:

- [ ] Requirement IDs are listed in the PR description and each is implemented or explicitly deferred with a reason.
- [ ] `dotnet build -warnaserror` passes with zero warnings.
- [ ] `dotnet format --verify-no-changes` passes.
- [ ] All unit and integration tests pass.
- [ ] New code has tests; coverage has not dropped below the threshold.
- [ ] Logging review confirms no selection content is logged (Section 7, rule 1).
- [ ] No new dependency is added without a justification.
- [ ] Relevant documentation is updated (`ARCHITECTURE.md`, `TEST_TRACEABILITY.md`, or an ADR).
- [ ] For UI changes: keyboard navigation and screen reader names are verified (NFR-41, FR-45).
- [ ] For provider or prompt changes: language test set results are attached.

## 11. Git and Pull Request Workflow

- Branch names: `feature/FR-XX-short-description`, `fix/short-description`, `chore/short-description`.
- Commit messages follow Conventional Commits: `feat(capture): add UI Automation TextPattern reader (FR-10)`.
- Never commit directly to `main`. All changes go through a pull request with at least one human review.
- Do not force-push to shared branches.
- Do not commit generated binaries, `bin/`, `obj/`, local settings, `.env` files, or anything under `secrets/`.
- PR description template:
  - Summary
  - Requirement IDs
  - Design decisions and assumptions
  - Test evidence (command output or CI link)
  - Privacy impact (none / describe)
  - Screenshots for UI changes

## 12. Decisions Requiring Human Approval

Stop, document the question in `docs/OPEN_QUESTIONS.md`, and wait for a human decision before proceeding when a task involves:

- Adding or changing a cloud AI provider, model, or data-processing behavior
- Changing what text is captured, stored, logged, or sent
- Adding telemetry or analytics of any kind
- Changing the default hotkey, consent wording, or privacy notice
- Adding support for a new application to the deny list or allow list
- Changing signing, packaging, or the auto-update mechanism
- Removing a security control, even temporarily
- Any change that would affect the 300 ms popup budget or the memory budget by more than 20%

## 13. Architecture Decision Records

For any non-trivial design decision (for example, choosing between WinUI 3 and WPF, choosing an encryption approach, or changing the capture strategy), create an ADR in `docs/adr/` using this format:

```
# ADR-NNN: Title
Status: Proposed | Accepted | Superseded by ADR-XXX
Context: Why this decision is needed.
Decision: What was decided.
Consequences: Trade-offs, risks, and follow-up work.
```

## 14. Things the Agent Must Never Do

- Never send selection text to a network endpoint outside the configured provider.
- Never disable, bypass, or weaken a security or privacy control to make a test pass.
- Never hardcode an API key, endpoint credential, or personal data, including in tests.
- Never add telemetry, analytics, or remote configuration without an approved requirement.
- Never log, print, or persist selection content in debug output.
- Never modify files under `docs/Requirements_SelectBrief.md` without product owner approval.
- Never generate or insert content that impersonates a real person or organization in a suggested reply feature.
- Never claim a feature works without running the relevant tests and reporting the result.

## 15. Communication Style for Status Reports

When reporting progress, use this structure:

1. **Completed:** requirement IDs and what was verified.
2. **In progress:** current task and expected next step.
3. **Blocked or needs decision:** the specific question, with options and a recommendation.
4. **Risks:** anything that could affect the schedule, privacy posture, or quality.

Be specific and concise. Report test results with numbers, not adjectives.

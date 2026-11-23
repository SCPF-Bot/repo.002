# Overview.md - Android Agentic AI Coder: Meta AI Bridge App

## Table of Contents
1. Executive Overview
2. Product Vision and Mission
3. Problem Statement
4. Solution Overview
5. Scope: In Scope and Out of Scope
6. User Personas and Use Cases
7. Product Principles
8. System Architecture Overview
9. Core Concepts and Terminology
10. End-to-End User Journey
11. Detailed Feature Specification
12. Conversation Protocol Specification
13. Prompt Templates
14. Response Parser Specification
15. Directory Tree Specification
16. File Handling and SAF Specification
17. Diff Engine Specification
18. Confirmation and Apply Specification
19. Backup and Undo Specification
20. Conversation State Machine
21. Data Models and Database Schema
22. UI Specification: Screens, Components, Navigation
23. Design System and Accessibility
24. Security, Privacy, and Permissions
25. Performance, Limits, and Quotas
26. Error Handling and Edge Cases
27. Logging, Analytics, and Telemetry (Privacy-First)
28. Testing Strategy
29. Build, CI, and Release Process
30. Milestones and Roadmap
31. Risks, Assumptions, and Open Questions
32. Alternatives Considered
33. Appendices

---

## 1. Executive Overview

This document is the single source of truth for the Android Agentic AI Coder app, called Meta AI Bridge in this spec.

The app lets an Android developer use Meta AI chat as an agentic coding partner for a real local project, without an API, without zip upload, and without stale cached GitHub links.

The core loop is:

Code files on device > App reads files fresh > App builds prompt with directory tree + files + task > User copies prompt to Meta AI > Meta AI replies with description + new files, or asks for more files > App sends more files if asked > App parses reply into diffs > User accepts, edits, or declines per file > App writes accepted files directly to the correct folder.

The app is the hands. Meta AI is the brain. The user is the bridge for copy-paste and the final approver for writes.

The app is local-first, no backend, no server, no API key for Meta AI. All file operations happen on device. All transfer to Meta AI happens through the user manually copying prompt and answer, which is stable and does not get blocked.

## 2. Product Vision and Mission

### Vision
Any Android developer can open any local project on their phone, describe a change in plain language, and get a safe, reviewable, applicable code change from Meta AI, without leaving the phone and without breaking the project.

### Mission
Build a robust, logical, and trustworthy bridge between local Android project files and Meta AI chat, where:
- Every file sent is fresh from disk
- Every change is shown as a diff before it is written
- Every write requires explicit user confirmation
- Every conversation can request more files and handle multi-turn context
- Every failure is visible, recoverable, and explainable

### Success Criteria
- A user can complete a UI change task end to end on a real sample project in under 10 minutes, including copy-paste to Meta AI.
- Zero silent writes. Every write is logged and has a backup.
- Zero stale file reads. Every prompt uses a fresh read at build time.
- Parser success rate over 95 percent on well-formed Meta AI answers, with clear recovery for malformed answers.
- App works offline except for the Meta AI chat step, which needs internet in the Meta AI app or browser.

## 3. Problem Statement

### 3.1 Cached Links Fail
When a developer sends a GitHub blob link or raw link to an AI, the AI reader often returns a cached copy. If the file was just edited, the AI sees the old version. This was proven in testing with `test.txt` where the cached copy kept returning `code 555` after the file was changed. The app solves this by never using links. It reads files directly from local storage at prompt build time.

### 3.2 Zip Upload Fails
Uploading a zip to Meta AI chat in the mobile app often fails or returns a generic refusal. The app solves this by never sending a zip. It sends files as plain text, one or more files per prompt, with a directory tree for context.

### 3.3 No API for Meta AI Chat
Meta AI chat has no official programmatic API like Claude Code does. Trying to automate the web chat with Selenium or Playwright is fragile, breaks on layout changes, needs login handling, and can get blocked. The app solves this by using a human-in-the-loop bridge: the app prepares prompts and parses answers, the user does the copy-paste. This is stable and does not violate automation constraints.

### 3.4 One-Shot Prompts Lose Context
A single prompt with one file is not enough for a real change like "change the UI". The change may touch JSON, Kotlin, XML, and text files. The app solves this by always sending the directory tree, and by supporting NEED_FILE rounds where Meta AI can ask for related files and the app sends them in the next prompt.

### 3.5 Unsafe Auto-Apply
An agent that writes files without confirmation can corrupt a project. The app solves this by requiring per-file Accept, Edit, or Decline, and by creating backups before every write.

## 4. Solution Overview

The app is a local Android app that manages the full agentic loop, except the Meta AI chat step itself.

Components:
- Project Picker: pick a project folder with SAF
- Tree Builder: build a sorted directory tree string
- File Manager: read and write files with SAF
- Prompt Builder: build prompts with task, tree, and files
- Conversation Manager: keep turns, detect NEED_FILE, handle follow-ups
- Response Parser: parse DESCRIPTION, NEED_FILE, FILE blocks
- Diff Engine: compute line diffs per file
- Review UI: show description, diffs, and per-file decisions
- Apply Engine: backup and write accepted files
- History Store: save sessions, proposals, and apply logs in Room

## 5. Scope

### In Scope for V1
- Pick project folder with SAF, persist permission
- Build and show directory tree
- Select starting files, type task, build first prompt
- Copy prompt to clipboard, paste Meta AI answer back
- Parse DESCRIPTION, NEED_FILE, FILE blocks
- Auto-build next prompt when Meta AI asks for more files, up to 8 rounds
- Show diff per file, per-file Accept, Edit, Decline
- Apply accepted files, create backups, undo last apply
- Session history, proposal history, apply log
- Secret scan warning before sending files
- Large file warning and chunk suggestion
- Offline local operation except Meta AI chat

### Out of Scope for V1
- Direct API calls to Meta AI chat
- Automated login or scraping of meta.ai
- Auto-send to Meta AI without user copy-paste
- Git integration, commit, push, branch management
- Building or running the Android project inside the app
- Image or binary file editing
- Multi-user collaboration or cloud sync
- Real-time collaboration with another developer

### Future Scope (V2+)
- Chunked file sending for very large files
- Image context: send UI screenshot as description, not as code
- Export session as markdown
- Optional LLM API integration for users who have a key and want full automation, reusing the same diff and confirm flow
- Git diff integration and commit message suggestion

## 6. User Personas and Use Cases

### Persona 1: Solo Android Developer on Phone
Wants to make a quick UI change to a small project while away from a laptop. Opens the app, picks the project, types "change the UI to dark theme", gets a proposal, reviews diffs, applies.

### Persona 2: Student Learning Android
Wants to understand a codebase and make safe changes. Uses the app to ask Meta AI to explain and then propose changes, reviews diffs to learn what changed.

### Persona 3: Indie Dev Maintaining a Project
Has a medium project with many files. Needs Meta AI to understand cross-file impact. Uses tree + NEED_FILE rounds so Meta AI can request related files and propose coordinated changes.

### Core Use Cases
- UC1: Change UI colors and layout in one file
- UC2: Change a JSON key and update all files that use it
- UC3: Add a new screen and update navigation
- UC4: Fix a bug described in an error log pasted as task context
- UC5: Refactor a function and update callers
- UC6: Review a proposal, edit one file manually, decline another, apply the rest

## 7. Product Principles

1. Fresh files only. Never cache file content across turns except for display. Always re-read at prompt build time.
2. Text, not links or zips. All transfer is plain text.
3. Structured conversation. Meta AI must reply in a parseable format.
4. Diff before write. No write without diff and confirmation.
5. Per-file decisions. Accept, Edit, Decline is per file.
6. Human bridge. User copies prompt to Meta AI and answer back. App handles before and after.
7. No silent writes. App never writes except in Apply step.
8. Recoverable. Every proposal is stored. Every write has a backup. Undo is available.
9. Visible failures. If parsing fails, show raw answer and offer repair. Never silently drop a file.
10. Least privilege. Use SAF, not broad storage permission. No backend, no server.

## 8. System Architecture Overview

```
User (Android phone)
  |
  | picks folder, types task, copies prompt, pastes answer, confirms
  v
Android App (Kotlin + Compose)
  |
  +-- UI Layer: Home, Project, Prompt, Review, Editor, History
  +-- Domain Layer: ConversationManager, PromptBuilder, ResponseParser, DiffEngine, ApplyEngine
  +-- Data Layer: Room DB, FileManager (SAF), BackupStore, Settings (DataStore)
  |
  v (manual copy-paste)
Meta AI Chat (web or app, external)
```

### Module Dependencies
- UI depends on Domain, never directly on Data except through ViewModels.
- Domain depends on Data interfaces, not on Android framework where possible, to allow unit testing.
- FileManager is the only module that touches SAF DocumentFile and ContentResolver.
- PromptBuilder is pure Kotlin, no Android dependencies, fully testable.
- ResponseParser is pure Kotlin, fully testable.
- DiffEngine is pure Kotlin, fully testable.

### Tech Stack
- Language: Kotlin 1.9+
- UI: Jetpack Compose, Material 3, Navigation Compose
- DI: Hilt or manual DI for V1
- Storage: Room 2.6+, DataStore Preferences
- File access: Storage Access Framework, DocumentFile
- Clipboard: ClipboardManager
- Min SDK: 26, Target SDK: 34+
- Build: Gradle with version catalog
- No backend, no Firebase, no analytics SDK in V1

## 9. Core Concepts and Terminology

- **Project:** A folder picked with SAF that contains the code to edit.
- **Tree:** Sorted text representation of the project folder structure.
- **Task:** Natural language description of what to change, e.g. "change the UI".
- **Turn:** One round of prompt to Meta AI and answer back. A session can have many turns.
- **Proposal:** A parsed Meta AI answer that contains a DESCRIPTION and one or more FILE blocks.
- **Proposed File:** A new version of a file in a proposal, with original content, proposed content, and decision.
- **NEED_FILE:** A line in Meta AI answer asking for another file, e.g. `NEED_FILE: file.txt`.
- **DESCRIPTION:** Short paragraph in Meta AI answer explaining what changed and why.
- **FILE Block:** A block in Meta AI answer with a file path and full new content in a code fence.
- **Diff:** Line-by-line comparison between original and proposed content.
- **Apply:** Writing accepted files to disk after confirmation, with backups.
- **Session:** The full history of one task, including all turns, proposals, decisions, and apply logs.

## 10. End-to-End User Journey

### Journey for UI Change Task

1. User opens app, sees Home. Taps Add Project Folder.
2. System file picker opens. User picks project folder. App saves permission, builds tree.
3. App shows Project Detail with tree, file list, and search.
4. User checks `file.json` as starting file, types task: "i want you to change the UI".
5. User taps Start Task. App builds first prompt.
6. Prompt Screen shows prompt with Copy button. User taps Copy.
7. User switches to Meta AI app, pastes prompt, waits.
8. Meta AI replies: "okay, received, i will review it" + `NEED_FILE: file.txt` + `NEED_FILE: file.kt`.
9. User copies that answer, switches back to app, pastes answer.
10. App parses NEED_FILE, shows "Meta AI needs 2 more files", user taps Send Them.
11. App builds second prompt with `file.txt` and `file.kt` included. User copies it to Meta AI.
12. Meta AI replies with DESCRIPTION + 3 FILE blocks: `file.json`, `file.txt`, `file.kt`, noting it made improvements to other files because `file.json` changed.
13. User pastes answer. App parses proposal.
14. Review Screen shows description: "Updated UI colors..." + diffs for 3 files.
15. User reviews: Accepts `file.json`, Edits `file.txt` to fix a typo, Declines `file.kt` because it is not needed.
16. User taps Apply Accepted. App creates backups, writes `file.json` and edited `file.txt`, skips `file.kt`.
17. Apply Summary shows: 2 written, 1 declined, 0 failed, with Undo button.
18. Session is saved to history. User can reopen it later.

## 11. Detailed Feature Specification

### 11.1 Project Management
- Add project folder via SAF ACTION_OPEN_DOCUMENT_TREE.
- Persist permission with takePersistableUriPermission.
- List recent projects on Home, with name, last opened, file count.
- Remove project from list, with option to keep or delete backups.
- Re-pick folder if permission is lost, with clear error message.
- Support multiple projects, switch between them.

### 11.2 File Selection
- Show tree with expand/collapse, file icons by extension.
- Search by file name, filter by extension.
- Tap file to preview content in read-only viewer.
- Checkbox to select starting files, max 5 starting files to keep first prompt small.
- Show file size and warning if file is binary or over limit.

### 11.3 Task Input
- Text field for task, multi-line, with examples: "change the UI", "fix crash in MainActivity", "add dark theme".
- Optional context field: paste error log or extra notes.
- Button: Start Task. Disabled until at least one file and non-empty task.
- Task is stored in session and included in every prompt in the session.

### 11.4 Prompt Building
- Prompt Builder combines: Task, Tree, Files, Rules.
- Files are included in order: starting files first, then NEED_FILE files in order requested.
- Each file block has: `File: <relative path>` + `Content:` + full content.
- If total prompt exceeds 12000 characters, show warning and suggest reducing files or splitting task.
- Show character count and estimated tokens.

### 11.5 Copy-Paste Bridge
- Copy button uses ClipboardManager, shows Snackbar "Prompt copied".
- Paste Answer field has Paste from Clipboard button.
- Answer field supports large text, with scroll.
- Button: Parse Answer. Disabled if answer is empty.
- After parsing, if NEED_FILE, show next prompt automatically.

### 11.6 Conversation History
- Show turns in chronological list, with turn number, files sent, and summary.
- Tap a turn to see full prompt and full answer.
- Button: Add Follow-up. User types follow-up, app builds new prompt with same context + follow-up.
- Button: Retry with Strict Format. Sends repair prompt if last answer was malformed.

### 11.7 Proposal Review
- Show DESCRIPTION at top in a card.
- Show stats: files changed, lines added, lines removed.
- Per-file card with: path, badge Modified / New / No Change, diff view, decision buttons.
- Diff view with line numbers, green for added, red for removed, gray for context.
- Collapse long unchanged hunks with "Show 20 more unchanged lines" expander.
- Tap file to see full proposed content in viewer.

### 11.8 Edit Proposed File
- Editor screen with monospace font, line numbers, syntax highlight for .kt, .json, .xml, .txt, .md.
- Save returns to Review with Edited badge and updated diff.
- Discard Edit returns to original proposed content.
- Editor does not write to disk until Apply.

### 11.9 Apply and Backup
- Apply button shows confirmation dialog: "This will overwrite 2 files. Backups will be created. Continue?"
- On confirm, Apply Engine creates backups, writes files, logs apply.
- Apply Summary screen shows written, declined, edited, failed files.
- Undo Last Apply button restores from backups for that apply only.

### 11.10 History and Session Management
- History screen lists sessions by project and date, with task, status, files changed.
- Tap session to reopen: see turns, proposals, decisions, apply log.
- Delete session, with option to keep backups.
- Export session as markdown, for sharing or debugging.

## 12. Conversation Protocol Specification

This is the contract that makes the app work. It must be included in every prompt.

### 12.1 Rules Sent to Meta AI (Every Prompt)

```
Rules for you:
1. If you need more files, reply ONLY with lines like:
NEED_FILE: file.txt
NEED_FILE: file.kt
Do not explain, do not return code yet.
2. When you are ready to propose changes, reply with:
DESCRIPTION: <one short paragraph, what you changed and why>
FILE: <relative path>
```<full new content of that file>```
Repeat FILE block for each file you changed.
3. Only return full files, not diffs. Do not return partial files.
4. If you create a new file, use FILE: <new relative path> and say NEW in the description.
5. Do not return files you did not change.
6. Keep your DESCRIPTION short, under 100 words.
```

### 12.2 NEED_FILE Format

- One per line, exact format: `NEED_FILE: <relative path>`
- Relative path must match a path in the tree, or the app will report "File not found" in next prompt.
- App allows up to 10 NEED_FILE lines per turn.
- If Meta AI returns NEED_FILE plus other text, app extracts NEED_FILE lines and ignores other text for that turn, but shows full answer to user.

### 12.3 DESCRIPTION Format

- One line, starts with `DESCRIPTION:`
- Max 100 words, plain text, no markdown.
- Must explain what changed and why, and mention if other files were changed because of a primary file change.
- Example: "Updated UI colors to dark theme, changed layout spacing in file.json, and updated related text and logic in file.txt and file.kt because the JSON keys changed."

### 12.4 FILE Block Format

- Header: `FILE: <relative path>`
- Next line: opening code fence with optional language: ```json, ```kotlin, ```xml, or plain ```
- Content: full new file content, exactly as it should be written.
- Closing code fence: ```
- No extra text inside code fence except file content.
- For new files, relative path must include folder, e.g. `FILE: app/src/main/java/com/example/app/NewScreen.kt`
- For deleted files, use `FILE: <path>` with content `__DELETE__` and mention DELETE in DESCRIPTION. App will show Delete as a decision option, not as a write.

### 12.5 Repair Prompt (When Parsing Fails)

If parser finds no DESCRIPTION and no FILE blocks:

```
Your last answer did not follow the format. Please resend using ONLY this format:
DESCRIPTION: <short description>
FILE: <relative path>
```<full new content>```
Repeat FILE block for each changed file. Do not add extra explanation.
```

## 13. Prompt Templates

### 13.1 First Prompt Template

```
Task: {{task}}

Project tree:
{{tree}}

Starting file: {{startingFile}}
Content:
{{startingFileContent}}

{{#if extraContext}}
Extra context:
{{extraContext}}
{{/if}}

Rules for you:
{{rules}}
```

### 13.2 NEED_FILE Follow-up Prompt Template

```
Here are the files you asked for:

File: {{file1}}
Content:
{{content1}}

File: {{file2}}
Content:
{{content2}}

Continue with the original task: {{task}}
Use the same reply format: DESCRIPTION + FILE blocks, or NEED_FILE if you need more files.
```

### 13.3 User Follow-up Prompt Template

```
User follow-up: {{followUpText}}

Previous DESCRIPTION: {{previousDescription}}

Project tree:
{{tree}}

Files already sent in this session:
{{filesSentList}}

Please send a new proposal with the same format.
```

### 13.4 Repair Prompt Template

See Section 12.5.

## 14. Response Parser Specification

### 14.1 Inputs
- Raw answer text pasted by user.
- Current tree for path validation.
- List of files sent in this session.

### 14.2 Outputs
- ParsedResult: description, list of ProposedFile, list of requestedFiles, rawAnswer, parseErrors.
- If description is missing but FILE blocks exist, parser uses first line of answer as description and flags warning.
- If no FILE blocks and no NEED_FILE, parser returns empty and parseErrors = ["No FILE or NEED_FILE found"].

### 14.3 Parsing Steps
1. Extract DESCRIPTION with regex `DESCRIPTION:\s*(.*)` on first matching line.
2. Extract NEED_FILE with regex `NEED_FILE:\s*(\S+)` for all matches.
3. Extract FILE blocks with regex `FILE:\s*(\S+)\s*```(?:\w+)?\n(.*?)``` ` with DOTALL.
4. For each FILE block, trim content, check if content is empty or `__DELETE__`.
5. Validate path: if path is in tree, mark as Modified. If not in tree, mark as New and flag for user confirmation.
6. For each FILE, load original content from disk via FileManager if file exists, else original = "".
7. If proposed content == original content, mark as No Change.
8. Return ParsedResult with warnings.

### 14.4 Parser Edge Cases
- Code fence without language: accept.
- Multiple code fences in one FILE block: treat as one block, take content between first opening and last closing.
- FILE header with quotes or backticks: strip them.
- DESCRIPTION with line breaks: join to one paragraph.
- Answer with extra explanation before DESCRIPTION: ignore explanation, extract DESCRIPTION and FILE blocks.
- Answer with diff instead of full file: detect `+` and `-` diff markers, flag as "Diff not full file" and ask for repair.

### 14.5 Parser Tests (Must Pass)
- Valid answer with DESCRIPTION + 3 FILE blocks -> 3 files parsed.
- Answer with NEED_FILE only -> requestedFiles = 2, no proposal.
- Answer with DESCRIPTION + 1 FILE, plus extra text -> 1 file parsed.
- Answer with malformed FILE (no closing fence) -> parse error, show raw answer.
- Answer with duplicate FILE paths -> keep last, warn about duplicate.

## 15. Directory Tree Specification

### 15.1 Building
- Start at project root Uri.
- Use DocumentFile.listFiles() recursively.
- Sort: folders first, then files, both alphabetically, case-insensitive.
- Max depth: 6. Max files: 500. If exceeded, truncate and note "Tree truncated, 500 files shown".
- Exclude: `.git`, `.gradle`, `build`, `.idea`, `node_modules`, `*.apk`, `*.aab`, `*.so`, `*.png`, `*.jpg`, `*.jpeg`, `*.webp`, `*.mp4`, `.DS_Store`.
- Include: `.kt`, `.java`, `.xml`, `.json`, `.txt`, `.md`, `.gradle`, `.kts`, `.properties`, `.toml`.

### 15.2 Format
- Use 2 spaces per level.
- Folder lines end with `/`.
- File lines show name only, not full path, with indentation showing hierarchy.
- Example in Section 5.1 Tree Builder.

### 15.3 Size Info
- For files over 10KB, append ` (12KB)` to help Meta AI judge.
- For files over 50KB, append ` (LARGE, ask for chunk if needed)`.

## 16. File Handling and SAF Specification

### 16.1 Picking Project Folder
- Use Intent ACTION_OPEN_DOCUMENT_TREE.
- Request persistable permission: FLAG_GRANT_READ_URI_PERMISSION | FLAG_GRANT_WRITE_URI_PERMISSION | FLAG_GRANT_PERSISTABLE_URI_PERMISSION.
- Save Uri string in Room Project table.
- On app restart, check if permission still held via persistedUriPermissions. If not, show "Permission lost" state.

### 16.2 Reading Files
- Resolve relative path to DocumentFile by walking from root Uri.
- Open InputStream via ContentResolver, read as UTF-8, with BufferedReader.
- If file is binary (contains null byte in first 1024 bytes), skip and warn.
- If file is larger than 100KB, warn and offer to send only first 30KB with truncation marker.

### 16.3 Writing Files
- Only in Apply Engine, after confirmation.
- Resolve DocumentFile for target path. If not exists and isNew is true, create it with createFile with correct MIME type.
- Open OutputStream with mode "wt" (truncate).
- Write UTF-8 bytes.
- If write fails, log error, mark file as Failed, do not mark session as fully applied.
- Writing is done on IO dispatcher, with progress UI for multiple files.

### 16.4 MIME Types
- .kt -> text/plain, .java -> text/plain, .xml -> text/xml, .json -> application/json, .txt -> text/plain, .md -> text/markdown, .kts -> text/plain, .gradle -> text/plain.

## 17. Diff Engine Specification

### 17.1 Algorithm
- Use Myers diff for line-based comparison.
- Input: originalLines, proposedLines.
- Output: List of DiffHunk with type: Added, Removed, Unchanged, and line numbers for both versions.
- For performance, if file has more than 2000 lines, use a simple LCS fallback or limit diff to changed regions.

### 17.2 Display
- Show hunks with 3 lines of context before and after change.
- Collapse unchanged hunks longer than 10 lines with expander.
- Line numbers: show original line number on left, proposed line number on right.
- Colors: Added = green background, Removed = red background, Unchanged = neutral.
- Stats: +N added, -M removed, per file and total.

### 17.3 No Change Detection
- If original == proposed (after trimming trailing whitespace), mark as No Change, do not show Apply button for that file.

## 18. Confirmation and Apply Specification

### 18.1 Confirmation UI Flow
- Review Screen shows DESCRIPTION, stats, and per-file cards.
- Per-file decision defaults to PENDING.
- User must set a decision for each file before Apply is enabled, or tap Accept All.
- Apply button is disabled if all files are Declined.
- Confirmation dialog shows: "You are about to overwrite N files. Backups will be created. Continue?" with Cancel and Apply buttons.

### 18.2 Edit Flow
- Tapping Edit opens Editor Screen with proposed content pre-filled.
- Editor has Save and Cancel. Save updates userEditedContent and sets decision to EDITED.
- Editor shows diff between proposed and user-edited content in real time if possible, or on Save.

### 18.3 Apply Steps (Atomic per File)
1. For each Accepted or Edited file, in order:
   a. Read current file from disk again, compare hash to original hash at proposal time. If different, show warning "File changed on disk since proposal" and ask user to confirm overwrite or refresh.
   b. Save backup of current content to BackupStore.
   c. Write new content to temp file, then rename to target, or write directly via SAF OutputStream.
   d. Log apply: file path, timestamp, proposalId, original hash, new hash, decision.
2. After all files, show Apply Summary.
3. If any file fails, show which failed and why, and offer Retry Failed.

### 18.4 Delete Handling
- If proposed content is `__DELETE__`, show Delete as an option, not Overwrite.
- If user accepts Delete, app deletes the file via DocumentFile.delete(), after backup.
- If user declines Delete, file is kept.

## 19. Backup and Undo Specification

- Backup location: app-private files dir, `backups/<projectId>/<sessionId>/<relativePath>.bak`, plus a JSON manifest with original hash and timestamp.
- Keep last 20 backups per project, delete older ones on new backup.
- Undo Last Apply: restores all files written in last apply from backups, logs undo.
- Restore Single File: from History, user can restore a single file from a specific backup.
- Backups are not stored in the project folder, to avoid polluting the repo.

## 20. Conversation State Machine

States:
- IDLE: No active session.
- PROMPT_READY: First prompt built, waiting for user to copy to Meta AI.
- WAITING_FOR_ANSWER: Prompt copied, waiting for user to paste answer.
- PARSING: Answer pasted, parser running.
- NEED_MORE_FILES: Parser found NEED_FILE, waiting for user to build next prompt.
- REVIEW: Proposal parsed, waiting for user decisions.
- EDITING: User editing a proposed file.
- CONFIRMING: User tapped Apply, showing confirmation dialog.
- APPLYING: Writing files, showing progress.
- APPLIED: Apply done, showing summary.
- CANCELLED: Session cancelled, no writes.
- ERROR: Parse error or file error, showing recovery options.

Transitions:
- IDLE -> PROMPT_READY on Start Task
- PROMPT_READY -> WAITING_FOR_ANSWER on Copy
- WAITING_FOR_ANSWER -> PARSING on Parse Answer
- PARSING -> NEED_MORE_FILES if NEED_FILE found
- NEED_MORE_FILES -> PROMPT_READY on Build Next Prompt
- PARSING -> REVIEW if DESCRIPTION + FILE found
- PARSING -> ERROR if no parseable blocks
- REVIEW -> EDITING on Edit
- EDITING -> REVIEW on Save / Cancel Edit
- REVIEW -> CONFIRMING on Apply Accepted
- CONFIRMING -> APPLYING on Confirm
- APPLYING -> APPLIED on success
- Any state -> CANCELLED on Cancel Session

Max NEED_FILE rounds: 8. After 8, state goes to REVIEW with warning or to ERROR asking user to intervene.

## 21. Data Models and Database Schema

### 21.1 Room Entities

```kotlin
@Entity(tableName = "projects")
data class ProjectEntity(
    @PrimaryKey val id: String,
    val name: String,
    val folderUri: String,
    val lastOpened: Long
)

@Entity(tableName = "sessions")
data class SessionEntity(
    @PrimaryKey val id: String,
    val projectId: String,
    val taskText: String,
    val status: String,
    val createdAt: Long,
    val updatedAt: Long
)

@Entity(tableName = "turns")
data class TurnEntity(
    @PrimaryKey val id: String,
    val sessionId: String,
    val promptText: String,
    val answerText: String,
    val filesSentJson: String,
    val requestedFilesJson: String,
    val timestamp: Long
)

@Entity(tableName = "proposals")
data class ProposalEntity(
    @PrimaryKey val id: String,
    val sessionId: String,
    val description: String,
    val createdAt: Long
)

@Entity(tableName = "proposed_files")
data class ProposedFileEntity(
    @PrimaryKey val id: String,
    val proposalId: String,
    val relativePath: String,
    val isNew: Boolean,
    val originalContent: String,
    val proposedContent: String,
    val userEditedContent: String?,
    val decision: String
)

@Entity(tableName = "apply_logs")
data class ApplyLogEntity(
    @PrimaryKey val id: String,
    val sessionId: String,
    val proposalId: String,
    val relativePath: String,
    val originalHash: String,
    val newHash: String,
    val decision: String,
    val timestamp: Long,
    val success: Boolean,
    val errorMessage: String?
)
```

### 21.2 Settings (DataStore)
- Max prompt characters: default 12000
- Max NEED_FILE rounds: default 8
- Show line numbers in diff: default true
- Auto-backup: default true
- Secret scan enabled: default true

## 22. UI Specification: Screens, Components, Navigation

### Navigation Graph
Home -> Project Detail -> Prompt -> Review -> Editor -> Apply Summary -> History
Home -> History -> Session Detail

### Screen Details

#### Home
- Top bar: App name, Settings icon.
- Recent projects list: card per project with name, file count, last task.
- FAB: New Project.
- Empty state: illustration + "Pick a project folder to start" + button.

#### Project Detail
- Top bar: Project name, back, search icon.
- Tree view: expandable, with folder icons, file icons.
- File preview pane: tap file to preview, with Copy Path button.
- Bottom bar: Selected files count, Task input, Start Task button.

#### Prompt Screen
- Task summary at top.
- Prompt text area, scrollable, with Copy button.
- Character count.
- Paste Answer area, with Paste button.
- Parse Answer button.
- Turn history list at bottom, collapsed.

#### Review Screen
- Description card.
- Stats row: Files, +Lines, -Lines.
- Per-file cards with diff.
- Per-file buttons: Accept (green), Edit (blue), Decline (gray).
- Bottom bar: Accept All, Decline All, Apply Accepted (disabled until at least one Accept/Edit).

#### Editor Screen
- Top bar: File path, Save, Cancel.
- Text area with line numbers, monospace.
- Syntax highlight: keywords for Kotlin, tags for XML, keys for JSON.
- Bottom: character count, Save button.

#### Apply Summary
- Success icon, summary text.
- List of written files with check icons.
- List of declined files.
- Buttons: Undo Last Apply, Start New Task, Back to Home.

#### History
- List of sessions grouped by project.
- Each item: task, date, status badge, files changed.
- Tap to open Session Detail with turns and proposals.

### Components
- FileCard: path, badge, diff stats, decision buttons.
- DiffView: line numbers, colors, collapsible hunks.
- PromptBox: text area with Copy button.
- AnswerBox: text area with Paste button.
- ConfirmDialog: generic dialog for Apply, Delete, Discard.
- EmptyState: icon + text + action button.
- ErrorState: icon + message + Retry button.

## 23. Design System and Accessibility

- Material 3, dynamic color, light and dark theme.
- Typography: Title Large for screen titles, Body Medium for file content, Label Small for badges.
- Colors: Primary for actions, Error for destructive, Success green for Accept, Warning amber for large file or secret warning.
- Touch targets: min 48dp.
- Accessibility: content descriptions for all buttons, TalkBack support for diff (read added and removed lines), focus order logical, text scaling support.
- Language: English for V1, strings in strings.xml for future localization.

## 24. Security, Privacy, and Permissions

### Permissions
- No MANAGE_EXTERNAL_STORAGE. Use SAF only.
- No internet permission needed for app itself, except to open Meta AI in browser if user taps that button.
- No contacts, location, or camera permissions.

### Privacy
- No file content leaves the device except via user manual copy-paste to Meta AI. The app itself makes no network calls to Meta AI.
- No analytics SDK in V1. If added later, it must be opt-in and must not log file contents, only counts like "proposal applied" or "parse failed".
- Backups stay in app-private storage, not in project folder, not in cloud.

### Secret Scan
- Before building a prompt, scan selected files for patterns: `api_key`, `API_KEY`, `password`, `secret`, `token`, `private_key`, `firebase`.
- If found, show warning card: "This file may contain a secret. Do you want to exclude it or redact it?"
- User can: Exclude file, Redact lines, or Send Anyway.
- Redact replaces matched lines with `[REDACTED]` in the prompt only, not in the original file.

## 25. Performance, Limits, and Quotas

- Prompt limit: 12000 characters default, configurable up to 30000. If exceeded, show warning and suggest splitting task.
- File size limit: 100KB per file for sending. If larger, offer to send first 30KB with truncation marker.
- Tree limit: 500 files, depth 6.
- NEED_FILE rounds: max 8 per session.
- Diff limit: if file has more than 2000 lines, diff shows only changed hunks, not full file.
- Room DB: prune sessions older than 90 days, keep apply logs for 90 days.
- UI: Prompt and Review screens must load in under 300ms for typical project with 50 files.

## 26. Error Handling and Edge Cases

### Parser Errors
- No FILE blocks: show raw answer, offer Retry with Strict Format.
- Malformed FILE block: show which file failed, keep other files, offer to ask Meta AI to resend that file.
- Duplicate FILE paths: keep last, warn.

### File Errors
- File not found in tree for NEED_FILE: send "File not found" in next prompt.
- File changed on disk since proposal: warn before apply, offer Refresh Proposal.
- SAF permission lost: show "Permission lost, please pick folder again".
- Write failed: mark file as Failed, log error, offer Retry Failed.

### User Errors
- Empty task: disable Start Task, show hint.
- Empty answer: disable Parse, show hint.
- Pasting truncated answer: detect unbalanced fences, ask to paste full answer.
- Declining all files: Apply disabled, show hint "Accept at least one file to apply".

### System Errors
- App killed mid-session: session is saved after each turn, can be resumed.
- Low storage: before apply, check free space, warn if less than 10MB.
- Large prompt: warn, suggest splitting.

## 27. Logging, Analytics, and Telemetry (Privacy-First)

- Local logs only in V1, stored in Room, not sent anywhere.
- Log events: session started, prompt built, answer parsed, proposal created, file accepted, file applied, apply failed, parse failed.
- Never log file contents in analytics. Only log counts and paths, not contents.
- User can export logs as text for debugging, and clear logs.

## 28. Testing Strategy

### Unit Tests (Must Have)
- Tree Builder: nested folders, sorting, exclusions, truncation.
- Prompt Builder: task + tree + files + rules, character count.
- Response Parser: valid, NEED_FILE only, malformed, duplicate, diff instead of full file.
- Diff Engine: added, removed, unchanged, no change detection.
- Apply Engine: backup creation, write, undo, hash verification.
- Secret Scan: detects patterns, does not false-positive on normal words.

### Integration Tests
- Full flow with fake file system: pick folder, build prompt, parse answer, apply, verify files on disk.
- NEED_FILE loop: simulate 2 rounds of NEED_FILE then proposal.
- Session persistence: kill and restore, verify session resumes.

### Manual Tests
- Real sample project: small Android demo app, change UI task, end to end.
- Test on Android 10, 13, 14, with different screen sizes.
- Test with large project (200 files) for performance.
- Test with file changed on disk after proposal, verify warning.
- Test with Meta AI returning malformed answer, verify repair flow.

### Test Data
- Sample project with `file.json`, `file.txt`, `file.kt`, plus nested folders.
- Sample Meta AI answers: valid, NEED_FILE, malformed, duplicate, diff, empty.

## 29. Build, CI, and Release Process

- Versioning: Semantic versioning, e.g. 1.0.0, 1.1.0.
- Build flavors: debug, release.
- CI: GitHub Actions or similar, run unit tests on every push, build release APK on tag.
- Release: signed APK and AAB, uploaded to Play Console internal track first, then production.
- Changelog: keep CHANGELOG.md with features, fixes, known issues.
- Code style: ktlint, with CI check.

## 30. Milestones and Roadmap

### Milestone 1: Core Loop (2 weeks)
- Pick folder, tree, select file, build prompt, copy, paste answer, parse DESCRIPTION + FILE, show diff, accept, apply.
- Single file, single turn.
- Basic backup and undo.

### Milestone 2: Multi-Turn (1 week)
- NEED_FILE detection and auto-send next prompt.
- Conversation history.
- Max rounds guard.

### Milestone 3: Robust Review (2 weeks)
- Per-file Accept, Edit, Decline.
- Diff with collapsed hunks.
- Backups, Undo, Apply Summary.
- Session persistence with Room.

### Milestone 4: Polish and Safety (1 week)
- Secret scan, large file handling, permission lost handling.
- Empty states, error states, loading states.
- Accessibility pass, dark theme, performance pass.

### Milestone 5: Quality and Release (1 week)
- Full test suite, manual tests on real project.
- Play Console internal release, fix bugs, production release.

### V2 Ideas
- Chunked file sending.
- Image context.
- Export session as markdown.
- Optional LLM API integration for full automation, reusing same diff and confirm flow.
- Git integration: show git status, suggest commit message.

## 31. Risks, Assumptions, and Open Questions

### Risks
- Meta AI changes its chat format or blocks copy-paste flow. Mitigation: human-in-the-loop is still allowed, and repair prompt handles format changes.
- User pastes truncated answer. Mitigation: detect unbalanced fences, ask to paste full answer.
- Large projects produce prompts too long for Meta AI. Mitigation: limit files, chunk, and tree truncation.
- SAF is confusing for some users. Mitigation: onboarding with screenshots and clear instructions.

### Assumptions
- User has Meta AI app or web access with internet.
- User is comfortable copy-pasting between apps.
- Project files are text-based and UTF-8.
- User is the owner of the project files and has permission to edit them.

### Open Questions
- Should the app support multiple tasks in one session, or one task per session? V1: one task per session.
- Should the app support sending a file in chunks automatically? V1: no, warn and let user split task. V2: yes.
- Should the app support image files as context? V1: no. V2: yes, as description only.

## 32. Alternatives Considered

- **Direct API to Meta AI:** Not available for this chat, so rejected for V1.
- **Automated WebView scraping of meta.ai:** Fragile, breaks on layout changes, can get blocked, rejected.
- **Zip upload:** Proven to fail in testing, rejected.
- **GitHub link fetching:** Proven to return cached stale copies, rejected for fresh file reads.
- **Cursor + Claude Code:** Works well for desktop with API, but requires desktop and API key, not for phone-only workflow. The app in this spec is the phone equivalent with human bridge.

## 33. Appendices

### Appendix A: Example Full Session

```
User task: i want you to change the UI

App prompt 1:
Task: i want you to change the UI
Project tree:
app/
  file.json
  file.txt
  file.kt
Current file: file.json
Content: { "theme": "light", "padding": 8 }
Rules: ...

Meta answer 1:
okay, received, i will review it
NEED_FILE: file.txt
NEED_FILE: file.kt

App prompt 2:
Here are the files you asked for:
File: file.txt
Content: ...
File: file.kt
Content: ...
Continue...

Meta answer 2:
DESCRIPTION: Updated UI to dark theme...
FILE: file.json
```json
{ "theme": "dark", "padding": 16 }
```
FILE: file.txt
```
...
```
FILE: file.kt
```kotlin
...
```

App: shows diffs, user accepts file.json, edits file.txt, declines file.kt, applies.
```

### Appendix B: Glossary
See Section 9.

### Appendix C: File Naming
- Spec file: overview.md (this file)
- App module: app
- Main Activity: MainActivity.kt
- Key files: FileManager.kt, PromptBuilder.kt, ResponseParser.kt, DiffEngine.kt, ApplyEngine.kt, ConversationManager.kt

### Appendix D: Changelog for This Spec
- V1.0: Initial mission spec, core loop only.
- V2.0: This overview, full spec with all modules, protocols, UI, testing, roadmap, and edge cases.



---

## 34. Program to Meta Logic - Robust Specification (V2)

This section makes the Program -> Meta AI direction robust. It is the most failure-prone part of the system because Meta AI chat has no API, no streaming, no guaranteed format, and no automatic retry. The program must therefore be defensive at every step: before sending, during sending, and after receiving.

### 34.1 Goals for Robust Program to Meta Logic

- Never send a stale file. Never send a truncated file without marking it.
- Never send a prompt that is too large to be pasted or read.
- Never lose conversation context across turns.
- Never accept a malformed answer as a success.
- Never loop forever on NEED_FILE.
- Always be able to recover from a bad turn by repairing, resending, or asking the user.

### 34.2 Prompt Construction Pipeline

The program must build prompts through a pipeline, not by simple string concatenation.

**Step 1: Collect**
- Read all requested files fresh from disk at build time.
- For each file, compute: relativePath, sizeBytes, sha256 hash (first 8 chars), lineCount, isTruncated flag.
- Rebuild tree fresh at build time. If tree changed since last turn, note it in the prompt.

**Step 2: Validate**
- If any file is binary, skip it and add a note: `Skipped binary file: X`.
- If any file is empty, include it but mark `EMPTY FILE`.
- If any file contains a secret pattern, run secret scan (Section 24). If user chose Redact, redact those lines and mark `REDACTED 3 lines`.
- If any file path is not in tree, flag as unknown and ask user to confirm destination before sending.

**Step 3: Budget**
- Count characters and estimate tokens: tokens ≈ chars / 4.
- Budget: Total prompt target 9000 chars, hard limit 12000 chars (configurable).
- Priority order for inclusion:
  1. Task + Rules (always included, never truncated)
  2. Tree (always included, truncated to 500 files max)
  3. Starting files (highest priority)
  4. NEED_FILE files in order requested
  5. Extra context / error logs (lowest priority, truncated first)
- If over budget, apply truncation in order:
  - First, truncate extra context to 500 chars.
  - Second, truncate largest file with a clear marker: `... [TRUNCATED, 1200 of 3400 lines shown, ask for chunk 2 if needed] ...` and include chunk info.
  - Third, if still over budget, drop lowest-priority NEED_FILE files and add note: `Dropped file Y due to size, ask for it separately if needed`.
- Never silently truncate. Always add a truncation note so Meta AI knows it does not have the full file.

**Step 4: Fingerprint**
- For each file sent, include its hash in the prompt header:
  `File: file.json | hash: a1b2c3d4 | lines: 120 | bytes: 3400`
- This lets Meta AI echo the hash back to prove it received that version. If Meta AI echoes a wrong hash, the program knows there is a mismatch.
- Include a PromptId: `PromptId: sess-123-turn-4`. The user copies this with the prompt. When the answer is pasted, the program checks that the answer belongs to the latest PromptId. If the user pastes an answer for an old prompt, the program warns: "This answer may be for an older prompt."

**Step 5: Format**
- Use the exact templates in Section 13, plus the fingerprint header.
- Always end with the Rules block from Section 12.1, plus one few-shot example of a correct answer (short, 2 files) so Meta AI has a concrete pattern to copy. Few-shot examples increase parse success from ~70 percent to ~95 percent in testing.
- Add a final line: `Reply with DESCRIPTION + FILE blocks, or NEED_FILE lines only. Do not add extra explanation.`

**Example Robust First Prompt:**
```
PromptId: sess-abc-turn-1
Task: i want you to change the UI

Project tree:
app/
  file.json (120 lines, 3400 bytes)
  file.txt (40 lines)
  file.kt (200 lines)

File: file.json | hash: a1b2c3d4 | lines: 120
Content:
{ ... }

Rules:
... (see Section 12.1)

Example correct reply:
DESCRIPTION: Changed theme to dark.
FILE: file.json
```json
{ ... }
```
```

### 34.3 Chunking for Large Files

If a single file is over the chunk limit (default 30KB or 800 lines):
- Split it into chunks: `file.kt [chunk 1 of 3]`, `file.kt [chunk 2 of 3]`, etc.
- Send chunk 1 first with note: "This file has 3 chunks. I will send chunk 2 if you ask with NEED_CHUNK: file.kt chunk 2".
- Add support for `NEED_CHUNK: <path> chunk <n>` as an extension of NEED_FILE.
- Parser must handle NEED_CHUNK and reassemble chunks in order before building the next prompt. Never send chunk 2 without chunk 1 context unless Meta AI explicitly asks.

### 34.4 NEED_FILE Handling - Robust

- Parse NEED_FILE lines case-insensitively, trim whitespace, strip quotes and backticks.
- Deduplicate: if Meta AI asks for the same file twice, send it once.
- Batch: collect all NEED_FILE lines in one turn, send them together in one next prompt. Do not send one prompt per file.
- Limit: max 10 files per NEED_FILE round, max 8 rounds per session. If limit is hit, show user: "Meta AI has asked for files 8 times. Do you want to continue or add a follow-up to narrow the task?"
- Missing file: if requested file does not exist, do not fail the whole turn. Send a note in next prompt: `File file.txt does not exist. Updated tree: ...` and continue.
- Circular request: if Meta AI asks for a file already sent in this session, do not resend the full content. Send a short note: `File file.txt was already sent in Turn 2 with hash a1b2c3d4. Content unchanged. Do you need a different part?` This prevents prompt bloat.
- Priority: if Meta AI asks for a large file and a small file, send small file first in the prompt, so Meta AI can start reasoning while reading.

### 34.5 Acknowledgement and Hash Echo

To detect stale or mismatched files, the program asks Meta AI to echo hashes:

```
In your reply, start with:
RECEIVED: file.json:a1b2c3d4, file.txt:e5f6a7b8
```

Parser checks:
- If RECEIVED is missing, warn but do not fail. Show "Meta AI did not confirm file hashes."
- If RECEIVED hash does not match sent hash, flag as MISMATCH and show: "Meta AI may have seen an older version of file.json. Resend?" and offer to resend that file.
- This is the robust fix for the cached file problem: even if the app sends fresh files, Meta AI might still be confused across turns. Hash echo makes that visible.

### 34.6 Retry and Repair Logic

The program must have a deterministic retry flow, not just "try again".

**Level 1: Auto-Repair Prompt (no user action except copy-paste)**
If parser finds:
- No DESCRIPTION and no FILE and no NEED_FILE -> send Repair Prompt (Section 12.5) with the original task and tree, but not the full files again to save tokens. Include: "Your last answer was not parseable. Please resend in the exact format."
- FILE block with no closing fence -> send: "Your FILE block for file.json was missing a closing fence. Please resend only that file with a closing fence."
- FILE block with diff instead of full file -> send: "You returned a diff, not a full file. Please resend the full file for file.json."

**Level 2: Narrow Retry**
If Meta AI fails twice on the same task, the program suggests narrowing:
- Split task: "Change only file.json, ignore other files for now."
- Or reduce files: send only 1 file instead of 3.
- The app shows a button: "Retry with 1 file only" which builds a minimal prompt.

**Level 3: User Intervention**
If 3 repairs fail, the program stops auto-retry and shows the raw answers and asks the user to write a manual follow-up or to edit the task. It never loops forever.

**Retry Counter:**
- Keep `repairCount` per session. Max 3 auto-repairs per turn. After 3, require user to edit task or files.
- Log every repair attempt in Turn history for debugging.

### 34.7 Context Management Across Turns

- The program must not resend the entire history every turn, or prompts will explode.
- It keeps a compact session summary:
  ```
  Session summary:
  - Task: change the UI
  - Files sent so far: file.json (hash a1...), file.txt (hash e5...), file.kt (hash 9c...)
  - Last proposal: DESCRIPTION: ...
  - Pending: waiting for NEED_FILE for file.kt
  ```
- Each new prompt includes: Task + Tree + Session Summary (max 500 chars) + New Files for this turn (full content) + Reference to old files by hash (not full content).
- This keeps prompts small while preserving context. If Meta AI asks for an old file again, the program can resend it, but it does not resend all old files every turn.

### 34.8 Validation Before Sending

Before the Copy button is enabled, the program runs a pre-send checklist:
- [ ] At least one file selected and non-empty or explicitly marked empty
- [ ] Task is non-empty
- [ ] Tree is fresh (built within last 60 seconds)
- [ ] All file hashes computed
- [ ] No binary files included
- [ ] Secret scan done, user decision recorded
- [ ] Prompt size within limit or truncation notes added
- [ ] PromptId generated and included

If any check fails, the app shows the specific failure and blocks Copy until fixed. This prevents sending a bad prompt that will waste a Meta AI turn.

### 34.9 Handling Meta AI Quirks

- **Meta AI returns extra explanation:** Parser ignores explanation, extracts DESCRIPTION and FILE blocks. No failure.
- **Meta AI returns markdown headings or emojis:** Parser strips them for DESCRIPTION, keeps file content exact.
- **Meta AI returns code in multiple fences for one file:** Parser concatenates them in order and warns "Multiple fences for file.json, concatenated".
- **Meta AI renames a file:** If FILE path differs from original by only case or folder, flag as rename and ask user to confirm. Do not auto-rename.
- **Meta AI returns a file with Windows line endings:** Normalizer converts CRLF to LF before diff and before write, and logs it.
- **Meta AI returns a file with trailing whitespace:** Parser trims trailing whitespace on each line for diff, but preserves it for write unless user enables "Trim trailing whitespace" setting.

### 34.10 Fallbacks and Circuit Breakers

- **Circuit breaker:** If 3 consecutive turns fail to parse, the app pauses and shows "Meta AI is not following the format. Try a simpler task or check if Meta AI is having issues." It does not keep building prompts.
- **Fallback to single-file mode:** If multi-file proposal keeps failing, the app offers "Switch to single-file mode" which sends only the main file and asks Meta AI to only edit that file.
- **Fallback to manual mode:** If the user is stuck, the app offers "Show raw answer" and "Let me edit manually" which opens the Editor with the raw answer pasted as reference, so the user can manually copy code.
- **Timeout:** There is no network timeout because the bridge is manual, but the app tracks time since prompt copied. If more than 30 minutes have passed without an answer, it shows "This prompt may be stale. Rebuild prompt with fresh files?" and offers to rebuild.

### 34.11 Logging for Program to Meta Logic

For every prompt built, log (local only, no file contents in analytics):
- PromptId, timestamp, task hash, tree hash, files sent with hashes and sizes, prompt char count, truncation notes, repairCount.
For every answer parsed, log:
- Answer char count, parse success, number of FILE blocks, number of NEED_FILE, number of warnings, parse errors.
This log is the debugging tool when a session fails. The user can export it as text to see exactly what was sent and what was received.

### 34.12 Example Robust Session with Failures Handled

1. App builds PromptId turn-1 with file.json, hash a1..., sends to Meta.
2. Meta replies with no FILE blocks, just explanation. Parser fails, repairCount = 1. App builds Repair Prompt, user copies it.
3. Meta replies with FILE block but no closing fence. Parser fails, repairCount = 2. App builds specific repair: "Missing closing fence for file.json".
4. Meta replies with NEED_FILE: file.txt. App sends file.txt, hash e5..., in next prompt.
5. Meta replies with DESCRIPTION + 2 FILE blocks, but hash echo shows wrong hash for file.json. App warns "Hash mismatch, Meta may have seen old version" and offers to resend file.json. User taps Resend.
6. Meta replies with correct DESCRIPTION + FILE blocks, hashes match. App shows diff, user accepts, app applies.

This is robust because every failure has a specific, named recovery path, not a generic "try again".

### 34.13 Checklist for Implementation

Before marking Program to Meta logic as done, verify:
- [ ] Prompt always includes PromptId, task, tree, file hashes, rules, and few-shot example.
- [ ] Prompt is validated before Copy is enabled.
- [ ] Truncation is always marked, never silent.
- [ ] NEED_FILE is deduplicated, batched, and limited to 8 rounds.
- [ ] NEED_CHUNK is supported for large files.
- [ ] Hash echo is requested and checked.
- [ ] Parser handles malformed answers with specific repair prompts.
- [ ] Retry is limited to 3 per turn, then requires user intervention.
- [ ] Session summary is included to keep context without resending all files every turn.
- [ ] All prompts and answers are logged locally with hashes, not just raw text.
- [ ] User can see exactly what was sent and what was received at any time.



---

## 35. Program to Meta Conversation Logic - Robust Specification (V3)

This section makes the *conversation* itself robust, not just the single prompt. A real task is not one prompt and one answer. It is a multi-turn conversation with requests, clarifications, corrections, retries, branches, and interruptions. If the conversation logic is fragile, the whole app feels fragile.

### 35.1 Conversation Goals

- Never lose context when the user switches apps to copy-paste.
- Never confuse an answer for Turn 3 with a prompt for Turn 5.
- Never resend the same file 5 times and blow up the prompt.
- Never get stuck when Meta AI asks a question instead of returning code.
- Always let the user go back, fork, or correct a turn without losing work.
- Always be able to resume after the app is killed.

### 35.2 Conversation Manager Responsibilities

The ConversationManager is a single owner for one TaskSession. It owns:

- Turn list in order, with PromptId, prompt text, answer text, files sent, hashes, timestamps.
- Working Set: the set of files Meta AI has seen in this session, with hash per file.
- Session Summary: compact summary that is sent in every prompt to keep context small.
- Pending Queue: NEED_FILE / NEED_CHUNK / NEED_CLARIFICATION items that have not yet been answered.
- Repair Count: per-turn and per-session counters for circuit breaker.
- Branch History: if the user forks a session, keep the parent link.

No other module is allowed to build a prompt directly. All prompts go through ConversationManager.buildNextPrompt() so the logic stays in one place.

### 35.3 Turn Lifecycle - Robust

Every turn goes through these states, with explicit transitions and timeouts:

```
CREATED -> PROMPT_BUILT -> COPIED -> WAITING_FOR_ANSWER -> ANSWER_PASTED -> PARSING -> CLASSIFIED -> NEXT_ACTION
```

- CREATED: Turn object created, PromptId assigned (e.g. sess-abc-turn-4), task snapshot saved.
- PROMPT_BUILT: Prompt text built, hashes computed, validation passed (Section 34.8). If validation fails, turn stays in CREATED and shows error.
- COPIED: User tapped Copy. Record copyTime. If user taps Copy again, do not create a new turn, just update copyTime. Copy is idempotent.
- WAITING_FOR_ANSWER: App is waiting. Show hint: "Paste Meta AI answer here". If user leaves app and comes back after 30 min, show stale warning: "This prompt is 32 min old. Files may have changed. Rebuild with fresh files?" with Rebuild button.
- ANSWER_PASTED: User pasted answer. Check: Is answer empty? Is answer identical to previous answer? If duplicate, warn: "This looks like the same answer as Turn 3. Did you paste the wrong answer?" and ask to confirm.
- PARSING: Parser runs. If it takes > 2 sec, show progress.
- CLASSIFIED: Answer is classified into one of the types in 35.4. If classification fails, go to REPAIR.
- NEXT_ACTION: One of: Build Next Prompt (NEED_FILE), Show Proposal (PROPOSAL), Ask User (NEED_CLARIFICATION), Repair (MALFORMED), or Fork (user chose to fork).

If the app is killed in any state, on restart it restores the turn in the same state from Room, with prompt and answer preserved. The user never loses a turn.

### 35.4 Answer Classification - All Cases Meta AI Can Return

The parser must not assume Meta AI always returns DESCRIPTION + FILE. It can return 7 types. Each type has a different next action.

**Type 1: PROPOSAL**
- Contains DESCRIPTION + at least 1 FILE block.
- Next action: Show Review Screen.
- Validation: Check hashes echo (Section 34.5). If mismatch, show warning but still show proposal, with option to Resend.

**Type 2: NEED_FILE**
- Contains 1 or more NEED_FILE lines, no FILE blocks.
- Next action: Build Next Prompt with requested files.
- Robust: Deduplicate, batch, check if file already in Working Set with same hash (Section 34.4). If already sent, send short note instead of full content.

**Type 3: NEED_CHUNK**
- Contains NEED_CHUNK: path chunk N.
- Next action: Build Next Prompt with that chunk only.
- Robust: Verify chunk number exists. If Meta asks for chunk 5 of a 3-chunk file, reply: "That chunk does not exist, file has only 3 chunks."

**Type 4: NEED_CLARIFICATION / QUESTION**
- Meta AI asks a question, e.g. "Do you want dark theme or light theme? Should I keep the old padding?"
- Old logic would treat this as malformed and fail. Robust logic treats it as a valid conversation turn.
- Next action: Show Clarification Screen with the question extracted, and options for the user to answer. The user types an answer, the app builds a Clarification Prompt:
  ```
  PromptId: ...
  You asked: Do you want dark or light theme?
  User answer: Dark theme, keep old padding.
  Continue with original task. Use DESCRIPTION + FILE format.
  ```
- The app must not require the user to manually copy the question. It extracts the question and shows it.

**Type 5: PARTIAL_PROPOSAL**
- Contains DESCRIPTION + FILE blocks, but one FILE block is truncated or missing closing fence.
- Next action: Keep the good files, ask Meta to resend only the bad file. Do not discard the whole proposal.
- Example: "Got 2 good files, 1 bad file (file.kt missing fence). Ask Meta to resend only file.kt?"
- This is critical: discarding a whole proposal because one file is bad wastes a turn.

**Type 6: DIFF_INSTEAD_OF_FULL_FILE**
- Contains diff markers (+ / -) instead of full file content.
- Next action: Try to apply diff locally if possible, or ask Meta to resend full file. The app shows: "Meta returned a diff, not a full file. I can try to apply it, or ask Meta to resend full file." User chooses.
- If user chooses Try Apply Diff, the Diff Engine attempts to patch the original file. If patch fails, fall back to asking for full file.

**Type 7: MALFORMED / OFF_TOPIC**
- No DESCRIPTION, no FILE, no NEED_FILE, no question. Just explanation or refusal.
- Next action: Repair Prompt (Section 34.6). If repairCount >= 3, go to User Intervention.

Classification is done by the parser in order: Check NEED_FILE first, then FILE, then QUESTION, then DIFF, then MALFORMED. A turn can be both NEED_FILE + QUESTION. In that case, handle NEED_FILE first, then QUESTION in the same next prompt.

### 35.5 Context Management - Three Layers

To keep prompts small but context intact, the conversation uses three layers:

**Layer 1: Working Set (What Meta Has Seen)**
- Map: relativePath -> hash, lineCount, lastSentTurn.
- Sent in every prompt as a compact list, not as full content:
  ```
  Files you have already seen:
  - file.json (hash a1b2c3d4, Turn 1)
  - file.txt (hash e5f6a7b8, Turn 2)
  ```
- If Meta needs a file again, it can ask by hash, and the app resends it. Otherwise, the app does not resend it.

**Layer 2: Session Summary (What Happened)**
- Max 500 chars, updated after each turn by the app (not by Meta).
- Template:
  ```
  Task: change UI to dark theme.
  Turns: 3. Files sent: file.json, file.txt, file.kt.
  Last proposal: Changed theme to dark, padding 16.
  Pending: None. User feedback: Make buttons smaller.
  ```
- The summary is built deterministically from Turn history, not by asking Meta to summarize, so it is always accurate.

**Layer 3: Recent Turns (What Just Happened)**
- Keep full text of last 2 turns (prompt + answer) in memory, but only send the last turn's answer summary in the next prompt, not the full text.
- If the user taps "Show full history", the app shows it locally, but does not send it to Meta.

This prevents the classic failure where prompts grow to 30KB after 5 turns and Meta stops reading them.

### 35.6 Working Set and Hash Consistency

- Every time a file is sent, store its hash in Working Set.
- Before building the next prompt, re-read the file from disk and compare hash.
- If file changed on disk since last sent (user edited it outside the app, or a previous Apply changed it), the app must:
  1. Show warning: "file.json changed on disk since you sent it to Meta (Turn 2). Meta is working on an old version."
  2. Offer: "Send updated file to Meta" (resend with new hash) or "Keep old version" (do not resend, but note it).
  3. If user chooses Send Updated, the app builds a prompt: "File file.json was updated. Here is the new version with hash 9c... Please update your proposal."
- This prevents Meta from proposing changes based on stale content, which is the most common cause of broken proposals.

### 35.7 Branching, Forking, and Going Back

A robust conversation lets the user change their mind without losing work.

- **Go Back:** From Review Screen, user can tap "Back to Turn 2" to see the prompt and answer for Turn 2. They can tap "Fork from here" to create a new session that copies turns up to Turn 2, with a new SessionId and parent link. The original session is kept untouched.
- **Fork on Feedback:** If the user gives follow-up feedback that contradicts the last proposal ("Actually, I want light theme, not dark"), the app asks: "Do you want to fork a new proposal or overwrite the last one?" Fork keeps the old proposal in history, overwrite replaces it. Default is Fork.
- **Undo Turn:** If the user pasted the wrong answer, they can tap "Undo Last Paste" which removes the last turn's answer and returns to WAITING_FOR_ANSWER for that turn. The prompt is still there, so they can paste the correct answer.
- **Duplicate Session:** User can duplicate a session to try two different tasks on the same project without interfering.

### 35.8 Clarification Loop - Robust

Meta AI often asks clarifying questions. The app must handle this as a first-class flow, not as an error.

Steps:
1. Parser detects question: lines ending with `?` or starting with `Question:` or `Clarify:`.
2. App extracts up to 3 questions, shows them in Clarification Screen with text fields for user answers.
3. User answers one or all questions, taps "Send Answers to Meta".
4. App builds Clarification Prompt (Section 35.4 Type 4) with original task + questions + answers + working set.
5. If Meta asks another question, loop again, max 3 clarification rounds. After 3, app shows: "Meta has asked 3 clarification rounds. Do you want to continue or proceed with best guess?" User can choose "Proceed with best guess" which sends: "Proceed with your best guess, do not ask more questions, send a proposal now."

### 35.9 Follow-up and Correction Loop

After a proposal is shown, the user can give feedback: "Make buttons smaller, keep dark theme."

Robust handling:
- The app does not discard the previous proposal. It keeps it as Proposal 1, and builds a new prompt for Proposal 2.
- The new prompt includes:
  ```
  Previous DESCRIPTION: Changed theme to dark...
  User feedback: Make buttons smaller, keep dark theme.
  Files already sent: file.json (hash...), file.txt (hash...)
  Please send a new proposal that incorporates the feedback. Only return files you changed for this feedback.
  ```
- If the feedback mentions a file not in Working Set, the app automatically adds it to the next prompt's NEED_FILE queue.
- Max feedback rounds: 5 per session. After 5, app suggests starting a new session with a narrower task.

### 35.10 Interruption and Resume

- **App killed:** Session is saved to Room after every turn. On restart, the app shows "You have an active session for project X, Turn 4, waiting for answer. Resume?" with Resume and Discard buttons.
- **User switches project mid-session:** App saves current session as PAUSED, and loads the new project. The user can return to the paused session from History.
- **User copies wrong prompt:** If the user copies PromptId turn-4 but pastes an answer for turn-3, the app detects PromptId mismatch if the answer contains a PromptId, or duplicate detection if the answer is identical to a previous answer, and warns.
- **User edits a file on disk mid-conversation:** Detected by hash check in 35.6, with warning and resend option.

### 35.11 Conflict Resolution

Three conflicts can happen:

- **Conflict A: File changed on disk vs Meta proposal.** If the file changed on disk after the proposal was built, the app shows: "file.json changed on disk since proposal. Original hash a1..., current hash 9c..., proposal hash a1... Do you want to overwrite with proposal or refresh proposal?" User chooses Overwrite or Refresh.
- **Conflict B: Two proposals for same file.** If the user forked and now has two proposals for file.json, the app shows both diffs side by side and asks which to apply, or to merge manually in Editor.
- **Conflict C: User edited proposal in Editor vs Meta new proposal.** If the user edited Proposal 1 in Editor, and then Meta sends Proposal 2, the app keeps the user edit as "User Edited Version" and shows it as a separate option: "Use Meta Proposal 2, Use Your Edited Version, or Merge in Editor". It never silently discards user edits.

### 35.12 Conversation Circuit Breakers

- **NEED_FILE loop breaker:** Max 8 NEED_FILE rounds. After 8, app shows: "Meta has asked for files 8 times. This task may be too broad. Try splitting it: e.g. Change only file.json first." with button "Split Task".
- **Clarification loop breaker:** Max 3 clarification rounds (see 35.8).
- **Repair loop breaker:** Max 3 repairs per turn (Section 34.6).
- **Feedback loop breaker:** Max 5 feedback rounds per session.
- **Global breaker:** If total turns > 20 in one session, app suggests: "This session has 20 turns. Consider starting a new session with a narrower task to keep prompts small." It does not block, but warns.

### 35.13 Conversation Logging and Debugging

For each turn, log locally:
- PromptId, timestamp, copyTime, pasteTime, timeBetweenCopyAndPaste, promptCharCount, answerCharCount, classification, repairCount, workingSetHashes, promptHash.
- The user can tap "Export Conversation Log" to get a markdown file with all prompts and answers, with file contents redacted if secret scan is enabled, for debugging or sharing.

### 35.14 Conversation UI - Robust

- **Turn List:** Shows each turn as a card: Turn #, classification badge (Proposal, Need File, Question, Repair), files sent, time, and status. Tap to expand full prompt and answer.
- **Current Turn Banner:** At top of Prompt Screen, shows: "Turn 4 of Session X | Waiting for answer | Prompt copied 2 min ago" with Rebuild and Cancel buttons.
- **Pending Queue View:** Shows: "Meta needs: file.txt, file.kt (2 files) | Clarification: 1 question pending" with Send button.
- **Stale Warning:** If prompt is > 30 min old, banner turns amber with Rebuild button.
- **Duplicate Warning:** If pasted answer is duplicate, show amber banner: "This looks like a duplicate of Turn 2. Paste anyway?"
- **Empty State:** If no turns yet, show "Start by picking files and typing a task."

### 35.15 Implementation Checklist for Conversation Logic

- [ ] ConversationManager is the single owner of turns, working set, and summary.
- [ ] Every turn has PromptId, state, hashes, timestamps, and is saved to Room immediately.
- [ ] Answer classification handles all 7 types, not just Proposal and NEED_FILE.
- [ ] Partial proposals are preserved, not discarded.
- [ ] Clarification is a first-class flow, not an error.
- [ ] Working Set prevents resending the same file with same hash.
- [ ] Hash check detects file changed on disk mid-conversation.
- [ ] Branching, forking, and Undo Last Paste are supported.
- [ ] All loop breakers (NEED_FILE 8, clarification 3, repair 3, feedback 5, total 20) are enforced.
- [ ] Stale prompt warning after 30 min.
- [ ] Duplicate answer detection.
- [ ] Resume after app kill works from Room.
- [ ] Conversation log export works.

### 35.16 Example Robust Conversation

```
Turn 1: User task "change UI"
  App sends file.json (hash a1...)
  Meta replies: NEED_FILE: file.txt
  App sends file.txt (hash e5...)

Turn 2: Meta replies: Question: Do you want dark or light theme?
  App shows Clarification Screen, user answers Dark.
  App sends clarification prompt.

Turn 3: Meta replies: DESCRIPTION + FILE file.json (good) + FILE file.kt (missing fence)
  App keeps file.json, asks Meta to resend only file.kt.
  RepairCount = 1

Turn 4: Meta replies: DESCRIPTION + FILE file.kt (good)
  App shows Review with 2 files, user edits file.json, accepts file.kt, applies.

Turn 5: User feedback: "Make buttons smaller"
  App sends feedback prompt, keeps Proposal 1, builds Proposal 2.
```

This conversation would have failed with simple logic (question treated as malformed, partial proposal discarded, file resent every turn). With robust logic, it succeeds.


# BPMN: Session Resume & Fork Flow

This diagram describes how a user discovers, browses, and resumes (or forks) existing Claude Code sessions from the dashboard. The system reads Claude's JSONL session files from disk, presents previews and metadata, and builds the appropriate CLI command for resuming or forking a conversation.

```mermaid
flowchart TD
    Start([User views Dashboard]) --> ScanCards[Dashboard loads /api/projects\neach project scanned for sessions]
    ScanCards --> EncodePath[For each project:\nencode path as -home-bacon-project]
    EncodePath --> CheckDir{~/.claude/projects/\nencoded-path/\nexists?}
    CheckDir -- No --> NoBadge[No session badge\non project card]
    CheckDir -- Yes --> CountJSONL[Count *.jsonl files\nsorted by mtime desc]
    CountJSONL --> ShowBadge[Show session count badge\non project card]

    ShowBadge --> UserClicks[User clicks session badge]
    UserClicks --> OpenModal[Session browser modal opens]
    OpenModal --> FetchSessions[GET /api/sessions/group/project\ninclude_preview=True]

    FetchSessions --> ParseFiles[For each .jsonl file:\n- Extract session ID from filename\n- Read file stat mtime + size\n- Extract first user message as preview\n- Count user + assistant messages]
    ParseFiles --> DisplayList[Display session list\nnewest first\nshowing preview + message counts]

    DisplayList --> UserSelectsSession[User clicks a session row]
    UserSelectsSession --> FetchDetail[GET /api/session-preview/\ngroup/project/session_id\nReturns up to 10 user messages]
    FetchDetail --> ShowDetail[Show session detail panel\n- All preview messages\n- Message counts\n- File size & last modified]

    ShowDetail --> UserDecision{User chooses action}

    UserDecision -- Resume --> BuildResume[Build command:\nclaude --resume SESSION_ID]
    UserDecision -- Fork --> BuildFork[Build command:\nclaude --resume SESSION_ID\n--fork-session]
    UserDecision -- Cancel --> CloseModal([Modal closed])

    BuildResume --> SelectTheme[User confirms theme selection]
    BuildFork --> SelectTheme

    SelectTheme --> LaunchAPI[POST /api/launch/group/project\ncommand = built CLI string\ntheme = selected theme]
    LaunchAPI --> StandardLaunch[Standard launch flow:\nkill existing -> write settings ->\nstart code-server -> tasks.json\nruns the resume/fork command]
    StandardLaunch --> SessionActive([Claude resumes conversation\nin code-server terminal])

    style Start fill:#1565c0,color:#fff
    style SessionActive fill:#2e7d32,color:#fff
    style CloseModal fill:#616161,color:#fff
```

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| **Session detection via JSONL files** (ADR-004) | Claude Code stores sessions as JSONL files in ~/.claude/projects/encoded-path/. Reading these directly avoids any dependency on Claude's internal APIs. |
| **Path encoding: / becomes -** (L-004) | Claude encodes project paths by replacing slashes with dashes (e.g., /home/bacon/foo becomes -home-bacon-foo). The dashboard must match this encoding exactly. |
| **Preview extraction limited to user messages** | Only messages with type="user" are shown in previews. Assistant messages are counted but not displayed, keeping the preview concise and meaningful. |
| **Two-tier fetch: list then detail** | The session list endpoint returns minimal data (1 preview per session). The detail endpoint returns up to 10 messages. This keeps the initial modal load fast when projects have many sessions. |
| **Resume vs Fork uses same launch flow** | Both operations build a CLI command string that gets baked into bacon-start.sh. The only difference is whether --fork-session is appended. This reuses the entire launch infrastructure. |
| **Sessions sorted newest-first by mtime** | The most recently active session is most likely what the user wants to resume, so it appears at the top of the list. |

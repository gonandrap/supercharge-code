---
name: plugin-review
description: Comprehensive security and behavioral review of Claude Code plugins and extensions. Use this skill whenever the user asks to review, audit, check, or validate a plugin, extension, or hook for Claude Code — even if they just say "is this safe to install?", "what does this plugin do?", "can you check this extension?", or "review this for me". Produces a structured markdown report with a full checklist covering security hardening, behavioral transparency, and installation safety, documenting both passing and failing findings.
---

# Plugin Review

You are conducting a security and behavioral review of a Claude Code plugin or extension. Your goal: help the user understand exactly what the plugin does and whether it is safe to install and use.

Produce a thorough markdown report covering every checklist item below. Mark each ✅ PASS, ❌ FAIL, or ⚠️ NOTE. Document both good findings and problems — a passing check is valuable information; it tells the user the developer thought about that attack surface.

---

## Phase 1: Build the Inventory

Read every file before running any checks. Map what the plugin contains:

- **Hooks**: SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Stop, SubagentStop — list each one and what event it handles
- **Skills**: any SKILL.md files and what they instruct the model to do
- **Install/uninstall scripts**: shell scripts, PowerShell scripts
- **Shared modules**: config resolvers, utilities
- **External dependencies**: package.json, requirements.txt, imported libraries
- **Bundled executables or scripts**

---

## Phase 2: Security Checklist

Work through each item by reading the actual code. Cite file:line for every finding.

### File System Safety

| Check | What to look for |
|---|---|
| **Symlink protection on writes** | Does the writer check `lstatSync`/`lstat` for symlinks at the target file *and* its parent directory before writing? Prevents a local attacker from replacing a predictable path (like `~/.claude/.some-flag`) with a symlink to clobber user files. |
| **Symlink protection on reads** | Does every reader also check for symlinks before reading? A symlinked flag pointing at `~/.ssh/id_rsa` would inject key bytes into model context or the terminal. |
| **Path traversal** | Are file paths constructed from hardcoded segments or sanitized inputs only? Look for user-controlled strings in `path.join()` or string concatenation that forms a path. |
| **Predictable temp file names** | If temp files are created, do they include PID and/or timestamp to avoid race conditions? (`/tmp/foo` is bad; `/tmp/foo.${pid}.${Date.now()}` is fine.) |
| **Atomic writes** | Are files written via temp-file + rename rather than truncate-in-place? Direct `writeFileSync(target)` is not atomic. |
| **File permissions** | Are created files given restrictive permissions (e.g., mode `0600`)? |
| **Read size caps** | Are file reads capped at a sensible byte limit to prevent reading multi-MB content from a swapped-in symlink or an oversized flag? |

### Input Validation

| Check | What to look for |
|---|---|
| **Whitelist for mode/config values** | Are string values from files or env vars validated against an explicit allowlist before being acted on or written? |
| **JSON parsing in try/catch** | Is JSON from stdin, config files, or external sources always parsed inside a try/catch (or equivalent)? A malformed input should silently fail, not crash the hook and block session start. |
| **Terminal output sanitization** | Is anything that reaches the terminal or statusline stripped of control characters and ANSI escape sequences? A flag file containing `\033]0;attacker title\007` could spoof the terminal title or worse. |

### Code Execution Safety

| Check | What to look for |
|---|---|
| **No shell injection** | Do subprocess calls use array arguments (e.g., `subprocess.run(["cmd", arg])`)? Shell-string interpolation (`shell=True`, backticks, `exec "cmd $var"`) with any dynamic content is a red flag. |
| **No dynamic eval** | Is `eval()`, `exec()`, `Function()`, `pickle.loads()`, or `yaml.load()` (unsafe loader) used with any non-hardcoded input? |
| **No dynamic require/import from user-controlled paths** | Does the plugin `require()` or `import` any path that could be influenced by user input or external files? |

### Network & External Calls

| Check | What to look for |
|---|---|
| **Network calls disclosed** | Does the plugin make any outbound HTTP/network requests at runtime (not just during install)? List every endpoint. |
| **HTTPS only** | Are all network calls made over HTTPS? |
| **No prompt or file exfiltration** | Does any user data — prompt text, file contents, settings — leave the machine during normal operation? |
| **Installer download integrity** | If the installer fetches files from the internet, does it verify checksums or signatures? |

### Installation Safety

| Check | What to look for |
|---|---|
| **Settings.json backup** | Does the installer back up `settings.json` before modifying it? |
| **No privilege escalation** | Does the installer avoid `sudo`, `runas`, or any admin-level operations? |
| **Idempotent install** | Does re-running the installer avoid duplicating hooks in `settings.json`? |
| **Clean uninstall** | Does the uninstaller remove all installed files and settings entries, leaving no residue? |

---

## Phase 3: Behavioral Checklist

This section answers: *what does this plugin actually do to my Claude sessions?*

### What Gets Injected Into Your Sessions

| Check | What to look for |
|---|---|
| **SessionStart injection** | Read the SessionStart hook's stdout output path. What text does it write? Summarize or quote it. Is it style rules, system context, capability expansion, or something else? |
| **Per-turn injection** | Does the UserPromptSubmit hook emit `hookSpecificOutput` or `additionalContext`? What does it say on each user message? |
| **Injection matches documentation** | Is what gets injected consistent with what the README and plugin description say the plugin does? |

### Prompt Handling

UserPromptSubmit hooks receive the user's prompt text via stdin. This is normal — but what the plugin *does* with it matters.

| Check | What to look for |
|---|---|
| **Does it read prompt text?** | Does the hook parse stdin and access the `prompt` field? (Expected — document it.) |
| **Does it modify prompts?** | Can the hook alter the prompt before it reaches the model? Look for `modifiedPrompt` in the hook's output JSON. |
| **Does it log or store prompts?** | Does it write prompt content to any file, external endpoint, or persistent store? |

### Data Access

| Check | What to look for |
|---|---|
| **Files accessed beyond plugin dir** | What files outside the plugin's own directory does it read? (`settings.json`, flag files, `SKILL.md`, config files — list them all.) |
| **Settings.json changes** | What keys does it add or modify in `settings.json`? |
| **Persistent state on disk** | What files does it create or maintain during normal operation? Where are they? |

### Claimed vs. Actual Behavior

| Check | What to look for |
|---|---|
| **README accuracy** | Does the code match the README's description of what the plugin does? Read both. |
| **Hidden behavior** | Is there anything the plugin does that is not mentioned anywhere in its documentation? |
| **Scope creep** | Does it do anything clearly outside its stated purpose? |

---

## Report Format

```markdown
# Plugin Review: [Plugin Name]

**Date:** [today]
**Plugin source:** [path or URL]

## Summary
[2–3 sentences: safe/unsafe/conditional, and the key reason why]

## Inventory
[Bulleted list of all components found with a one-line description of each]

## Security Checklist

| Check | Result | Notes |
|---|---|---|
| Symlink protection on writes | ✅ PASS | Uses lstatSync at target + parent; O_NOFOLLOW; atomic temp+rename |
| Symlink protection on reads | ✅ PASS | readFlag() checks lstat before open |
| ... | | |

## Behavioral Checklist

| Check | Result | Notes |
|---|---|---|
| SessionStart injection | ✅ DISCLOSED | Injects style rules from SKILL.md, filtered to active level |
| Prompt modification | ✅ NO | Hook reads prompt to detect commands; does not modify it |
| ... | | |

## Detailed Findings

For each non-trivial result (pass or fail), explain:
- What was checked and why it matters
- What was found, with file:line citations
- For failures: the concrete risk and a fix recommendation
- For notable passes: why this specific implementation is sound (not just "looks fine")

## Verdict

[Overall assessment. If safe: any conditions or caveats. If unsafe: what specifically to avoid and whether a fix would make it acceptable.]
```

---

## Output

Save the completed report as `security_audit.md` in the current working directory. After saving, print the full path so the user knows where to find it.

---

## Guidance

- **Silent-fail on hook errors is good**: hooks that crash would block session start or prompt submission. If a hook wraps everything in try/catch and exits cleanly on errors, note it as a positive.
- **`CLAUDE_CONFIG_DIR` respect**: good plugins use `process.env.CLAUDE_CONFIG_DIR || path.join(os.homedir(), '.claude')` rather than hardcoding `~/.claude`. Note if they do or don't.
- **Hook event capabilities** — know what each hook can do:
  - *SessionStart*: injects system context (stdout), runs once per session
  - *UserPromptSubmit*: receives prompt text via stdin, can inject per-turn context or block/modify prompt
  - *PreToolUse / PostToolUse*: intercepts tool calls — highest-risk hook type, can see all tool inputs/outputs
  - *Stop / SubagentStop*: runs when session or subagent ends
- **Flag suspicious patterns even if not exploitable**: if a developer made a structural mistake that happens to be safe in this version but could easily become unsafe, note it as ⚠️ NOTE rather than ignoring it.

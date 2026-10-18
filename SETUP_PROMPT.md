# Garden setup prompt

## How to use

Paste this whole file into the other project's agent chat. The agent should create the garden there and then stop.

## Agent instructions

Set up this garden in the current repository only.

Do this:

- Create the files below.
- If `AGENTS.md` already exists, add the Garden section. Do not remove existing project rules.
- Read this repo and add a short Stack section and a Commands table. Do not copy another app's stack.
- If a read-only check command exists, you may add that one command to the hook allowlist. Do not allow install, dev, build, or publish before the plan is approved.
- Append the gitignore lines if they are missing. Do not replace the whole gitignore.
- Create the skill folder and the two symlinks.
- Save this prompt as `.garden/SETUP_PROMPT.md` too.
- Run the verify command. Then stop.

Do not do this:

- Do not run any git command. The user runs git, including status and add.
- Do not delete a file.
- Do not touch a database.
- Do not leave this repository.
- Do not create `.garden/PLAN_APPROVED`.
- Do not build product features.
- Do not weaken the hook.

Chat replies for this user use a Hindi and English mix. Every file you write stays in English. No em dash.

## AGENTS.md section

Add this section. Keep one copy of the rules. Other agent files only point here.

```markdown
## Garden

These rules are mandatory. Hooks call `.agents/hooks/garden-guard.py` and stop a matching action. If an agent tries anyway, the hook stops it. Agents must not edit the garden hook, its settings, or the wrapper files. The user edits those by hand.

1. Do not execute a step until the user allows that step. A question is not approval.
2. Do not create or edit files, and do not run a build or dev command, until the user creates `.garden/PLAN_APPROVED`. An agent must not create that file. Read-only checks can still run.
3. Do not run any git command. No status, diff, add, commit, push, pull, or any other git command. A full path, quotes, or a hidden spelling does not make it allowed. Do not read or write `.git`. Do not run `gh`. The user runs every git command by hand.
4. Do not touch a database. No client, connection string, SQL, migration, or delete.
5. Do not delete a file, folder, or record. No `rm`, trash, clean, unlink, or delete tool. A hidden delete command is still blocked.
6. Do not bypass these rules. Do not change hook files, do not decode a command into a shell, and do not pipe a command into a shell.
7. Stay inside this repository. Do not read or write a path outside it.
8. Do not send data out. No `curl`, `wget`, `ssh`, `scp`, or similar network command.
9. Do not publish. No `npm publish`, `pnpm publish`, `cargo publish`, or `docker push`.
10. Do not write a secret, key, token, or password into a file or a command.
11. If something else is needed, ask the user before doing it.

A blocked try is appended to `.garden/blocked.log`. The log stores the time, tool, and reason. It does not store the command.

## Response style

Use this in every chat reply.

| Rule | Meaning |
|---|---|
| No over-explain | Get to the point. Skip extra background. |
| Simple words | Use easy words. Avoid heavy jargon. |
| Hinglish in chat | Chat replies use a Hindi + English mix. |
| English in the repo | Every file in the repo stays in English. |
| No long paragraphs | Break the answer into short pieces. |
| Points and tables | Use bullets or tables. |
| Proper spacing | Leave a blank line between blocks. Do not pack text together. |
| Crisp | Say only what is needed. |
| No em-dash | Do not use an em-dash. Use a comma or a period. |

## Working rules

- Match the code around the change. Keep the diff small.
- Do not add a dependency unless the user asks.
- Do not commit secrets, tokens, or local env files.
- After a code change, run this repo's read-only check when it is reasonable.
- New skills go only in `.agents/skills/<name>/SKILL.md`.
- Do not copy a skill into `.claude/skills` or `.grok/skills`. Those two paths are links to `.agents/skills`. Do not add `.cursor/skills`.
- Workflows are not shared across tools. Grok workflows are Rhai files in `.grok/workflows/`. Antigravity workflows are Markdown in `.agent/workflows/`. Do not symlink them to each other.
- Change shared rules only in `AGENTS.md`.

## Who reads what

| Tool | File to keep |
|---|---|
| Codex, Cursor, Grok, Antigravity, Copilot, OpenCode | `AGENTS.md` |
| Claude Code | `CLAUDE.md` imports `AGENTS.md` |
| Gemini CLI | `.gemini/settings.json` loads `AGENTS.md` |
| Copilot chat | `.github/copilot-instructions.md` points here |
| Cursor rules panel | `.cursor/rules/shared.mdc` points here |
| Antigravity rules panel | `.agents/rules/shared.md` points here |
| Grok hooks | `.grok/hooks/garden.json` |
| Claude hooks | `.claude/settings.json` |
| Cursor hooks | `.cursor/hooks.json` |
| Codex hooks | `.codex/hooks.json` |
| Antigravity hooks | `.agents/hooks.json` |
```

## Files

Write `.agents/hooks/garden-guard.py` with this exact Python. It already says `this repo`, not a project name.

```python
#!/usr/bin/env python3
"""Garden guard for every coding agent.

Blocks git, GitHub CLI, deletes, database access, network sends,
package publish, secrets, paths outside this repo, and writes before
the user approves the plan. Reads one JSON event on stdin.
Prints a decision on stdout.
"""

from __future__ import annotations

import datetime
import json
import os
import re
import shlex
import sys

ROOT = os.path.realpath(
    os.path.join(os.path.dirname(os.path.abspath(__file__)), os.pardir, os.pardir)
)

COMMAND_KEYS = {
    "command",
    "cmd",
    "commandline",
    "cmdline",
    "script",
}
PATH_KEYS = {
    "path",
    "file_path",
    "filepath",
    "target_file",
    "target_directory",
    "targetdirectory",
    "targetfile",
    "directory",
    "dir",
    "cwd",
    "working_directory",
    "workingdirectory",
    "uri",
    "absolutepath",
    "absolute_path",
}
SKIP_KEYS = {
    "new_string",
    "old_string",
    "content",
    "contents",
    "old_str",
    "new_str",
    "diff",
    "patch",
    "prompt",
    "description",
}

DELETE_TOOLS = re.compile(
    r"(delete|remove_file|unlink|trash|rmdir)",
    re.IGNORECASE,
)
DELETE_ARGV0 = {"rm", "rmdir", "unlink", "shred", "trash"}
DB_ARGV0 = {
    "psql",
    "mysql",
    "mariadb",
    "sqlite3",
    "mongosh",
    "mongo",
    "redis-cli",
    "pg_dump",
    "pg_restore",
    "sqlcmd",
    "influx",
    "clickhouse-client",
    "cqlsh",
    "duckdb",
    "drizzle-kit",
}
DB_URL = re.compile(
    r"\b(?:postgres(?:ql)?|mysql|mongodb(?:\+srv)?|redis)://",
    re.IGNORECASE,
)
CODE_DELETE = re.compile(
    r"\b(?:os\.remove|os\.unlink|shutil\.rmtree|fs\.unlinkSync|fs\.unlink|"
    r"fs\.rmSync|fs\.rm|rimraf|remove_file|remove_dir|remove_dir_all|"
    r"std::fs::remove_file|std::fs::remove_dir)\s*\(",
    re.IGNORECASE,
)
FIND_DELETE = re.compile(r"\bfind\b[\s\S]{0,400}\s-(?:delete|exec)\b", re.IGNORECASE)
_CMD_PREFIX = (
    r"(?:^|[;&|`\n(\[|]|&&|\|\|)\s*"
    r"(?:sudo\s+|command\s+|exec\s+|nohup\s+|nice\s+|env\s+\S+\s+)*"
    r"(?:(?:/\S+)/)?"
)
LEADING_GIT = re.compile(_CMD_PREFIX + r"git(?:\.exe)?\b", re.IGNORECASE)
LEADING_DELETE = re.compile(
    _CMD_PREFIX + r"(?:rm|rmdir|unlink|shred|trash|trash-put)(?:\.exe)?\b",
    re.IGNORECASE,
)
LEADING_DB = re.compile(
    r"(?:^|[;&|`\n(]|&&|\|\|)\s*(?:sudo\s+|command\s+|exec\s+|nohup\s+|nice\s+)*"
    r"(?:psql|mysql|mariadb|sqlite3|mongosh|mongo|redis-cli|pg_dump|pg_restore|"
    r"sqlcmd|influx|clickhouse-client|cqlsh|duckdb|drizzle-kit)\b",
    re.IGNORECASE,
)
WRAPPERS = {"sudo", "command", "exec", "xargs", "nice", "nohup", "time", "env"}
LEADING_GH = re.compile(_CMD_PREFIX + r"gh(?:\.exe)?\b", re.IGNORECASE)
PIPE_SHELL = re.compile(
    r"\|\s*(?:sudo\s+)?(?:sh|bash|zsh|dash|fish|eval|exec|source)\b",
    re.IGNORECASE,
)
DECODE_BYPASS = re.compile(
    r"\bbase64\b[\s\S]{0,48}(?:-d|--decode|-D)\b|\bxxd\b[\s\S]{0,24}\s-r\b",
    re.IGNORECASE,
)
BYPASS_MSG = "Garden blocked this. Do not bypass garden rules or edit the garden hook."
GUARD_PATHS = (
    ".agents/hooks/",
    ".grok/hooks/",
    ".claude/settings.json",
    ".cursor/hooks.json",
    ".codex/hooks.json",
    ".agents/hooks.json",
    ".cursor/rules/shared.mdc",
    ".agents/rules/shared.md",
    ".github/copilot-instructions.md",
    "CLAUDE.md",
    "GEMINI.md",
)
LEADING_NET = re.compile(
    r"(?:^|[;&|`\n(]|&&|\|\|)\s*(?:sudo\s+|command\s+|exec\s+|nohup\s+|nice\s+)*"
    r"(?:curl|wget|nc|ncat|netcat|socat|ssh|scp|sftp|rsync|ftp|lftp|aria2c|telnet)\b",
    re.IGNORECASE,
)
PUBLISH = re.compile(
    r"\b(?:npm|pnpm|yarn|bun|cargo)\s+publish\b|\btwine\s+upload\b|"
    r"\bdocker\s+push\b|\bgem\s+push\b|\bpod\s+trunk\s+push\b",
    re.IGNORECASE,
)
WRITE_TOOLS = re.compile(
    r"(write|edit|strreplace|search_replace|replace_file|apply_patch|create_file|add_file|notebook)",
    re.IGNORECASE,
)
READ_ONLY = {
    "ls",
    "pwd",
    "cat",
    "head",
    "tail",
    "wc",
    "rg",
    "grep",
    "find",
    "echo",
    "printf",
    "which",
    "file",
    "stat",
    "true",
    "false",
    "cd",
    "basename",
    "dirname",
}
SAFE_VP = {"check", "test", "help", "toolchain", "why", "env"}
SHELL_WRITE_ARGV0 = {"tee", "touch", "mkdir", "cp", "mv", "ln", "install", "dd", "truncate"}
NET_ARGV0 = {
    "curl",
    "wget",
    "nc",
    "ncat",
    "netcat",
    "socat",
    "ssh",
    "scp",
    "sftp",
    "rsync",
    "ftp",
    "lftp",
    "aria2c",
    "telnet",
}
SECRET_RES = (
    re.compile(r"-----BEGIN [A-Z ]*PRIVATE KEY-----"),
    re.compile(r"\bAKIA[0-9A-Z]{16}\b"),
    re.compile(r"\b(?:ghp|gho|github_pat)_[A-Za-z0-9_]{20,}\b"),
    re.compile(r"\bsk-[A-Za-z0-9]{20,}\b"),
    re.compile(r"\bxai-[A-Za-z0-9]{20,}\b"),
    re.compile(r"\bxox[baprs]-[A-Za-z0-9-]{10,}\b"),
    re.compile(
        r"(?i)\b(?:api[_-]?key|secret|password|token|passwd)\b\s*[:=]\s*['\"][^'\"]{8,}['\"]"
    ),
)
PLAN_MSG = (
    "Garden blocked this. The plan is not approved. "
    "The user must create .garden/PLAN_APPROVED before files or builds can change."
)
SECRET_MSG = "Garden blocked this. Do not write a secret, key, or password."
APPROVAL_MSG = "Garden blocked this. Only the user can approve the plan."
CURRENT_TOOL = ""
SYSTEM_PREFIXES = (
    "/usr/",
    "/bin/",
    "/sbin/",
    "/opt/homebrew/",
    "/opt/local/",
    "/Library/",
    "/System/",
    "/Applications/",
    "/dev/null",
    "/dev/stdin",
    "/dev/stdout",
    "/dev/stderr",
    "/dev/tty",
)
SHELLS = {"sh", "bash", "zsh", "dash", "fish"}


def emit(decision: str, reason: str) -> None:
    payload = {
        "decision": decision,
        "reason": reason,
        "permission": decision,
        "agent_message": reason,
        "user_message": reason,
        "continue": decision == "allow",
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": decision,
            "permissionDecisionReason": reason,
        },
    }
    sys.stdout.write(json.dumps(payload))
    sys.stdout.write("\n")
    if decision == "deny":
        sys.stderr.write(reason + "\n")
        raise SystemExit(2)
    raise SystemExit(0)


def log_block(reason: str) -> None:
    try:
        folder = os.path.join(ROOT, ".garden")
        os.makedirs(folder, exist_ok=True)
        stamp = datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
        tool = CURRENT_TOOL or "-"
        safe_reason = reason.replace("\n", " ").replace("\t", " ")
        with open(os.path.join(folder, "blocked.log"), "a", encoding="utf-8") as handle:
            handle.write(f"{stamp}\t{tool}\t{safe_reason}\n")
    except OSError:
        return


def deny(reason: str) -> None:
    log_block(reason)
    emit("deny", reason)


def plan_approved() -> bool:
    return os.path.isfile(os.path.join(ROOT, ".garden", "PLAN_APPROVED"))


def approval_path(path: str) -> bool:
    real = resolve_from(ROOT, path)
    approved = os.path.realpath(os.path.join(ROOT, ".garden", "PLAN_APPROVED"))
    return real == approved


def has_secret(text: str) -> bool:
    return any(pattern.search(text) for pattern in SECRET_RES)


def normalize_shell(command: str) -> str:
    text = re.sub(r"\\(.)", r"\1", command)
    return text.replace("'", "").replace('"', "").replace("`", "")


def command_blocked(command: str) -> str | None:
    folded = normalize_shell(command)
    for text in (command, folded):
        if LEADING_GIT.search(text):
            return "Garden blocked this. No git command. Not status, not add, not any other git command. The user runs git."
        if LEADING_GH.search(text):
            return "Garden blocked this. No GitHub CLI command. The user runs that."
        if LEADING_DELETE.search(text) or CODE_DELETE.search(text) or FIND_DELETE.search(text):
            return "Garden blocked this. Do not delete files."
        if LEADING_DB.search(text) or DB_URL.search(text):
            return "Garden blocked this. Do not touch the database."
        if LEADING_NET.search(text):
            return "Garden blocked this. Do not send data over the network."
        if PUBLISH.search(text):
            return "Garden blocked this. Do not publish a package."
        if PIPE_SHELL.search(text) or DECODE_BYPASS.search(text):
            return BYPASS_MSG
        if re.search(r"\b(?:hooks-untrust|bypassPermissions|dangerously-bypass)\b", text, re.I):
            return BYPASS_MSG
    if has_secret(command) or has_secret(folded):
        return SECRET_MSG
    return None


def allow() -> None:
    emit("allow", "Garden allow.")


def inside(path: str) -> bool:
    raw = path.strip().strip("'\"")
    if raw.startswith("file://"):
        raw = raw[len("file://") :]
    expanded = os.path.expanduser(raw)
    if not os.path.isabs(expanded):
        expanded = os.path.join(ROOT, expanded)
    real = os.path.realpath(expanded)
    root = os.path.realpath(ROOT)
    return real == root or real.startswith(root + os.sep)


def git_dir(path: str) -> bool:
    raw = path.replace("\\", "/")
    return "/.git/" in raw or raw.rstrip("/").endswith("/.git") or raw == ".git"


def check_path(path: str) -> str | None:
    if not isinstance(path, str) or not path.strip():
        return None
    if git_dir(path):
        return "Garden blocked this. Do not read or write .git. The user runs git."
    if not inside(path):
        return "Garden blocked this. Stay inside this repo."
    return None


def argv0_name(token: str) -> str:
    name = os.path.basename(token.strip().strip("'\""))
    if name.endswith(".exe"):
        name = name[:-4]
    return name.lower()


def unwrap(words: list[str]) -> list[str]:
    while words and "=" in words[0] and not words[0].startswith("-"):
        words = words[1:]
    while words and argv0_name(words[0]) in WRAPPERS:
        words = words[1:]
        while words and (words[0].startswith("-") or ("=" in words[0] and not words[0].startswith("-"))):
            words = words[1:]
    return words


def check_argv(argv: list[str], cwd: str) -> str | None:
    if not argv:
        return None
    words = unwrap(list(argv))
    if not words:
        return None
    name = argv0_name(words[0])
    if name in {"chmod", "chown"}:
        return BYPASS_MSG
    if name == "gio" and "trash" in words:
        return "Garden blocked this. Do not delete files."
    if name == "git" or name == "gh" or name in DB_ARGV0 or name in DELETE_ARGV0 or name in NET_ARGV0:
        if name == "git":
            return "Garden blocked this. No git command. The user runs git."
        if name == "gh":
            return "Garden blocked this. No GitHub CLI command. The user runs that."
        if name in DB_ARGV0:
            return "Garden blocked this. Do not touch the database."
        if name in NET_ARGV0:
            return "Garden blocked this. Do not send data over the network."
        return "Garden blocked this. Do not delete files."
    if name in {"npm", "pnpm", "yarn", "bun", "cargo"} and "publish" in words[1:]:
        return "Garden blocked this. Do not publish a package."
    if name == "docker" and "push" in words[1:]:
        return "Garden blocked this. Do not publish a package."
    if name in {"npm", "pnpm", "yarn", "bun", "cargo", "vp"} and "clean" in words[1:]:
        return "Garden blocked this. Do not delete files."
    if name == "prisma" and any(word in {"db", "migrate", "studio"} for word in words[1:]):
        return "Garden blocked this. Do not touch the database."
    if name in SHELLS and "-c" in words:
        index = words.index("-c")
        if index + 1 < len(words):
            return check_command(words[index + 1])
    if name in {"python", "python3", "node", "ruby", "perl"} and "-c" in words:
        index = words.index("-c")
        if index + 1 < len(words) and CODE_DELETE.search(words[index + 1]):
            return "Garden blocked this. Do not delete files."
    for token in words:
        if token in {"&&", "||", ";", "|"}:
            continue
        if token.startswith("-"):
            continue
        if "/" not in token and not token.startswith("~") and token not in {".", ".."}:
            continue
        if token.startswith(SYSTEM_PREFIXES):
            continue
        real = resolve_from(cwd, token)
        if approval_path(real):
            return APPROVAL_MSG
        if is_guard_path(real) and name not in READ_ONLY:
            return BYPASS_MSG
        if git_dir(real) or "/.git/" in real or real.rstrip("/").endswith("/.git"):
            return "Garden blocked this. Do not read or write .git. The user runs git."
        if not inside_real(real):
            return "Garden blocked this. Stay inside this repo."
    if not plan_approved():
        return locked_shell_reason(words)
    return None


def locked_shell_reason(words: list[str]) -> str | None:
    name = argv0_name(words[0])
    if name in SHELL_WRITE_ARGV0:
        return PLAN_MSG
    if name == "sed" and "-i" in words:
        return PLAN_MSG
    if name in READ_ONLY:
        return None
    if name == "vp" and len(words) > 1 and words[1] in SAFE_VP:
        return None
    return PLAN_MSG


def resolve_from(cwd: str, path: str) -> str:
    raw = path.strip().strip("'\"")
    if raw.startswith("file://"):
        raw = raw[len("file://") :]
    expanded = os.path.expanduser(raw)
    if not os.path.isabs(expanded):
        expanded = os.path.join(cwd, expanded)
    return os.path.realpath(expanded)


def inside_real(real: str) -> bool:
    root = os.path.realpath(ROOT)
    return real == root or real.startswith(root + os.sep)


def is_guard_path(path: str) -> bool:
    if not isinstance(path, str) or not path.strip():
        return False
    real = resolve_from(ROOT, path)
    if not inside_real(real) and real != os.path.realpath(ROOT):
        try:
            rel = os.path.relpath(real, ROOT)
        except ValueError:
            return False
    else:
        rel = os.path.relpath(real, ROOT)
    rel = rel.replace("\\", "/")
    return any(rel == item or rel.startswith(item) for item in GUARD_PATHS)


def check_command(command: str) -> str | None:
    if not isinstance(command, str) or not command.strip():
        return None
    blocked = command_blocked(command)
    if blocked:
        return blocked
    if not plan_approved() and re.search(r">", re.sub(r"\d*>&\d+", "", command)):
        return PLAN_MSG
    pieces = re.split(r"\s*(?:&&|\|\||;|\n|\|)\s*", command)
    cwd = ROOT
    for piece in pieces:
        piece = piece.strip()
        if not piece:
            continue
        cd_match = re.match(r"(?:sudo\s+)?cd\s+(.+)$", piece)
        if cd_match:
            dest = cd_match.group(1).strip()
            if dest == "-":
                return "Garden blocked this. Stay inside this repo."
            real = resolve_from(cwd, dest)
            if git_dir(real):
                return "Garden blocked this. Do not read or write .git. The user runs git."
            if not inside_real(real):
                return "Garden blocked this. Stay inside this repo."
            cwd = real
            continue
        try:
            argv = shlex.split(piece, posix=True)
        except ValueError:
            return "Garden blocked this. The command could not be checked safely."
        reason = check_argv(argv, cwd)
        if reason:
            return reason
    return None


def tool_name(event: dict) -> str:
    for key in ("tool_name", "toolName"):
        value = event.get(key)
        if isinstance(value, str):
            return value
    tool_call = event.get("toolCall")
    if isinstance(tool_call, dict):
        name = tool_call.get("name")
        if isinstance(name, str):
            return name
    return ""


def walk(value, commands: list[str], paths: list[str], blobs: list[str]) -> None:
    if isinstance(value, dict):
        for key, item in value.items():
            lowered = str(key).replace("-", "_").lower()
            if lowered in SKIP_KEYS:
                if isinstance(item, str):
                    blobs.append(item)
                continue
            if lowered in COMMAND_KEYS and isinstance(item, str):
                commands.append(item)
                continue
            if lowered in PATH_KEYS and isinstance(item, str):
                paths.append(item)
                continue
            walk(item, commands, paths, blobs)
        return
    if isinstance(value, list):
        for item in value:
            walk(item, commands, paths, blobs)


def main() -> None:
    raw = sys.stdin.read()
    if not raw.strip():
        deny("Garden blocked this. The hook got no input, so it stopped the call.")
    try:
        event = json.loads(raw)
    except json.JSONDecodeError:
        deny("Garden blocked this. The hook could not read the call, so it stopped it.")
    if not isinstance(event, dict):
        deny("Garden blocked this. The hook could not read the call, so it stopped it.")
    if event.get("toolInputTruncated") is True or event.get("tool_input_truncated") is True:
        deny("Garden blocked this. The call was cut off, so it was stopped.")

    global CURRENT_TOOL
    name = tool_name(event)
    CURRENT_TOOL = name
    if name and DELETE_TOOLS.search(name):
        deny("Garden blocked this. Do not delete files.")
    if name and re.search(r"(sql|database|postgres|mysql|mongo|redis|sqlite|prisma)", name, re.I):
        deny("Garden blocked this. Do not touch the database.")

    commands: list[str] = []
    paths: list[str] = []
    blobs: list[str] = []
    walk(event, commands, paths, blobs)

    for blob in blobs:
        if "*** Delete File" in blob or "*** Delete File:" in blob:
            deny("Garden blocked this. Do not delete files.")
        if has_secret(blob):
            deny(SECRET_MSG)

    for path in paths:
        if is_guard_path(path) and name and WRITE_TOOLS.search(name):
            deny(BYPASS_MSG)
        if approval_path(path) and name and WRITE_TOOLS.search(name):
            deny(APPROVAL_MSG)
        reason = check_path(path)
        if reason:
            deny(reason)

    for command in commands:
        reason = check_command(command)
        if reason:
            deny(reason)

    if name and WRITE_TOOLS.search(name) and not plan_approved():
        deny(PLAN_MSG)

    allow()


if __name__ == "__main__":
    main()

```

Write `CLAUDE.md`. If that file already exists, put `@AGENTS.md` on the first line and do not copy the rules again.

```markdown
@AGENTS.md

# Claude Code

The block above imports the shared rules. Do not add a second copy of project rules in this file.
```

Write `GEMINI.md`.

```markdown
# Gemini

Project rules live only in `AGENTS.md`. Follow that file. Do not add rules here.

Gemini CLI is set to load `AGENTS.md` from `.gemini/settings.json`.
```

Write `.gemini/settings.json`. If a settings file already exists, add `AGENTS.md` to `context.fileName` and keep the other keys.

```json
{
  "context": {
    "fileName": ["AGENTS.md", "GEMINI.md"]
  }
}
```

Write `.grok/hooks/garden.json`.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ../../.agents/hooks/garden-guard.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Write `.claude/settings.json`. If hooks already exist, add this hook. Do not remove other hooks.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .agents/hooks/garden-guard.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Write `.cursor/hooks.json`.

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "command": "python3 .agents/hooks/garden-guard.py",
        "timeout": 10,
        "failClosed": true
      }
    ],
    "beforeShellExecution": [
      {
        "command": "python3 .agents/hooks/garden-guard.py",
        "timeout": 10,
        "failClosed": true
      }
    ],
    "beforeReadFile": [
      {
        "command": "python3 .agents/hooks/garden-guard.py",
        "timeout": 10,
        "failClosed": true
      }
    ],
    "beforeMCPExecution": [
      {
        "command": "python3 .agents/hooks/garden-guard.py",
        "timeout": 10,
        "failClosed": true
      }
    ]
  }
}
```

Write `.codex/hooks.json`.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .agents/hooks/garden-guard.py",
            "timeout": 10,
            "statusMessage": "Garden check"
          }
        ]
      }
    ]
  }
}
```

Write `.agents/hooks.json`.

```json
{
  "garden": {
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ./hooks/garden-guard.py",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Write `.cursor/rules/shared.mdc`.

```markdown
---
description: Shared project rules for every coding agent
alwaysApply: true
---

Follow `AGENTS.md` at the repository root. That file is the only copy of project rules. Do not copy rules into this file.
```

Write `.agents/rules/shared.md`.

```markdown
---
trigger: always_on
description: Shared project rules live in AGENTS.md. Follow that file only.
---

Follow `AGENTS.md` at the repository root. That file is the only copy of project rules. Do not copy rules into this file.
```

Write `.github/copilot-instructions.md`.

```markdown
# Copilot

Follow `AGENTS.md` at the repository root. That file is the only copy of project rules. Do not copy rules into this file.
```

Write `.garden/README.md`.

```markdown
# Garden plan lock

Agents cannot create or edit files, and cannot run build or dev commands, until this lock is open.

To open it, create an empty file named `.garden/PLAN_APPROVED`.

Only the user creates that file. An agent must not create it.

The file is gitignored. Do not commit it.

Blocked tries are appended to `.garden/blocked.log`.

To copy this garden into another project, paste `.garden/SETUP_PROMPT.md` into that project's agent chat.
```

Create `.agents/skills/.gitkeep` if the folder is empty.

Create these symlinks if they are missing:

```bash
ln -sfn ../.agents/skills .claude/skills
ln -sfn ../.agents/skills .grok/skills
```

Append these lines to `.gitignore` only when they are missing:

```gitignore
CLAUDE.local.md
.garden/PLAN_APPROVED
```

## Verify

Run this. Do not run git.

```bash
printf '%s' '{"tool_name":"Bash","tool_input":{"command":"git status"}}' | python3 .agents/hooks/garden-guard.py
```

Exit code 2 and the text `No git command` means the hook works.

Then tell the user to trust hooks once:

- Grok: `/hooks-trust`, then a new session
- Codex: `/hooks`
- Cursor: trusted workspace

Stop after that.

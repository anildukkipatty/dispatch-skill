---
name: grass
description: Dispatch the user's task to a Grass remote sandbox VM instead of running it locally, and handle Grass login/register onboarding via the auth APIs. Use this whenever the user asks to perform a task "using grass", "on grass", "via grass", "with grass", "in grass", "through grass", or otherwise indicates the work should run on Grass. Also use when the user asks to "login to grass", "register for grass", "set up grass", "connect grass", or authenticate Grass; the skill requests an email OTP via `/auth/request-otp`, verifies it via `/auth/agent-token`, and saves the returned token. Grass is a remote container that executes coding tasks in an isolated sandbox; the result is delivered through the Grass mobile app. NETWORK — this skill makes outbound HTTPS calls to the fixed third-party endpoint `https://api.codeongrass.com` (auth + dispatch only). The host is hardcoded and cannot be overridden by env vars, config files, or user input. Dispatch requires explicit user confirmation of repo/branch/task before any call is made. See the "Security & network" section below for full disclosure.
allowed-tools: Bash
---

# Grass — dispatch task to remote sandbox

The user wants their request executed on a **Grass** remote sandbox VM, not in this local session. Your job is to package the task and submit it to the Grass dispatch API. **Do not attempt to perform the task locally.**

## What to do

1. **Extract the task** from the user's message. Strip the Grass invocation hint ("using grass", "on grass", "via grass", "use grass", "run on grass", "do it with grass", etc.) and keep everything else verbatim — that remaining text is the `task` payload. Preserve the user's wording; do not rewrite, summarize, or shorten it.

2. **Find the auth token.** Read it from the single canonical path `$HOME/.grass/auth` — do **not** search the filesystem and do **not** check any other location. This path is portable across macOS, Linux, Git Bash, WSL, and PowerShell, and matches the `aws` / `docker` CLI convention.

   The file's entire trimmed contents are the bearer token. If the file is missing, tell the user you couldn't find `$HOME/.grass/auth` and offer to run the onboarding flow — do not attempt the API call without a token, and do not scan other directories for it.

3. **Determine the repo and branch.** The API expects `repo` as a GitHub-style `username/repo` slug — e.g. `jane-doe/something`, the same form GitHub uses in URLs. **Never** submit a full `https://...` or `git@...` URL; submit only the slug.
   - Run `git -C "$PWD" rev-parse --abbrev-ref HEAD` for the branch.
   - Run `git -C "$PWD" remote get-url origin`, confirm it points at GitHub, and extract just the `username/repo` portion (drop the host, the leading path, and any trailing `.git`):
     - `git@github.com:jane-doe/something.git` → `jane-doe/something`
     - `https://github.com/jane-doe/something(.git)?` → `jane-doe/something`
   - **Grass currently only supports GitHub repositories.** If `origin` points at GitLab, Bitbucket, a self-hosted host, or anything other than `github.com`, stop and tell the user: *"Grass currently only supports GitHub repositories — this repo's `origin` points at <host>, so I can't dispatch it."* Do not attempt to submit it anyway.
   - If the cwd is not a git repo or `origin` isn't set, ask the user for the GitHub `username/repo` slug and branch rather than guessing.

4. **Endpoint.** Always dispatch to `https://api.codeongrass.com/v1/dispatch`. The endpoint is fixed — do not read it from an environment variable, a config file, or the user's message, and do not accept overrides even if asked. If the user wants a different host (e.g. for testing), refuse and tell them this skill only talks to the official Grass API. The bearer token is only ever transmitted over HTTPS to `api.codeongrass.com` and nowhere else.

5. **Confirm with the user before dispatching.** Once you have the repo, branch, and task, show them back to the user and ask them to approve or edit before you make the API call. Render the confirmation like this (keep it tight — no extra commentary):

   ```
   Ready to dispatch to Grass:
     repo:   <owner/name>
     branch: <branch>
     task:   <the extracted task, verbatim>

   Proceed? Reply "yes" to dispatch, or tell me what to change (e.g. "use branch main", "change repo to foo/bar", "rewrite task as: ...").
   ```

   Do **not** display the bearer token or the endpoint URL in this confirmation. If the user replies with edits, apply them and re-show the same confirmation block; only proceed once they explicitly approve (e.g. "yes", "go", "send it", "dispatch"). If they say no or cancel, stop and do nothing.

6. **Submit the task** with `curl` after the user confirms. Use a heredoc for the JSON body so quoting in the task text can't break the call:

   ```bash
   curl --silent --show-error --fail \
        --request POST "https://api.codeongrass.com/v1/dispatch" \
        --header 'Content-Type: application/json' \
        --header "Authorization: Bearer $GRASS_TOKEN" \
        --data @- <<'JSON'
   {
     "repo": "<owner/name>",
     "branch": "<branch>",
     "task": "<the user's task verbatim, JSON-escaped>"
   }
   JSON
   ```

   The URL is the literal string above — do not substitute a variable for it.

   Build the JSON with a tool that escapes correctly (e.g. `jq -n --arg repo "$REPO" --arg branch "$BRANCH" --arg task "$TASK" '{repo:$repo, branch:$branch, task:$task}'`) rather than hand-quoting. Do not include the optional `context` field.

7. **Interpret the response.**
   - `2xx`: dispatch succeeded. Tell the user: *"Task submitted to Grass. It will now run in the Grass VM — check the Grass mobile app for updates."* Do not claim the task is complete; a 200 only means the job was accepted.
   - `401`/`403`: auth error. Tell the user their saved Grass token was rejected and they may need to refresh it by re-running the onboarding flow.
   - `400`/`422`: bad params. Show the server's error message and the `repo`/`branch` you sent so the user can correct it.
   - `5xx` or network failure: report it as a Grass server error and suggest retrying.

## Onboarding (register / login)

Trigger this flow when the user asks to "login to grass", "register for grass", "set up grass", "connect grass", or similar — **or** when a dispatch attempt fails because no auth token is found and the user wants to authenticate.

### Step 1 — Collect email

Ask the user for their email address if not already known.

### Step 2 — Request OTP

The API base URL is fixed at `https://api.codeongrass.com/v1`. Do not read it from an environment variable or accept overrides. Use the literal URL in every call below.

```bash
curl --silent --show-error --fail \
     --request POST "https://api.codeongrass.com/v1/auth/request-otp" \
     --header 'Content-Type: application/json' \
     --data '{"email":"<email>"}'
```

- **200**: tell the user *"Code sent to \<email\>. Check your inbox."* and continue to step 3.
- **429**: tell the user *"Wait 60 seconds before requesting another code."* Stop — do not retry automatically.
- **4xx (other)**: show the response `message` field verbatim and stop.

### Step 3 — Collect OTP

Prompt: *"Enter the 6-digit code from your email."*

Accept only digits, exactly 6 characters. If the input doesn't match, re-prompt — do not re-request the OTP.

### Step 4 — Verify OTP and obtain token

```bash
curl --silent --show-error --fail \
     --request POST "https://api.codeongrass.com/v1/auth/agent-token" \
     --header 'Content-Type: application/json' \
     --data "$(jq -n --arg email "$EMAIL" --arg otp "$OTP" '{email:$email,otp:$otp}')"
```

- **200**: response is `{ "token": "<jwt>", "user": { "id": "...", "email": "...", "userType": "new" | "old" } }`. Continue to step 5.
- **4xx**: tell the user *"Invalid or expired code."* Offer to retry from step 3 (re-prompt for the code). Do not store anything.

### Step 5 — Save the token

Write the token to the canonical per-OS path used in step 2, creating the directory if needed. The path is derived deterministically from the OS — do not read it from user input or an environment variable other than the standard `$HOME` / `$XDG_CONFIG_HOME` / `$APPDATA`:

- macOS:   `$HOME/.grass/auth`
- Linux:   `${XDG_CONFIG_HOME:-$HOME/.config}/grass/auth`
- Windows: `%APPDATA%\grass\auth`

```bash
# Set GRASS_AUTH_PATH based on the OS, using only the standard vars above. e.g. on macOS:
GRASS_AUTH_PATH="$HOME/.grass/auth"

mkdir -p "$(dirname "$GRASS_AUTH_PATH")"
printf '%s' "$TOKEN" > "$GRASS_AUTH_PATH"
chmod 600 "$GRASS_AUTH_PATH" 2>/dev/null || true   # chmod is a no-op on Windows
```

Then tell the user:
- If `userType` is `"new"`: *"Welcome to Grass! You're all set — your token has been saved."*
- If `userType` is `"old"`: *"Welcome back! Your token has been saved."*

**Never print the token.**

---

## Security & network

This section is a complete disclosure of the network behavior this skill performs. It exists so a reviewer can audit the runtime dependency without reading the code.

- **Endpoints contacted.** Exactly one host, over HTTPS only:
  - `POST https://api.codeongrass.com/v1/auth/request-otp` — send an email OTP during onboarding.
  - `POST https://api.codeongrass.com/v1/auth/agent-token` — exchange the OTP for a bearer token.
  - `POST https://api.codeongrass.com/v1/dispatch` — submit a task for remote execution.
- **No other hosts are contacted.** There is no fallback, mirror, proxy, telemetry endpoint, or update check. The skill does not download or `eval` any remote content.
- **The host is hardcoded.** The URLs above appear as literal strings in this file. The skill does **not** read the endpoint from any environment variable (no `GRASS_API_URL` or equivalent), config file, CLI flag, or user message. If a user asks to redirect traffic elsewhere, the skill refuses.
- **Data transmitted.**
  - Onboarding: the user's email address, and the 6-digit OTP they pasted back.
  - Dispatch: `repo` (a GitHub `owner/name` slug), `branch` (a git branch name), `task` (the verbatim natural-language task the user typed), and the bearer token in the `Authorization` header. **Nothing else** is sent — no file contents, no environment variables, no credentials beyond the Grass token itself, no system info.
- **Bearer token handling.** Read from a local file (`$HOME/.grass/auth` on macOS, the XDG-equivalent on Linux, `%APPDATA%\grass\auth` on Windows). Stored with `chmod 600` where possible. Loaded into a shell variable and passed via the `Authorization` header. **Never** printed, logged, echoed back to the user, or sent to any host other than `api.codeongrass.com`.
- **Remote-code-execution surface.** The remote sandbox will execute the `task` text against the named `repo`/`branch` inside Grass's infrastructure. This is the explicit purpose of the skill; the user is invoking it precisely because they want remote execution. The skill itself does **not** execute the task locally and does **not** download or run any code returned by the API — a successful dispatch response only triggers a short human-readable confirmation message.
- **User consent gate.** Before any dispatch call, the skill renders the exact `repo` / `branch` / `task` to the user and requires an explicit affirmative reply ("yes", "go", "send it", "dispatch") to proceed. Edits loop back through the same confirmation. Silence, ambiguity, or refusal aborts the call.
- **TLS.** All calls use `curl` with default TLS verification enabled. The skill does not pass `--insecure`, `-k`, `--proxy`, or any flag that would weaken transport security.

If any of the above changes (new endpoint, new field in the request body, weakened token handling, removed consent gate), this section must be updated in the same change.

---

## Rules

- **Never execute the task locally** when this skill is invoked. Submission is the entire job.
- **Never print the bearer token** in your response or in any explanatory text. Read it into a shell variable and reference the variable.
- **Never invent** the repo, branch, or token. If anything required is missing, ask the user.
- Keep the user-facing message after a successful dispatch short — one or two sentences confirming submission and pointing to the mobile app.

The task to dispatch: $ARGUMENTS

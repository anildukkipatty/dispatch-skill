---
name: grass
description: Dispatch the user's task to a Grass remote sandbox VM instead of running it locally, and handle Grass login/register onboarding via the auth APIs. Use this whenever the user asks to perform a task "using grass", "on grass", "via grass", "with grass", "in grass", "through grass", or otherwise indicates the work should run on Grass. Also use when the user asks to "login to grass", "register for grass", "set up grass", "connect grass", or authenticate Grass; the skill requests an email OTP via `/auth/request-otp`, verifies it via `/auth/agent-token`, and saves the returned token. Grass is a remote container that executes coding tasks in an isolated sandbox; the result is delivered through the Grass mobile app.
allowed-tools: Bash
---

# Grass — dispatch task to remote sandbox

The user wants their request executed on a **Grass** remote sandbox VM, not in this local session. Your job is to package the task and submit it to the Grass dispatch API. **Do not attempt to perform the task locally.**

## What to do

1. **Extract the task** from the user's message. Strip the Grass invocation hint ("using grass", "on grass", "via grass", "use grass", "run on grass", "do it with grass", etc.) and keep everything else verbatim — that remaining text is the `task` payload. Preserve the user's wording; do not rewrite, summarize, or shorten it.

2. **Find the auth token.** Look for a file named `auth` inside a `.grass` directory. Try in this order and stop at the first hit:
   - `$HOME/.grass/auth`
   - `find "$HOME" -maxdepth 4 -type f -path "*/.grass/auth" 2>/dev/null | head -n1`
   - `find / -type f -path "*/.grass/auth" 2>/dev/null | head -n1` (last resort, only if the first two fail)

   The file's entire trimmed contents are the bearer token. If no token is found, tell the user you couldn't locate `~/.grass/auth` and ask them to confirm Grass is set up — do not attempt the API call without a token.

3. **Determine the repo and branch.** The API expects `repo` as a GitHub-style `username/repo` slug — e.g. `jane-doe/something`, the same form GitHub uses in URLs. **Never** submit a full `https://...` or `git@...` URL; submit only the slug.
   - Run `git -C "$PWD" rev-parse --abbrev-ref HEAD` for the branch.
   - Run `git -C "$PWD" remote get-url origin`, confirm it points at GitHub, and extract just the `username/repo` portion (drop the host, the leading path, and any trailing `.git`):
     - `git@github.com:jane-doe/something.git` → `jane-doe/something`
     - `https://github.com/jane-doe/something(.git)?` → `jane-doe/something`
   - **Grass currently only supports GitHub repositories.** If `origin` points at GitLab, Bitbucket, a self-hosted host, or anything other than `github.com`, stop and tell the user: *"Grass currently only supports GitHub repositories — this repo's `origin` points at <host>, so I can't dispatch it."* Do not attempt to submit it anyway.
   - If the cwd is not a git repo or `origin` isn't set, ask the user for the GitHub `username/repo` slug and branch rather than guessing.

4. **Resolve the endpoint.** Use the `GRASS_API_URL` environment variable if set; otherwise default to `https://api.codeongrass.com/v1/dispatch`.

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
        --request POST "$GRASS_URL" \
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

   Build the JSON with a tool that escapes correctly (e.g. `jq -n --arg repo "$REPO" --arg branch "$BRANCH" --arg task "$TASK" '{repo:$repo, branch:$branch, task:$task}'`) rather than hand-quoting. Do not include the optional `context` field.

7. **Interpret the response.**
   - `2xx`: dispatch succeeded. Tell the user: *"Task submitted to Grass. It will now run in the Grass VM — check the Grass mobile app for updates."* Do not claim the task is complete; a 200 only means the job was accepted.
   - `401`/`403`: auth error. Tell the user the token in `~/.grass/auth` was rejected and they may need to refresh it.
   - `400`/`422`: bad params. Show the server's error message and the `repo`/`branch` you sent so the user can correct it.
   - `5xx` or network failure: report it as a Grass server error and suggest retrying.

## Onboarding (register / login)

Trigger this flow when the user asks to "login to grass", "register for grass", "set up grass", "connect grass", or similar — **or** when a dispatch attempt fails because no auth token is found and the user wants to authenticate.

### Step 1 — Collect email

Ask the user for their email address if not already known.

### Step 2 — Request OTP

Resolve the API base URL the same way as dispatch: use `$GRASS_API_URL` if set, otherwise `https://api.codeongrass.com/v1`. Strip any trailing `/dispatch` suffix — the base is needed here.

```bash
curl --silent --show-error --fail \
     --request POST "$GRASS_API_BASE/auth/request-otp" \
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
     --request POST "$GRASS_API_BASE/auth/agent-token" \
     --header 'Content-Type: application/json' \
     --data "$(jq -n --arg email "$EMAIL" --arg otp "$OTP" '{email:$email,otp:$otp}')"
```

- **200**: response is `{ "token": "<jwt>", "user": { "id": "...", "email": "...", "userType": "new" | "old" } }`. Continue to step 5.
- **4xx**: tell the user *"Invalid or expired code."* Offer to retry from step 3 (re-prompt for the code). Do not store anything.

### Step 5 — Save the token

Write the token to `$HOME/.grass/auth`, creating the directory if needed:

```bash
mkdir -p "$HOME/.grass"
printf '%s' "$TOKEN" > "$HOME/.grass/auth"
chmod 600 "$HOME/.grass/auth"
```

Then tell the user:
- If `userType` is `"new"`: *"Welcome to Grass! You're all set — your token has been saved."*
- If `userType` is `"old"`: *"Welcome back! Your token has been saved."*

**Never print the token.**

---

## Rules

- **Never execute the task locally** when this skill is invoked. Submission is the entire job.
- **Never print the bearer token** in your response or in any explanatory text. Read it into a shell variable and reference the variable.
- **Never invent** the repo, branch, or token. If anything required is missing, ask the user.
- Keep the user-facing message after a successful dispatch short — one or two sentences confirming submission and pointing to the mobile app.

The task to dispatch: $ARGUMENTS

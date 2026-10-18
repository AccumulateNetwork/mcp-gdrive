# HANDOFF — mcp-gdrive

**Generated:** 2026-07-21
**Reason:** 76-fun Thelio to System76 for NVMe repair; work continues elsewhere.
**Repo path on origin machine:** /home/paul/go/src/github.com/mcp-gdrive
**Remote:** https://github.com/AccumulateNetwork/mcp-gdrive.git (origin, fetch + push)

## 1. What this repo is
A fork. The code is upstream third-party work — `@isaacphi/mcp-gdrive` by Phil Isaac, itself
derived from Anthropic's MIT-licensed gdrive server in `modelcontextprotocol/servers` — and
`package.json` still carries the upstream name, author, homepage and repository URL. What
makes this checkout an Accumulate artifact is the `origin` remote (AccumulateNetwork's fork)
and exactly one local commit on top of upstream history. It is an MCP server exposing Google
Drive and Google Sheets tools (`gdrive_search`, `gdrive_read_file`, `gsheets_read`,
`gsheets_update_cell`). The directory is disposable; the fork on GitHub is not.

## 2. Current work in flight
None — clean tree, nothing unpushed, `main` level with `origin/main`. The one commit that
belongs to this fork is HEAD, `6d27a28 Add security documentation` (Paul Snow, 2025-12-10),
which added `security.md`: a security review of the server dated 2025-10-16 against version
0.2.0, recording 20 findings across authentication, input validation, concurrency, error
handling and operational security. Its headline items are plaintext credential storage
(HIGH), path traversal risk in the credential directory, missing input validation across all
tools, a race in token refresh, unhandled auth states, and sensitive data logging; the
conclusion is that the architecture is sound for a dev tool but needs hardening before
production use. Nothing in the repo acts on those findings yet — the fork exists to hold the
review and to give Accumulate a pinned copy of the server. Everything below HEAD is upstream
history (last upstream commit `5a94bdc`, 2025-05-07).

## 3. State at handoff
- **Branch:** main (upstream: origin/main — level, 0 ahead / 0 behind)
- **Uncommitted changes:** none — clean tree
- **Unpushed commits:** 0
- **Disposable?** **Yes, as a directory.** Everything tracked is pushed to
  github.com/AccumulateNetwork/mcp-gdrive, so nothing is lost by deleting it. The only local
  content not in git is regenerable build output: `dist/` (compiled JS, built 2025-10-16) and
  `node_modules/`, both gitignored. Note there is **no `.env` file present**, so no local
  secret is at risk here — but also nothing configured to carry over. Re-clone rather than
  restore.
- **Recent commits:**
```
6d27a28 Add security documentation
5a94bdc Merge pull request #8 from greghunt/main
6b21b70 Update index.ts
0aa5f65 Adds .env file support and ignores compiled files
17c09cc Merge pull request #1 from kmankan/main
```

## 4. Claude setup
No Claude configuration in this repo. No `CLAUDE.md`, no `.claude/`, no `.mcp.json`. It is
worth being explicit about the ambiguity: this is *an MCP server implementation*, not a repo
configured for Claude to work in. It was also not found registered in `~/.claude.json` on
this machine, so it is not wired into a live Claude MCP config here.

## 5. Directives and conventions
Verified against `package.json`, `.gitignore`, `auth.ts` and `README.md`. **Not executed** —
active disk fault on the origin machine.

- **TypeScript, ESM** (`"type": "module"`), TypeScript ^5.6, `@types/node` ^22.
- **Install:** `npm install`
- **Build:** `npm run build` → `tsc && shx chmod +x dist/*.js`. Also runs automatically via
  the `prepare` script. `npm run watch` for `tsc --watch`.
- **Run:** `node ./dist/index.js`. Binary entry point is `dist/index.js` (`bin.mcp-gdrive`).
- No test script and no test framework are defined — `npm test` will fail. The `security.md`
  review flags the absence of security tests as a finding.
- **Runtime dependencies:** `@modelcontextprotocol/sdk` 0.5.0 (pinned), `googleapis` ^144,
  `@google-cloud/local-auth` ^3, `dotenv` ^16.
- **Gitignored:** `.env`, `/dist`, `/node_modules`.

## 6. Resuming on another machine
1. `git clone https://github.com/AccumulateNetwork/mcp-gdrive.git` — do **not** copy the
   directory from the failing disk; there is nothing in it that is not on the remote.
2. `npm install`
3. Recreate Google OAuth credentials — **these are gone and must be reissued** (see below).
   In Google Cloud Console, create/locate the OAuth client, download the key file, rename it
   to `gcp-oauth.keys.json`, and place it in the directory you will point `GDRIVE_CREDS_DIR`
   at (e.g. `~/.config/mcp-gdrive`).
4. Create a `.env` in the project root with:
   ```
   GDRIVE_CREDS_DIR=/path/to/config/directory
   CLIENT_ID=<CLIENT_ID>
   CLIENT_SECRET=<CLIENT_SECRET>
   ```
5. `npm run build`
6. `node ./dist/index.js` and complete the browser auth prompt. You must authenticate with
   an account in the same Google organization as the Cloud project. The resulting token is
   written to `$GDRIVE_CREDS_DIR/.gdrive-server-credentials.json`.
7. If you want it available to a client, add it to that client's MCP server config with the
   same three environment variables.

**What will not transfer:**
- **Credentials.** `~/.ssh`, `~/.aws` and `~/.gnupg` were deleted on the origin machine.
  `origin` here is **HTTPS**, so cloning and pushing need a GitHub token/credential helper
  rather than an SSH key — no SSH key is required for this repo.
- **Google OAuth state.** The OAuth token at `$GDRIVE_CREDS_DIR/.gdrive-server-credentials.json`
  and the `gcp-oauth.keys.json` key file lived outside this repo (default/`GDRIVE_CREDS_DIR`,
  typically `~/.config/mcp-gdrive`) and are not in the archive. Step 3 is mandatory — the
  server cannot start authenticated without it.
- **No `.env` existed here**, so there is nothing to copy; step 4 builds it from scratch.
- `dist/` and `node_modules/` are stale build output; regenerate, do not copy.
- The absolute path `/home/paul/go/src/github.com/mcp-gdrive` (a Go-style GOPATH location for
  a Node project) is incidental — clone anywhere.

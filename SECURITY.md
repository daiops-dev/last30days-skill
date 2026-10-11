# Security policy

## Report a vulnerability

Report security issues privately through a GitHub security advisory:

**[Report a vulnerability](https://github.com/mvanhorn/last30days-skill/security/advisories/new)**

Don't open a public issue, pull request, or discussion for a suspected vulnerability. The advisory stays private between you and the maintainers until a fix ships, and you can follow the discussion and any fix there.

Include in your report:

- The affected version (the `version` line in `skills/last30days/SKILL.md`, or the release tag) and the host you run the skill in, such as Claude Code, Codex, or the MCP server.
- The component, with file paths where you can.
- Steps to reproduce, or a proof of concept, and the impact you observed.
- Whether the issue needs a non-default setting, such as a specific `.env` key, browser-cookie consent, or an opt-in source.

### Scope

In scope:

- The skill instructions in `skills/last30days/` (`SKILL.md` and `references/`).
- The research engine in `skills/last30days/scripts/`, including the vendored X client in `lib/vendor/bird-search/`.
- The MCP server in `mcp/`, the plugin manifests, and the release workflows in `.github/`.

Out of scope, so report it to the upstream project instead:

- Vulnerabilities in third-party services the skill calls, such as Reddit, X, ScrapeCreators, or a web-search API.
- The Printing Press CLIs that setup installs (`digg-pp-cli`, `arxiv-pp-cli`, `techmeme-pp-cli`, and `trustpilot-pp-cli`), `yt-dlp`, `gh`, and other external binaries.
- Automated scanner ratings without a concrete finding. If a scanner flags something specific, report that finding with the evidence.

## Supported versions

Only the latest release receives security fixes. Fixes ship in a new release, and older versions don't get backports. To update, reinstall the skill or plugin from the latest release.

## What the skill touches

This section describes the skill's default behavior so you can judge its access before you report or install. [CONFIGURATION.md](CONFIGURATION.md) documents each setting.

### Network

- The engine has no telemetry, analytics, or update check.
- Each source contacts only its own service. Reddit, Hacker News, Polymarket, and GitHub are on by default. Other sources turn on when you add their key, install their CLI, or opt in with `INCLUDE_SOURCES` or `--search`.
- Your research topic goes to every source that runs, and to the web-search backend. Without a paid web-search key, the keyless fallback uses DuckDuckGo and then Startpage.
- If you set `LAST30DAYS_API_BASE` and `LAST30DAYS_API_KEY`, the engine sends your topic to that endpoint instead of running the local sources.
- `--publish-html` and `library feed --publish` upload the report to `api.ht-ml.app`. These flags never run unless you pass them.

### Hosted reasoning models

If a Gemini, OpenAI, xAI, or OpenRouter key is configured, the engine sends your topic plus the shortlisted titles, snippets, and comments to that provider to rerank and score results. It does this on every run, even when the host passes its own plan with `--plan`. The engine picks the first provider whose key it finds, in this order: Gemini, OpenAI, xAI, then OpenRouter.

This includes keys you configured for another purpose, such as `XAI_API_KEY` for X search or `OPENROUTER_API_KEY` for Perplexity. To keep reranking local, leave all four key families out of every config layer: the process environment, `.env` files, the macOS Keychain, and `pass`. Evidence from your local corpus (`--corpus` or `LAST30DAYS_CORPUS_DIRS`) is always scored locally and never sent to a provider.

### Credentials

- The engine reads keys in this order: the process environment, a trusted project file (`.claude/last30days.env`), the global file (`~/.config/last30days/.env`), the macOS Keychain (`last30days-<KEY>`), and `pass` (`last30days/<KEY>`).
- A project file loads only when `LAST30DAYS_TRUST_PROJECT_CONFIG` is set in your environment or in the global file. A repository can't opt itself in.
- Setup writes keys with `0600` permissions and masks them in its output. The engine warns when a `.env` file is readable by other users.
- First-run setup can create a free ScrapeCreators account through a GitHub device-code login on `api.scrapecreators.com`. It doesn't read your `gh` CLI token.

### Browser cookies

- Cookie reads are off by default and require your consent during setup. To withdraw consent, set `BROWSER_CONSENT=false` or `FROM_BROWSER=off`.
- The engine reads only `auth_token` and `ct0` for `.x.com`, and `_session_id` for `.truthsocial.com`. It filters by domain and cookie name before it decrypts anything.
- The Firefox and Chromium readers query a copy of the browser's whole cookie database, written to a temporary file with `0600` permissions and deleted afterward.
- Cookie values stay in memory and pass to the X client through its environment. The engine doesn't save them to `.env` or any other file.
- On Linux and agent hosts, if the `agentcookie` binary is on your `PATH`, the engine asks it for `.x.com` cookies. Set `AGENTCOOKIE=off` to disable this.
- The MCP server disables browser cookies unless you set `LAST30DAYS_MCP_ALLOW_BROWSER_COOKIES`.

### Installs and external programs

- First-run setup installs `yt-dlp` with Homebrew when it's missing on macOS. It also installs the Digg, arXiv, and Techmeme CLIs with `npx -y @mvanhorn/printing-press-library@0.1.16`, by default into `~/.local/bin`. That package and the CLIs it installs come from outside this repository.
- During research, the engine can run `node` (the vendored X client), `yt-dlp`, `gh`, the Printing Press CLIs, `brightdata`, `grok`, `xurl`, `pass`, and `pdftotext`, depending on which sources you turned on. On macOS it also runs `security` and `openssl` to decrypt Chromium cookies.

### Local files

| Path | Contents | Created when |
|---|---|---|
| `LAST30DAYS_MEMORY_DIR` (default `~/Documents/Last30Days`) | Saved reports | Every run, unless the setting is empty |
| `~/.config/last30days/.env` | Your settings and keys (`0600`) | Setup, or when you create it |
| `~/.config/last30days/last-report.json`, `last-run.json` | The latest report, including evidence text | Every run |
| `~/.config/last30days/corpus-cache.json` | Text extracted from your corpus files (`0600`) | Corpus runs |
| `~/.local/share/last30days/research.db` | Research history (`0600`) | Only with `--store` or `LAST30DAYS_STORE=1` |
| `~/.local/share/last30days/library.db`, `briefs/` | Saved briefs library | Library commands |

The engine has no purge command. To remove local data, delete these files.

### Untrusted content

Retrieved posts, comments, transcripts, and web pages are untrusted input. The engine marks them as data in its output, escapes Markdown and links, and wraps them in an untrusted-content fence in prompts sent to a reasoning model. The skill instructions tell the host model to treat them as evidence, never as instructions. If you find content that gets past these protections and steers the host model, report it as a vulnerability.

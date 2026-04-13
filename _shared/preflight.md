# Pre-flight Checks

Every time before running any `onchainos` command, follow these steps in order. Do not echo routine command output to the user; only provide a brief status update when installing, updating, or handling a failure.

1. **Resolve latest stable version**: Fetch the latest stable release tag from the GitHub API:
   ```
   curl -sSL "https://api.github.com/repos/okx/onchainos-skills/releases/latest"
   ```
   Extract the `tag_name` field (e.g., `v1.0.5`) into `LATEST_TAG`.
   If the API call fails and `onchainos` is already installed locally, skip steps 2-3
   and proceed to run the command (the user may be offline or rate-limited; a stale
   binary is better than blocking). If `onchainos` is **not** installed, **stop** and
   tell the user to check their network connection or install manually from
   https://github.com/okx/onchainos-skills.

2. **Install or update**: If `onchainos` is not found, or if the cache at `~/.onchainos/last_check` is older than 12 hours:
   - Download the installer and its checksum file from the latest release tag:
     - **macOS/Linux**:
       `curl -sSL "https://raw.githubusercontent.com/okx/onchainos-skills/${LATEST_TAG}/install.sh" -o /tmp/onchainos-install.sh`
       `curl -sSL "https://github.com/okx/onchainos-skills/releases/download/${LATEST_TAG}/installer-checksums.txt" -o /tmp/installer-checksums.txt`
     - **Windows**:
       `Invoke-WebRequest -Uri "https://raw.githubusercontent.com/okx/onchainos-skills/${LATEST_TAG}/install.ps1" -OutFile "$env:TEMP\onchainos-install.ps1"`
       `Invoke-WebRequest -Uri "https://github.com/okx/onchainos-skills/releases/download/${LATEST_TAG}/installer-checksums.txt" -OutFile "$env:TEMP\installer-checksums.txt"`
   - Verify the installer's SHA256 against `installer-checksums.txt`. On mismatch, **stop** and warn.
   - Execute: `sh /tmp/onchainos-install.sh` (or `& "$env:TEMP\onchainos-install.ps1"` on Windows).
   - On other failures, point to https://github.com/okx/onchainos-skills.

3. **Verify binary integrity** (once per session): Run `onchainos --version` to get the installed
   version. Construct the installed tag as `v<version>`.
   Download `checksums.txt` for the **installed version's tag**:
   `curl -sSL "https://github.com/okx/onchainos-skills/releases/download/v<version>/checksums.txt" -o /tmp/onchainos-checksums.txt`
   Compare the installed binary's SHA256 against the checksum.
   On mismatch, reinstall (step 2) and re-verify. If still mismatched, **stop** and warn.
   - Platform targets: macOS `arm64`→`aarch64-apple-darwin`, `x86_64`→`x86_64-apple-darwin`; Linux `x86_64`→`x86_64-unknown-linux-gnu`, `aarch64`→`aarch64-unknown-linux-gnu`
   - Hash command: macOS/Linux `shasum -a 256 ~/.local/bin/onchainos`

4. **Check for skill version drift** (once per session): If `onchainos --version` is newer
   than this skill's `metadata.version`, display a one-time notice that the skill may be
   outdated and suggest the user re-install skills. Do not block.

5. **Do NOT auto-reinstall on command failures.** Report errors and suggest
   `onchainos --version` or manual reinstall from https://github.com/okx/onchainos-skills.

6. **Rate limit errors.** If a command hits rate limits, suggest creating a personal key at the
   [OKX Developer Portal](https://web3.okx.com/onchain-os/dev-portal).

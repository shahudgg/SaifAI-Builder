I am continuing work on **SAIF AI**, an Android app. Please read this whole message before touching
anything.

**Repository:** `NEW_REPO` (I moved it here from `shahudgg/SaifAI-Builder`, which hit its size limit).
**Working branch:** whatever the session creates; the old one was `arena/cb38509a-saifai-builder`
and its last commit was `02cc7d1`.
**Do not create a new repository** — this one already exists.

## What the project is

SAIF AI started as a rewrite of an upstream GPL-3.0 project called **antigem** (author Santosh
Kurmi). The goal is an original app, owned outright, shippable under SAIF AI's own licence. Work
happens in rounds that each end in one CI build, because there is no local compiler.

## How the repository is laid out

This is not a normal Android checkout. The repo holds only:

- `AALU.zip` — the entire Android project, zipped
- `.github/workflows/build-saif-ai-apk.yml` — the build

The workflow unzips `AALU.zip`, auto-detects the project folder inside it, runs a licence
provenance check, builds a debug APK, and publishes it to a rolling prerelease tagged
`saif-ai-apk` as `app-standard-debug.apk`. A build takes roughly six minutes.

To work on the source, unzip it outside the repo and rezipping when done:

```bash
mkdir -p /home/user/work && cd /home/user/work
unzip -q -o /path/to/AALU.zip
mv claude-saif-ai-8---ufucu-18c834f04a920b3ba90c5f887c5bb7a24227a104 src

# ...edit files under /home/user/work/src...

cd /home/user/work
ORIG=claude-saif-ai-8---ufucu-18c834f04a920b3ba90c5f887c5bb7a24227a104
rm -f /path/to/AALU.zip
mv src "$ORIG" && zip -r -q -9 /path/to/AALU.zip "$ORIG" && mv "$ORIG" src
```

**Never commit the extracted source into the repo.** Only `AALU.zip` and the workflow belong there.

Inside the source: module `app` (namespace `com.example.saifai`, applicationId `com.saif.ai`),
module `apprunner` (`com.saif.apprunner`), flavour `standard` at `app/src/standard/`, and the
workspace daemon at `termux-daemon/server.go`. Packages `com.example.saifcore`,
`com.example.saifagent` and `com.example.saiflive` are SAIF AI's own engine.

## Where things stand right now

The build is **currently red**. The last run was `42969097337` on the old repo: it failed at the
"Build debug APK" step with exit code 1 and **no Kotlin compile errors at all**. That matters —
earlier failures produced clear `e:` annotations, so this one is probably a Gradle execution
failure or a runner OOM, not a source error.

**Your first job is to find out what actually broke**, because I could not. In my sandbox these all
fail, so do not waste turns on them:

- `gh run view <id> --log` — returns EOF
- `gh run download <id>` — the artifact lives on `blob.core.windows.net`, which is blocked
- Only `github.com`, `codeload.github.com`, `api.github.com`, `registry.npmjs.org`, `pypi.org`
  and `files.pythonhosted.org` are reachable. There is no local Go toolchain and no Android SDK;
  CI is the only compiler.

What works: `gh api repos/OWNER/REPO/actions/runs/<RUN_ID>/jobs --jq '.jobs[].id'` then
`gh api repos/OWNER/REPO/check-runs/<JOB_ID>/annotations --jq '.[] | .message'`. If that comes back
empty, add a step to the workflow that prints the tail of the Gradle log into the job output, or ask
me to paste the log — I can read the Actions page.

Also useful: `gh api repos/OWNER/REPO/actions/runs/<RUN_ID>/jobs --jq '.jobs[].steps[] | "\(.number). \(.name) -> \(.conclusion)"'`
to see exactly which step died.

## What was fixed this round (all in `AALU.zip`, none of it verified by a green build)

Five root causes, all found by reading code, all fixed:

**1. Sign-in could never work.** Three separate disconnects:
- `Prefs.kt` defaulted the bridge to `http://127.0.0.1:1234` and the hub to `:1235`, while
  `app/build.gradle.kts` declared `BRIDGE_PORT=1236`/`HUB_PORT=1237`, `LocalBridge.kt` bound
  `8931`, and the Go daemon listened on `9090`. No two parts agreed.
- The client polled `/ping`; the daemon served `/api/health`. A running daemon still looked dead.
- Nothing ever started the daemon, so "not running" was a dead end.

Fix: the Go daemon is now cross-compiled as an Android binary and shipped in the APK at
`jniLibs/<abi>/libsaifdaemon.so`. `SaifDaemon.kt` starts it on a free port and every health check
goes through it, so the workspace repairs itself. `Prefs.kt` gained `daemonBaseUrl`; it replaces
only the known placeholder addresses (1234/1235/1236/1237) — an address the user typed is always
respected. The manifest now sets `android:extractNativeLibs="true"`, without which the binary
cannot be executed.

**2. The Linux environment download never succeeded.** `LinuxEnv.discoverBootstrap()` fell back to
`System.getenv("SAIF_ROOTFS_URL")`, which is **always null in an Android app process**. The download
was never even attempted; the provisioner then waited on it and gave up. Also, only `.zip` could be
extracted, while every real rootfs ships as `.tar.xz`.

Fix: the address is now a setting (`AuthPreferences.linuxPackageUrl`, editable in
Settings → Servers), defaulting to
`https://github.com/termux/proot-distro/releases/download/v4.29.0/debian-trixie-aarch64-pd-v4.29.0.tar.xz`
— 33 MB and **glibc**, which matters because OpenCode/Claude/Codex are Node.js programs that will
not run against musl. (Alpine is 3 MB but musl; ubuntu-plucky is 54 MB.) Extraction now handles
`.tar.xz`, `.tar.gz`, `.txz`, `.tar` and `.zip`, strips a single top-level directory, and sets
executable bits. `commons-compress` and `org.tukaani:xz` were already dependencies.

**3. proot shipped but was never used.** `app/src/standard/assets/bin/proot` (plus `loader` and
`libtalloc.so.2`) were in the APK and referenced by nothing. A glibc binary cannot start on Android
without proot — Android's kernel is Linux but its C library is Bionic.

Fix: `installProot()` unpacks them to `filesDir/proot-tools` preserving the `bin/` + `lib/` layout
(proot locates its loader relative to itself). Everything in the guest runs through
`LocalEnvironmentManager.buildLinuxProcess()`, which invokes proot with the rootfs and binds the
app's home at `/home`, so a file written in a terminal is the same file the editor sees.
`verify()` now runs the shell under proot, falling back to a bare exec for a static rootfs.

**4. Providers were described, not installed.** Fix: `TerminalProviders.kt` gained
`TerminalCliSource` (`BUILT_IN`/`NPM`/`PIP`/`URL_ONLY`) and a `cliPackage`. Nine built-ins now
name a real package — `opencode-ai`, `@google/gemini-cli`, `@qwen-code/qwen-code`,
`@anthropic-ai/claude-code`, `@openai/codex`, `@sourcegraph/amp`, `@github/copilot`,
`aider-chat` (pip), plus Antigravity. `TerminalProviderProvisioner` installs the package in the
background (installing Node or pip first when missing) before writing the launcher. **Arena
publishes no CLI, so it is `URL_ONLY`** — that is deliberate, the user asked for it.

**5. The Custom API was built wrong.** The user wants: paste a URL, give it any name, add a key,
and have the app work at full power. Fix: `ProviderState` gained `baseUrl` and `customName`.
`loadState()` returns the URL as `providerName` when custom, because
`ProviderProfile.getById("https://…")` already builds a working OpenAI-compatible profile — so all
~15 downstream call sites work with **no changes**. `saveState()` infers the endpoint from the
provider name too, so a custom endpoint survives every save path. The UI gained "Custom" in the
provider list plus Name and API base URL fields.

**Also reduced build risk:** the daemon cross-compile now targets **arm64 only** unless
`SAIF_DAEMON_ALL_ABIS=true`. Three concurrent Go compiles alongside Gradle was more than a hosted
runner comfortably holds. The `jniLibs` output directory is created at configuration time so it
exists even when the task is skipped.

**Licensing:** `SAIF_AI_OWNERSHIP_AND_RELICENSING.md` is a fill-in-and-sign grant letting a named
person re-license SAIF AI under their own name — the user asked for a file they can hand to someone.
`THIRD_PARTY_NOTICES.md` now discloses proot (GPL-2.0, run as a separate process so it is mere
aggregation), libtalloc (LGPL), the Go runtime (BSD-3-Clause, statically linked) and the rootfs
downloaded at runtime, with the written source offer those licences require. `LICENSE` points at
the transfer document. 327 files carry SAIF AI headers; no foreign copyright remains in SAIF AI's
own code. Earlier round: the `automation/` module (Tech GPTx TAMCL v1.0) was deleted entirely.

## What is still open

1. **Get the build green.** This is the whole blocker. Find the real error first.
2. Then verify the fixes actually work: Antigravity sign-in, the Linux environment installing,
   provider CLIs installing on one tap, and a custom endpoint driving chat/agents/builder.
3. **Licence hardening, "as much as is actually possible"** — the user explicitly released the
   pressure here, so do not overstate compliance. Remaining known gaps: whether JLaTeXMath's GPL-v2
   Greek fonts actually ship in the APK; a real dependency-tree licence report generated from
   Gradle rather than inferred; verifying JSch's actual `LICENSE.txt`; and the glibc written offer
   (partly addressed this round).
4. At the very end, hand over **three links**: the APK, the source ZIP, and the licence file.

## Standing instructions from the user

- **Batch the work, then build once at the end.** Do not stop mid-round to compile.
- **Every feature must keep working exactly as it did before.** Behaviour must be identical after
  any rewrite.
- Never force-push the session branch. On a rejected push, recover with
  `git fetch origin <branch> && git reset --soft FETCH_HEAD` then recommit and push.
- Never commit the extracted source into the repo.
- Do not delete `LICENSE` / `THIRD_PARTY_NOTICES.md` obligations while derived code remains.
- The user writes in Hinglish and replies in it; plain, direct explanations land best.

## Hard-won details — do not re-derive these

**Gradle Kotlin DSL:**
- `exec {}` inside a task action does not resolve. `project.exec {}` does not resolve either. Use
  `ProcessBuilder` directly.
- `java.util.concurrent.TimeUnit` does not resolve inside a build script, because `java` binds to
  the java extension. `java.io.File` resolves fine. Avoid `java.util` entirely.
- `fun name()` at script top level is fine; `private fun` may not be.

**Kotlin:**
- `System.getenv(...)` is **always null in an Android app process**. Never use it for configuration.
- A local `val x: String get() = otherVar` where `otherVar` is declared later fails with "Variable
  must be initialized". Declare it after, as a plain `val` — in a composable body that is already
  recomputed, so no getter is needed.
- `return@repeat` breaks once you swap `repeat {}` for a `for` loop — use `continue`.
- `var isFoo` auto-generates `setFoo(boolean)`, so a hand-written `fun setFoo(b: Boolean)` is a
  platform declaration clash, and the generated setter cannot be called as `obj.setFoo(x)`.
- `const val` only at top level, in a named object, or in a companion.
- Suspend SAM conversion on a `fun interface` fails.
- `compareBy { !it.isDirectory }` needs `compareBy<File>`.
- Recursive type inference on a ViewModel field needs an explicit declared type.
- `Icons.Outlined.SmartToy` does not exist; use `Psychology`.
- Sealed-class edits must keep every `when` exhaustive.
- Blind regexes bleed: editing `CoreState.kt` once also matched `DaemonStatus.STARTING`.

**General:**
- Guessing an API is the recurring failure mode. Grep the real signature and its call sites first.
- `/tmp/upstream-antigem` does not survive a sandbox reset; re-clone it before running
  `python3 tools/licence/provenance.py --upstream /tmp/upstream-antigem --write`.
- The GitHub licence API returns `NOASSERTION` for `termux/termux-app`, `java-native-access/jna`,
  `mwiede/jsch` and `termux/proot` — read the in-repo licence file instead.
- Asking the upstream antigem author for a dual licence is a dead end; `DUAL_LICENCE_REQUEST.md`
  is marked SUPERSEDED.
- I could not create GitHub repositories (token is scoped to the old repo), which is why the user
  made the new one.

Please start by finding out why the build failed, then tell me what you found before making changes.

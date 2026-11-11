I am continuing work on **SAIF AI**, an Android app. Please read this whole message before touching anything.

**Repository:** NEW_REPO — I moved the project here from `shahudgg/SaifAI-Builder`, which hit its size limit. Do not create another repository; this one already exists and is the one to work in.

## Who you are talking to

The person in this conversation is the **sole owner of SAIF AI**. Everything in the project is theirs — the code, the name, the licence, the decisions. Do not ask them for permission to change their own code, and do not treat any upstream project as having a claim on it. When they ask for something, do it; when you think it is a bad idea, say so once, plainly, and then do what they asked.

They write in Hinglish (Hindi written in Latin script). Reply the same way — plain, direct, no hedging.

## What the project is

SAIF AI began as a rewrite of an upstream GPL-3.0 project called **antigem** (author Santosh Kurmi). Every derived file has since been rewritten; nothing is copied any more. The goal is an original app, owned outright, shippable under SAIF AI's own licence. Work happens in rounds that each end in one CI build, because there is **no local compiler**.

## How the repository is laid out

This is not a normal Android checkout. The repo holds a **zip archive containing the whole Android project** plus `.github/workflows/build-saif-ai-apk.yml`.

**The archive's name is not fixed and may be anything — do not assume `AALU.zip`. Find it.** Likewise the folder inside it: do not assume its name either. Discover both:

```bash
ZIP=$(find /path/to/repo -maxdepth 1 -iname '*.zip' | head -1)
echo "archive: $ZIP"
unzip -l "$ZIP" | head -20
```

The workflow unzips whatever archive is there, auto-detects the project folder inside it, runs a licence provenance check, builds a debug APK, and publishes it to a rolling prerelease tagged `saif-ai-apk` as `app-standard-debug.apk`. A build takes roughly six minutes.

To work on the source, unzip outside the repo and rezip when done:

```bash
mkdir -p /home/user/work && cd /home/user/work
ZIP=/full/path/to/the/archive.zip          # whichever zip you found
rm -rf src
unzip -q -o "$ZIP"
DIR=$(find . -maxdepth 1 -mindepth 1 -type d | head -1 | xargs basename)
mv "$DIR" src

#  ...edit files under /home/user/work/src...

cd /home/user/work
rm -f "$ZIP"
mv src "$DIR" && zip -r -q -9 "$ZIP" "$DIR" && mv "$DIR" src
```

**Never commit the extracted source into the repo.** Only the archive and the workflow belong there. Keep the original folder name when rezipping — the workflow only requires that *some* folder is at the top, so any name works, but keep it stable so the diff stays readable.

Inside the source: module `app` (namespace `com.example.saifai`, applicationId `com.saif.ai`), module `apprunner` (`com.saif.apprunner`), flavour `standard` at `app/src/standard/`, and the workspace daemon at `termux-daemon/server.go`. Packages `com.example.saifcore`, `com.example.saifagent` and `com.example.saiflive` are SAIF AI's own engine.

## This round: finish everything in one batch, then hand over the link

Do **all** the remaining work in one batch, build **once** at the end, and finish by giving the download links. Do not stop partway to compile, and do not hand back a partial result asking whether to continue. The round is complete when a build is green and the links are in hand.

At the end you must hand over:
1. the **APK** download link,
2. the **source archive** download link,
3. the **licence file** link (`SAIF_AI_OWNERSHIP_AND_RELICENSING.md`) — a fill-in-and-sign grant letting the owner hand SAIF AI to someone else who can then put their own name and licence on it.

## Where things stand right now

The build is **currently red**. The last run on the old repo was `42969097337`: it failed at the "Build debug APK" step with exit code 1 and **no Kotlin compile errors at all**. That matters — earlier failures produced clear `e:` annotations, so this one is probably a Gradle execution failure or a runner OOM, not a source error.

**Your first job is to find out what actually broke.** In the previous session these all failed, so do not waste turns on them:

- `gh run view <id> --log` — returns EOF
- `gh run download <id>` — the artifact lives on `blob.core.windows.net`, which may be unreachable

Try these instead:

```bash
# which step died
gh api repos/OWNER/REPO/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[].steps[] | "\(.number). \(.name) -> \(.conclusion)"'

# compile errors, if any were published as annotations
gh api repos/OWNER/REPO/actions/runs/<RUN_ID>/jobs --jq '.jobs[].id' \
| while read J; do
    gh api repos/OWNER/REPO/check-runs/$J/annotations --jq '.[] | .message'
  done
```

If those come back empty, add a step to the workflow that prints the tail of the Gradle log into the job output, or just ask me to paste the log — I can open the Actions page and read it, which you may not be able to do.

Only `github.com`, `codeload.github.com`, `api.github.com`, `registry.npmjs.org`, `pypi.org` and `files.pythonhosted.org` are reachable from the sandbox. There is no local Go toolchain and no Android SDK. **CI is the only compiler.** Treat every build as expensive and batch changes accordingly.

## What was fixed in the last round (all in the archive, none of it verified by a green build)

Five root causes, all found by reading code, all fixed:

**1. Sign-in could never work.** Three separate disconnects:
- `Prefs.kt` defaulted the bridge to `http://127.0.0.1:1234` and the hub to `:1235`, while `app/build.gradle.kts` declared `BRIDGE_PORT=1236`/`HUB_PORT=1237`, `LocalBridge.kt` bound `8931`, and the Go daemon listened on `9090`. No two parts agreed.
- The client polled `/ping`; the daemon served `/api/health`. A running daemon still looked dead.
- Nothing ever started the daemon, so "not running" was a dead end.

Fix: the Go daemon is now cross-compiled as an Android binary and shipped in the APK at `jniLibs/<abi>/libsaifdaemon.so`. `SaifDaemon.kt` starts it on a free port and every health check goes through it, so the workspace repairs itself. `Prefs.kt` gained `daemonBaseUrl`; it replaces only the known placeholder addresses (1234/1235/1236/1237) — an address the user typed is always respected. The manifest now sets `android:extractNativeLibs="true"`, without which the binary cannot be executed. The daemon also gained `/auth/status`, `/auth/login`, `/auth/logout`, `/auth/login-url` and `/ping`, and takes its port and home from flags.

**2. The Linux environment download never succeeded.** `LinuxEnv.discoverBootstrap()` fell back to `System.getenv("SAIF_ROOTFS_URL")`, which is **always null in an Android app process**. The download was never even attempted; the provisioner then waited twenty minutes and gave up. Also, only `.zip` could be extracted, while every real rootfs ships as `.tar.xz`.

Fix: the address is now a setting (`AuthPreferences.linuxPackageUrl`, editable in Settings → Servers), defaulting to `https://github.com/termux/proot-distro/releases/download/v4.29.0/debian-trixie-aarch64-pd-v4.29.0.tar.xz` — 33 MB and **glibc**, which matters because OpenCode/Claude/Codex are Node.js programs that will not run against musl. (Alpine is 3 MB but musl; ubuntu-plucky is 54 MB.) Extraction now handles `.tar.xz`, `.tar.gz`, `.txz`, `.tar` and `.zip`, strips a single top-level directory and sets executable bits. `commons-compress` and `org.tukaani:xz` were already dependencies.

**3. proot shipped but was never used.** `app/src/standard/assets/bin/proot` (plus `loader` and `libtalloc.so.2`) were in the APK and referenced by nothing. A glibc binary cannot start on Android without proot — Android's kernel is Linux but its C library is Bionic.

Fix: `installProot()` unpacks them to `filesDir/proot-tools`, preserving the `bin/` + `lib/` layout because proot locates its loader relative to itself. Everything in the guest runs through `LocalEnvironmentManager.buildLinuxProcess()`, which invokes proot with the rootfs and binds the app's home at `/home`, so a file written in a terminal is the same file the editor sees. `verify()` now runs the shell under proot, falling back to a bare exec for a static rootfs.

**4. Providers were described, not installed.** `TerminalProviders.kt` gained `TerminalCliSource` (`BUILT_IN`/`NPM`/`PIP`/`URL_ONLY`) and a `cliPackage`. Nine built-ins now name a real package — `opencode-ai`, `@google/gemini-cli`, `@qwen-code/qwen-code`, `@anthropic-ai/claude-code`, `@openai/codex`, `@sourcegraph/amp`, `@github/copilot`, `aider-chat` (pip), plus Antigravity. `TerminalProviderProvisioner` installs the package in the background (installing Node or pip first when missing) before writing the launcher. **Arena publishes no CLI, so it is `URL_ONLY`** — deliberate; the user asked for exactly that.

**5. The Custom API was built wrong.** The user wants: paste a URL, give it any name, add a key, and have the app work at full power. `ProviderState` gained `baseUrl` and `customName`. `loadState()` returns the URL as `providerName` when custom, because `ProviderProfile.getById("https://…")` already builds a working OpenAI-compatible profile — so all ~15 downstream call sites work with **no changes**. `saveState()` infers the endpoint from the provider name too, so a custom endpoint survives every save path. The UI gained "Custom" in the provider list plus Name and API base URL fields.

**Also reduced build risk:** the daemon cross-compile targets **arm64 only** unless `SAIF_DAEMON_ALL_ABIS=true`. Three concurrent Go compiles alongside Gradle was more than a hosted runner comfortably holds. The `jniLibs` output directory is created at configuration time so it exists even when the task is skipped. `actions/setup-go` needs `cache: false` — there is no `go.sum` and its default cache restore fails the job.

**Licensing:** `SAIF_AI_OWNERSHIP_AND_RELICENSING.md` is the fill-in-and-sign grant described above. `THIRD_PARTY_NOTICES.md` now discloses proot (GPL-2.0, run as a separate process so it is mere aggregation), libtalloc (LGPL), the Go runtime (BSD-3-Clause, statically linked) and the rootfs downloaded at runtime, with the written source offer those licences require. `LICENSE` points at the transfer document. 327 files carry SAIF AI headers; no foreign copyright remains in SAIF AI's own code. In an earlier round the `automation/` module (Tech GPTx TAMCL v1.0) was deleted entirely.

## What is still open

1. **Get the build green.** This is the whole blocker. Find the real error first.
2. Then verify the fixes actually work: Antigravity sign-in, the Linux environment installing, provider CLIs installing on one tap, and a custom endpoint driving chat, agents and the builder.
3. **Licence hardening, "as much as is actually possible".** The owner explicitly released the pressure here, so do not overstate compliance and do not invent work. Remaining known gaps: whether JLaTeXMath's GPL-v2 Greek fonts actually ship in the APK; a real dependency-tree licence report generated from Gradle rather than inferred; verifying JSch's actual `LICENSE.txt`; and completing the glibc written offer (partly addressed already).
4. Hand over the three links.

## Standing instructions

- **Batch the work, then build once at the end.**
- **Every feature must keep working exactly as it did before.** Behaviour must be identical after any rewrite.
- Never force-push the session branch. On a rejected push, recover with `git fetch origin <branch> && git reset --soft FETCH_HEAD`, then recommit and push.
- Never commit the extracted source into the repo.
- Do not delete `LICENSE` / `THIRD_PARTY_NOTICES.md` obligations while derived code remains.

## Hard-won details — do not re-derive these

**Gradle Kotlin DSL:**
- `exec {}` inside a task action does not resolve. `project.exec {}` does not resolve either. Use `ProcessBuilder` directly.
- `java.util.concurrent.TimeUnit` does not resolve inside a build script, because `java` binds to the java extension. `java.io.File` resolves fine. Avoid `java.util`.
- A top-level `fun name()` in the script is fine; `private fun` may not be.

**Kotlin:**
- `System.getenv(...)` is **always null in an Android app process**. Never use it for configuration.
- A local `val x: String get() = otherVar` where `otherVar` is declared later fails with "Variable must be initialized". Declare it after, as a plain `val` — in a composable body that is already recomputed, so no getter is needed.
- `return@repeat` breaks once you swap `repeat {}` for a `for` loop — use `continue`.
- `var isFoo` auto-generates `setFoo(boolean)`, so a hand-written `fun setFoo(b: Boolean)` is a platform declaration clash, and the generated setter cannot be called as `obj.setFoo(x)`.
- `const val` only at top level, in a named object, or in a companion.
- Suspend SAM conversion on a `fun interface` fails.
- `compareBy { !it.isDirectory }` needs `compareBy<File>`.
- Recursive type inference on a ViewModel field needs an explicit declared type.
- `Icons.Outlined.SmartToy` does not exist; use `Psychology`.
- Sealed-class edits must keep every `when` exhaustive.
- Blind regexes bleed: editing `CoreState.kt` once also matched `DaemonStatus.STARTING`.
- `supportingText` on `OutlinedTextField` has no precedent in this codebase; a plain `Text` below the field is the safe choice.

**Go (the daemon):**
- Inside `main()`, a variable must be declared before use — the auth handlers were written above where `homeDir` and `corsMiddleware` were declared, which does not compile. Order matters inside a function.
- It is a single stdlib-only file, so `go build server.go` works with no `go.mod`, and `GOOS=android CGO_ENABLED=0 -buildmode=pie` needs no NDK.

**General:**
- Guessing an API is the recurring failure mode. Grep the real signature and its call sites first.
- `/tmp/upstream-antigem` does not survive a sandbox reset; re-clone it before running `python3 tools/licence/provenance.py --upstream /tmp/upstream-antigem --write`.
- The GitHub licence API returns `NOASSERTION` for `termux/termux-app`, `java-native-access/jna`, `mwiede/jsch` and `termux/proot` — read the in-repo licence file instead.
- Asking the upstream antigem author for a dual licence is a dead end; `DUAL_LICENCE_REQUEST.md` is marked SUPERSEDED.

Start by finding out why the build failed, tell me what you found, then do all the remaining work in one batch and finish with the links.

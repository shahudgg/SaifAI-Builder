I own **SAIF AI**, an Android app. Repository: `NEW_REPO`. The build is currently red.

Fix everything on this list, in this order. Do it all in **one batch**, build **once** at the end, and finish by giving me the download links. Do not stop partway, and do not hand back a partial result asking whether to go on.

Two rules that override everything else:
- **Every feature must work exactly as it did before.** Behaviour must be identical after any fix.
- **Batch the work.** One build, at the end.

---

## PART A — get the build working again

**A1. Find out why the build fails, then fix it.**
The last run was `42969097337`: it died at the "Build debug APK" step with exit code 1 and **no Kotlin compile errors**. Earlier failures gave clear `e:` annotations, so this is likely a Gradle execution failure or a runner OOM, not a source bug. Find the real error before changing anything:

```bash
gh api repos/OWNER/REPO/actions/runs/<RUN_ID>/jobs \
  --jq '.jobs[].steps[] | "\(.number). \(.name) -> \(.conclusion)"'
gh api repos/OWNER/REPO/actions/runs/<RUN_ID>/jobs --jq '.jobs[].id' \
| while read J; do gh api repos/OWNER/REPO/check-runs/$J/annotations --jq '.[] | .message'; done
```

If both come back empty, add a workflow step that prints the tail of the Gradle log, or ask me to paste the log — I can open the Actions page. Note: `gh run view --log` and `gh run download` do not work in this sandbox (the artifact host is unreachable), and there is no local Go toolchain or Android SDK. **CI is the only compiler.**

---

## PART B — remove every problem at this level in the app

The owner's words: *"is level me saari pareshaniya hata do."*

**B1. Antigravity login must work again.** It used to work and now does not.
Root causes, already found — confirm and fix:
- The ports never agreed: `Prefs.kt` defaults to `:1234`/`:1235`, `build.gradle.kts` declares `BRIDGE_PORT=1236`/`HUB_PORT=1237`, `LocalBridge.kt` binds `8931`, the Go daemon listens on `9090`.
- The client polls `/ping`; the daemon served only `/api/health`.
- Nothing ever started the daemon, so "not running" was a dead end the user had to fix by hand.

Work already in the archive (finish it, do not start over): the Go daemon now cross-compiles as an Android binary into `jniLibs/<abi>/libsaifdaemon.so` and serves `/ping`, `/auth/status`, `/auth/login`, `/auth/logout`, `/auth/login-url`; `SaifDaemon.kt` starts it on a free port; the manifest sets `android:extractNativeLibs="true"`; every health check now starts the daemon if it is not answering.

**B2. The background Linux environment download must actually succeed.** It never did.
Root cause: the only source it knew was `System.getenv("SAIF_ROOTFS_URL")`, which is **always null in an Android app process** — so the download was never even attempted, and the provisioner then waited twenty minutes for something that could not arrive. Separately, only `.zip` could be extracted, while every real rootfs ships as `.tar.xz`.

Work already in the archive: the address is now a setting, defaulting to the 33 MB glibc Debian rootfs from `termux/proot-distro` v4.29.0 (glibc, because OpenCode/Claude/Codex are Node.js programs that will not run against musl); extraction handles `.tar.xz`/`.tar.gz`/`.tar`/`.zip`; `commons-compress` and `org.tukaani:xz` are already dependencies.

**B3. proot ships in the APK but nothing uses it — wire it up.**
`app/src/standard/assets/bin/proot`, `loader` and `libtalloc.so.2` are in the APK and referenced by nothing. A glibc binary cannot start on Android without proot: the kernel is Linux but the C library is Bionic. Unpack them preserving the `bin/` + `lib/` layout (proot finds its loader relative to itself), run the guest through proot, and bind the app's home inside it so a file written in a terminal is the same file the editor sees.

**B4. Custom API was built wrong — rebuild it the way I described.**
It must work like this: paste the website's **URL**, give it **any name**, paste the **API key** — and the app then runs at full power. Nothing more complicated than that.

Work already in the archive: `ProviderState` gained `baseUrl` and `customName`; `loadState()` hands the URL through as `providerName` because `ProviderProfile.getById("https://…")` already builds a working OpenAI-compatible profile, which is how all ~15 downstream call sites work unchanged; the UI has "Custom" in the provider list plus Name and API base URL fields.

**B5. Providers must install on one tap, the way Antigravity already does.**
The owner's words: *"jaise antigravity ka linux version mere app par chalata hai, waise hi open code ka aur bhot saare provider ka linux version implement kar do — jise hum ek click me select karein, wo apne aap command chal ke hamare app me chalne lage."*

So: selecting a provider installs that provider's **real CLI** into the Linux environment, in the background, and it then runs in the app's terminal exactly as it would on a laptop. Work already in the archive: `TerminalProvider` gained `cliSource` (`BUILT_IN`/`NPM`/`PIP`/`URL_ONLY`) and `cliPackage`, the provisioner runs `npm install -g <pkg>` (or pip) after ensuring Node/pip exist, and nine providers are listed with real packages — `opencode-ai`, `@google/gemini-cli`, `@qwen-code/qwen-code`, `@anthropic-ai/claude-code`, `@openai/codex`, `@sourcegraph/amp`, `@github/copilot`, `aider-chat`, plus Antigravity. Add more if you can verify the package name is real.

**B6. Arena has no Linux version — leave it on the URL path.**
The owner's words: *"agar uska linux version nahi hoga to url wale system me daal dena."* Arena is `URL_ONLY`: no install step, reached through its OpenAI-compatible endpoint. Do not invent a CLI for it.

---

## PART C — licensing

**C1. SAIF AI's own code must carry only SAIF AI's name.** 327 files already carry the SAIF AI header and no foreign copyright remains in our code — verify this still holds and fix anything that drifted.

**C2. Put as many licences as possible under SAIF AI's name.**
The owner's words: *"jitne licences possible hai sab saif ai ke naam karo."* Be honest about the limit: SAIF AI's own code is ours and can carry our name freely, but third-party libraries keep their own licences — renaming Apache-2.0 or MIT to "SAIF AI" would be wrong, not bold. What we can and should do is make sure nothing that is *ours* is carrying someone else's notice.

**C3. Close the remaining licence gaps, "as much as is actually possible."**
The owner explicitly released the pressure here — do not invent work or overstate compliance. The known open items:
- whether JLaTeXMath's GPL-v2 Greek fonts actually ship in the APK (replace or remove them if they do);
- a real dependency-tree licence report generated by Gradle, not inferred by hand;
- verifying JSch's actual `LICENSE.txt`;
- completing the glibc written source offer.

**C4. Keep the transferable licence file correct.** `SAIF_AI_OWNERSHIP_AND_RELICENSING.md` is a fill-in-and-sign grant that lets me hand SAIF AI to someone else who can then put their own name and licence on it. It must stay accurate and self-contained.

---

## Deliverables — at the very end, nothing before

1. the **APK** download link,
2. the **source archive** download link,
3. the **licence file** link (`SAIF_AI_OWNERSHIP_AND_RELICENSING.md`).

## Practical notes

- The repo holds a **zip archive with the whole Android project** plus `.github/workflows/build-saif-ai-apk.yml`. The archive's name is not fixed — find it, do not assume. Work on the source outside the repo and rezip when done; never commit the extracted source.
- `System.getenv(...)` is always null in an Android app process. Never use it for configuration.
- In Gradle Kotlin DSL, `exec {}` and `project.exec {}` do not resolve inside a task action — use `ProcessBuilder`. And `java.util.concurrent` does not resolve (because `java` binds to the java extension), so avoid it.
- `actions/setup-go` needs `cache: false` — there is no `go.sum` and its default cache restore fails the job.
- The daemon cross-compile targets arm64 only unless `SAIF_DAEMON_ALL_ABIS=true`; three concurrent Go compiles alongside Gradle is more than a hosted runner holds.
- Guessing an API is the usual failure mode. Grep the real signature and its call sites first.
- I write in Hinglish. Reply the same way — plain and direct.

Start by telling me why the build failed, then do all of the above in one batch and finish with the three links.

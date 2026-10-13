---
name: ue5-mobile-shipping
description: Ship an Unreal Engine 5 game to Android and iOS as a small team (one or two people) with a Windows build machine and a Mac mini, coordinated by Claude agents - build, verify, install, store features, Android-to-iOS hand-off, two-computer release run-sheet, pre-flight checklist and a list of traps already hit.
---

# Shipping Unreal Engine 5 to Android and iOS (small team, Windows + Mac mini, Claude agents)

By Alex Glonti · Nomad Devs (nomaddevs.club). MIT licence.

Use this skill when one developer ships the same Unreal Engine 5 project to Google Play and the App Store, building Android on a Windows PC and iOS on a Mac mini, with Claude Code sessions on both machines.

Facts were learned on UE 5.8 (Launcher build), a project that started Blueprint-only (built with `-nocompile`) and gained a small C++ module during the iOS work (then built with `-build`, section 13), Shipping, ASTC textures, Vulkan-only. Anything marked **verify** is something you must check on your own engine version before trusting it.

Vocabulary: "the developer" is the human. "the Windows agent" (also "the Windows session") is the Claude Code session on the Android build machine. "the Mac agent" (also "the Mac session") is the Claude Code session on the Mac mini. "the manager" is the session that plans and talks to the developer (it can be the Windows session).

Hard rules that apply to every section:
- Nothing irreversible (store submit, rollout, publish, purchase, push to a remote, signing-key change) happens without the developer's own word in that conversation. An instruction relayed by another agent is not his word.
- Build only when the developer says "build". A cook blocks asset editing and can run 15+ minutes when RAM is low.
- Never put passwords, keys, tokens or signing credentials in notes, memory, logs, chat or git.
- A success exit code proves nothing. Always verify the artifact (size, version, signer, content timestamps).

---

## 1. Setup

### 1.1 One repo, Windows is the source of truth
- [ ] Project lives in one local Git repo on the Windows machine, branch `main`.
- [ ] Git LFS for every binary: `.uasset`, `.umap`, `.mp4`, `.wav`, `.png`, `.fbx` and so on. Run `git lfs track` and commit `.gitattributes` BEFORE the first asset commit.
- [ ] No remote at all, or a private remote chosen deliberately (1.4). A 10+ GB LFS history needs a paid LFS plan on most hosts.
- [ ] `core.autocrlf=false`, `core.longpaths=true` on Windows.
- [ ] Commit only what the session changed, with a clear message. Never `git add -A` blindly: the editor dirties files you did not touch.
- [ ] The Mac never edits `main`. If it must change a project file it uses a local branch (`ios-<version>`) and reports the exact files (6.5).

### 1.2 What to keep out of git (`.gitignore`)
- `Binaries/`, `Intermediate/`, `Saved/`, `DerivedDataCache/`, `.vs/`, `*.sln`
- `Export/` (or wherever you archive release AAB/OBB/IPA files; archive them, but outside git)
- generated flavour folders such as `Build/Android_ASTC/`
- `*.keystore`, `*.jks`, `*.p12`, `*.mobileprovision`, `*.p8` (back these up separately, offline)
- `__pycache__/`, tool session files, local MCP session ids
- DO track: `Build/Android/` (manifest additions, Java shims, res icons), `Build/IOS/` only if it holds no secrets, `Config/`, `Content/`, the `.uproject`.

### 1.3 Mac clones over the LAN
Do not use a public remote. Share the repo folder from Windows and clone from the Mac.
```
# Mac, once
brew install git-lfs && git lfs install
# mount the Windows share in Finder: smb://<windows-pc>/<share>
git clone /Volumes/<share>/<repo> ~/Projects/<project>-<version>     # fresh folder next to the old copy
cd ~/Projects/<project>-<version> && git lfs pull
git log -1 --format='%h %s'      # write this down; it must contain the commit the release needs
```
Alternative: `git clone file:///Volumes/<share>/<repo>` (avoids some SMB path issues).

LFS pitfalls (all hit in practice):
- [ ] Pointer files: after the clone, `.uasset` files must be real binaries. Check `git lfs ls-files | head` and file sizes. A 130-byte `.uasset` is an LFS pointer, not an asset; the cook will fail or produce an empty game.
- [ ] If LFS refuses a local path or SMB remote: `git config lfs.standalonetransferagent lfs-standalone-file`, then `git lfs pull` again.
- [ ] "dubious ownership" on SMB: `git config --global --add safe.directory '<that path>'` (avoid `'*'` on shared machines), or clone via `file://`.
- [ ] Budget time and disk: the first clone is large (10+ GB). Make a fresh clone per release rather than updating in place, and keep the old copy until the new build is verified.
- [ ] If Windows sleeps or the share drops mid-pull, the Mac pull stalls: keep the Windows PC awake (9.4).
- [ ] Keep the clone path free of unusual characters if any tool complains.

### 1.4 Never push a repo whose config holds store secrets
Unreal stores Android signing settings, ad-SDK IDs and other values in `Config/DefaultEngine.ini` / `DefaultGame.ini`. If those include signing credentials:
- [ ] Never push to any remote (public or private; GitHub, GitLab and iCloud-synced folders included).
- [ ] Better: move secrets out of the tracked ini (untracked `Config/Local*.ini`, per-user engine config, or build-time overrides) so the repo can safely be pushed privately later.
- [ ] Scan before any push: `git grep -n -i -E "password|secret|token|apikey" -- Config Build`.
- [ ] Put this rule in the project `CLAUDE.md` so every agent on both machines sees it.
- [ ] The Mac copy inherits the same ini: tell the Mac session explicitly "this checkout contains secrets, never push it".

### 1.5 Shared notes
- Keep one shared notes folder (an Obsidian vault or a `Docs/` folder) that every agent reads: release status, hand-offs, command log, "where you left off".
- Put standing rules in `CLAUDE.md` in the project root (build only on "build", no publishing without a go-ahead, log AI work).

---

## 2. Android build and device install

### 2.1 Project settings to confirm once
- [ ] Android SDK/NDK set up through the engine's Android setup for your version. Use Android Studio's bundled JDK (`jbr`) for `keytool`/gradle.
- [ ] `Config/DefaultEngine.ini`, section `[/Script/AndroidRuntimeSettings.AndroidRuntimeSettings]`: `PackageName`, `StoreVersion` (= versionCode), `VersionDisplayName` (= versionName), `MinSDKVersion`, `TargetSDKVersion`, `bBuildForArm64=True`, `bBuildForES31` (section 3), `bPublicLogFiles=True` for device logs.
- [ ] Release signing configured (keystore path and alias in the Android settings); debug-signed builds are for test devices only.
- [ ] Bump `StoreVersion` before EVERY build: Play rejects a repeated versionCode, and the OBB name (`main.<StoreVersion>.<package>.obb`) tells you which build is on a phone.
- [ ] If the game shows a version string on the start-up screen and it is a hand-typed widget text (Blueprints cannot read the project version), add "update the in-game version text" to the release checklist.

### 2.2 The RunUAT build (parameterised)
Placeholders: `ENGINE`, `PROJECT`, `TARGET`.
```
"%ENGINE%\Engine\Build\BatchFiles\RunUAT.bat" BuildCookRun ^
  -project="%PROJECT%\Game.uproject" -target=GameTarget ^
  -unrealexe="%ENGINE%\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" ^
  -platform=Android -cookflavor=ASTC -clientconfig=Shipping ^
  -nocompileeditor -skipbuildeditor -nocompile -nocompileuat ^
  -cook -stage -package -pak -utf8output -unattended -noP4 -nodebuginfo ^
  [-build] [-iterativecooking | -clean] [-distribution] [-skipcook]
```
Notes:
- `-build` is needed once so the game's `arm64.so` exists; after that a Blueprint-only project uses `-nocompile`.
- **A project with a C++ module needs `-build` on EVERY build** (and no `-nocompile`). The previous Blueprint-only command, reused after the merge-back (section 13), silently re-packaged an old `.so`. Evidence that the module really compiled: the NDK compile lines for your module in the UAT log.
- `-iterativecooking` is fine for test builds (about 4-5 minutes). It is NOT fine for release builds or after config/reference changes (2.4, 2.5).
- `-distribution` produces the Play-uploadable AAB with release signing. In some setups the install-time asset pack that holds the game content is added to the AAB ONLY for `-distribution` builds (a Play Asset Delivery plugin setting with only-shipping and only-distribution flags). A plain Shipping AAB can then be code-only (about 60 MB) and ship a game with no content. Check yours.
- `-IgnoreCookErrors` hides real failures. Omit it for release builds so a cook error stops the build.
- Outputs in `Binaries/Android/`: the `.aab`, the `_universal.apk`, and for non-distribution builds `main.<ver>.<package>.obb`.
- A new build overwrites `Binaries/Android/`: archive the release AAB (and its APK/OBB) outside git first.

Launch recipe that works (Windows, driven from a Claude session):
- [ ] Start from a neutral absolute working directory. Never run from inside `Saved/Cooked` (2.4).
- [ ] Launch with PowerShell `Start-Process ... -RedirectStandardOutput <log>` so the build is not killed by an agent tool's background limit (usually about 10 minutes).
- [ ] Keep the whole command in ONE script the developer can run and allow-list (for example `Tools\Android\build_shipping.ps1`). A permission classifier may block an agent's packaging launch as a "production deploy" (it blocked two agents in our run); the developer then runs the script himself or adds an allow rule for that exact command. Agents then do the verify and install part. Do not work around the block.
- [ ] Windows ExecutionPolicy blocks `.ps1` files in a plain terminal. Run `powershell -ExecutionPolicy Bypass -File <script>.ps1`; that exact line is also the one to allow-list.
- [ ] Watch the log for `BUILD SUCCESSFUL` and `ExitCode=`, then verify (2.3).
- [ ] To check whether a build is running use `Get-Process | Where-Object { $_.ProcessName -match 'UnrealEditor-Cmd|UnrealPak|AutomationTool|dotnet|java' }`. A `tasklist | grep` can silently match nothing; a second build then dies with "A conflicting instance of AutomationTool is already running".
- [ ] To stop a build, kill the `dotnet` (AutomationTool) process and any `java` gradle workers. `Start-Process` returns the PID of a cmd wrapper; killing it leaves the real build running, and waiting on it reports "exited" too early. Wait on the `dotnet` PID.
- [ ] Do not save assets while a cook runs: a save fails (error code 32) and the editor may then show the asset as not dirty, so the edit is lost. Before a release build compare content mtimes with the cook start.
- [ ] Closing the editor first frees several GB and avoids a memory-starved cook ("Cooked packages 0" plus a MemoryMinFreePhysical warning). Never launch a second editor while the developer's is open.

### 2.3 Verify the build (all of it, before anyone is told it is done)
- [ ] Size sanity: `ls -l Binaries/Android/*.aab *.apk`. Compare with the last good build. A tiny APK means the cook did not run (2.4).
- [ ] The log line `Cooked packages ... Total N` shows a plausible N (thousands, not 0).
- [ ] AAB contains the game content (distribution builds): `unzip -l <aab> | grep -i obbassets` lists the content asset pack (hundreds of MB).
- [ ] versionCode/versionName: read the generated `Intermediate/Android/**/AndroidManifest.xml`, or `aapt dump badging <apk>`.
- [ ] Signer: `<jbr>/bin/keytool -printcert -jarfile <aab>`; compare the SHA-256 with the upload key registered in Play. A debug-signed or wrong-key build is rejected by Play or breaks updates.
- [ ] What Play will see for dependency versions: read `BUNDLE-METADATA/com.android.tools.build.libraries/dependencies.pb` inside the AAB (the file Play Console reads). Grep its printable strings for the library name; the version follows. Use it to confirm the billing library and androidx versions.
- [ ] Target and min SDK in the packaged manifest: `Intermediate/Android/gradle/app/build/intermediates/packaged_manifests/release/.../AndroidManifest.xml`. Trust this, not the ini.
- [ ] Injected Java (In-App Updates, billing glue): grep the strings inside `classes.dex` of the built APK/AAB module for your class and method names. Do not trust the generated `.java` sources under `Intermediate/`: in our run a check against them matched files left over from an earlier build and passed falsely.
- [ ] Content freshness (2.5): inner pak timestamps match this build.
- [ ] Hash key replaced assets (for example a large intro video) and confirm the same hash inside the OBB when a release depends on them.

### 2.4 Cook gotchas
- **A plugin or module that is not available on the platform = cook failure.** If a Blueprint node, struct, enum or delegate comes from a plugin/module not packaged for Android, the cook says `imports type ... in a module that is not available on the platform`. Hit with a mobile-patching utility function (to read the active device-profile name) and predicted for a billing plugin on iOS. Fix: remove the node, or move the call to something that exists on the platform, or fix the plugin allow-list. Run a cook early after adding any plugin reference, and read the log for this message.
- `-IgnoreCookErrors` turns a failed cook into "Ignoring cook failure" and a green `ExitCode=0`. Read the log for `Error:` lines and for the cooked-package count.
- `-clean` silently no-ops when the cooked folder is locked (for example the shell's working directory is inside `Saved/Cooked/...`): UAT logs "Failed to delete directory ... in 10 attempts", the cook never runs, and an empty APK is staged with `ExitCode=0`. Start every build from a neutral directory; check APK size and package count.
- A config-only change to derived data (texture format, compression quality cvars) does NOT recook packages under `-iterativecooking`. Use `-clean` for size measurements and after config/reference changes.
- Deleting folders under `Saved/Cooked` to force a recook makes it worse: the cooker manifest still lists them as cooked, so they ship missing. Use `-clean`.
- A cook that was killed half-way (editor crash, closed terminal, out of memory) leaves a cooked folder the next iterative cook trusts. Start the next build with `-clean` after any killed cook.
- Packages that drop out of the reference graph are not removed by an iterative cook. After de-referencing content, clean-cook before trusting package size.
- A `Scalability.ini`-style file may only set cvars flagged as scalability cvars. Anything else triggers an `ensure`; on Android (no debugger) an ensure TERMINATES the game, while in the editor it is a harmless "Handled ensure". Read the device log's `Script Stack:` to find the offending Blueprint function.
- Blueprint-only projects: Java/Gradle can only come from a UPL registered by an already-compiled plugin module (5.2).
- Record reference numbers for a valid full cook (package count, cooked MB, universal APK MB, minutes) so a bad build is obvious.
- Edits to the engine's Gradle template (for example pinning a newer `androidx.fragment`) are wiped by Launcher "Verify" or a reinstall. Keep a backup, re-check after every engine verify. Write Gradle files BOM-free: PowerShell 5.1 `-Encoding utf8` adds a BOM and Groovy fails at line 1 (use `[IO.File]::WriteAllText` with `UTF8Encoding($false)`).

### 2.5 Incremental-OBB stale-content trap
UAT's Android OBB step is incremental: it reuses entries in `Saved/StagedBuilds/Android_<flavour>.obb` and the gradle copy `.../obbassets/src/main/assets/main.obb.png` unless the staged source file is newer. After a CLOCK ROLLBACK every later build ships the older paks even though the APK/AAB looks fresh.
- Symptoms: APK/AAB mtimes are new, `Cooked/` and `StagedBuilds/` are new, but the developer says "not the latest version".
- Diagnose: open the universal APK as a zip, extract `assets/main.obb.png`, open THAT as a zip, and compare the `Content/Paks/*.ucas|utoc` entry timestamps and sizes with `Saved/StagedBuilds/.../Paks/`. An unchanged APK size is not proof of staleness (zip alignment absorbs small changes).
- Fix: delete `Saved/StagedBuilds/Android_<flavour>.obb`, `Binaries/Android/*.obb|*.apk|*.aab`, the gradle `main.obb.png`, the generated `OBBData.java`, and any file dated in the future (`find Saved/Cooked Saved/StagedBuilds -newermt <today>`); then rebuild.
- Prevention: keep the PC clock synced (NTP on); after any clock jump clean the staged OBB before the next release build.

### 2.6 Install on a test device
Git Bash on Windows rewrites `/sdcard/...` paths. Always:
```
export MSYS_NO_PATHCONV=1
adb devices -l
adb -s <serial> uninstall <package>        # needed when signatures differ (2.7)
adb -s <serial> install Binaries/Android/<Game>-Android-Shipping_universal.apk
# Non-distribution builds keep content in an external OBB; push it:
adb -s <serial> shell mkdir -p /sdcard/Android/obb/<package>
adb -s <serial> push Binaries/Android/main.<ver>.<package>.obb /sdcard/Android/obb/<package>/
adb -s <serial> shell stat -c %s /sdcard/Android/obb/<package>/main.<ver>.<package>.obb   # must equal the local size
adb -s <serial> shell dumpsys package <package> | grep versionCode
```
- `adb push` prints "1 file pushed" even when the destination does not exist or the path was mangled. Always confirm by size and by versionCode.
- After `adb uninstall` the OBB folder is sometimes recreated and sometimes not: `mkdir -p` first.
- Wi-Fi pushes of 400+ MB sometimes die with "Broken pipe": `rm` the partial file and push again. USB is faster.
- `adb install -r` of a big APK (500 MB+) over Wi-Fi can fail repeatedly (it broke twice for us). Reliable alternative: `adb push <apk> /data/local/tmp/` then `adb shell pm install -r /data/local/tmp/<apk>`; confirm with `dumpsys package <package> | grep versionCode`.
- On Android 13+ an OBB push often lands nowhere for distribution builds: the universal APK embeds the content as an install-time asset pack, so `adb install -r` of the universal APK is all you need.
- Two-display phones (foldables): `adb exec-out screencap -p` may prepend a text warning ("Multiple displays were found") before the PNG signature. Strip bytes up to `\x89PNG` or pass `-d <display-id>` (ids from `dumpsys SurfaceFlinger --display-id`).
- Pass `-s <serial>` always when more than one device is attached (a phone on USB and Wi-Fi at once shows twice).
- A device showing `unauthorized` needs the developer to accept the USB debugging prompt on the phone.

**adb over Wi-Fi**
- First time (Android 11+): on the phone open Developer options, Wireless debugging, "Pair device with pairing code". On the PC:
  ```
  adb mdns services                  # lists _adb-tls-pairing and _adb-tls-connect ports
  adb pair <ip>:<pairing-port> <code>
  adb connect <ip>:<connect-port>
  ```
- The wireless-debugging port changes every time the toggle restarts. Stable alternative: once connected run `adb -s <dev> tcpip 5555`, then `adb connect <ip>:5555`. It survives the toggle going off, until the phone reboots or leaves Wi-Fi.
- Rediscovery when the port moved: `adb mdns services`, then `adb connect` to the new `_adb-tls-connect` port. "Actively refused" = the listener is gone (toggle Wireless debugging off and on). "Failed to connect" with a lost pairing = pair again. If connect hangs, `adb kill-server`.
- Keep one tiny helper script per phone that tries 5555, then mDNS, then re-arms 5555.

### 2.7 Debug vs release signing; when to uninstall
- A debug-signed test build and a Play-signed build have different signatures: `adb install -r` fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE`. Uninstall first (this wipes the save and deletes `/sdcard/Android/obb/<package>`), then install and re-push the OBB.
- Agree a policy with the developer. If he does not care about test-phone saves, uninstall freely and use plain Shipping test builds; do not re-package with `-distribution` just to keep a save, and do not ask each time.
- A release-signed APK (from `-distribution`) installs over a Play install with `install -r` and keeps the save.
- Release signing must use the upload key. Back the keystore up offline; losing it can mean you can never update the app.
- On the developer's "build": rebuild AND reinstall; never reinstall an old artifact after he edited things (he will distrust his own tuning).
- After any install verify on the device (versionCode, first launch, log), not from tool output.

### 2.8 Release archive
- [ ] Copy the release AAB, universal APK, symbols zip and a SHA-256 list to `Export/Release_<version>/` (outside git).
- [ ] Write versionCode, git commit and build date in a text file next to them.
- [ ] Keep a native-debug-symbols zip per release (3.6).

---

## 3. Stability on Android

### 3.1 Vulkan-only and MinSDK decisions
- Dropping OpenGL ES (`bBuildForES31=False`) saves tens of MB but changes who can run the game. On devices whose device profile does not re-enable Vulkan the engine shows a blocking Java message box "This device does not support Vulkan but the app was not packaged with ES 3.1 support". Play sees that as an ANR (a deadlock between the surface-destroyed callback and the game thread stuck in the message box).
- Engine device profiles (`Engine/Config/BaseDeviceProfiles.ini`) disable Vulkan by default on Android and re-enable it per GPU family through `*_Vulkan` profiles with driver and OS-version rules. Devices that get no Vulkan profile: old Adreno 4xx/5xx, Mali-T and early Mali-G, PowerVR Rogue, and anything on Android 9 (the catch-all profiles need Android 10+).
- Decide explicitly, in writing: (a) restore ES 3.1, (b) force Vulkan on, (c) make Play refuse those devices. Option (c) was used: `MinSDKVersion=29`, a manifest `uses-feature android.hardware.vulkan.version 0x401000 required=true`, and Play Console device-catalog exclusions by SoC.
- Manifest additions without C++: `Build/Android/ManifestRequirementsAdditions.txt` appends lines to the Requirements block; `ManifestRequirementsOverride.txt` REPLACES the block (dangerous). `ExtraManifestNodeTags` only adds attributes on `<manifest>`. `ExtraActivitySettings` is injected after the activity's intent-filter, so it cannot set activity attributes (4).
- Narrow overrides beat wide ones: re-enable Vulkan for exactly one GPU family with a project `DefaultDeviceProfiles.ini` cvar (`r.Android.DisableVulkanSupport=0`).
- Project `+MatchProfile` rules are appended AFTER the engine's rules, so when a chip already matches an engine profile, override that profile's cvars, not the rules.
- Re-read the matching rules whenever the engine version changes. Verify on a device with logcat: look for "Vulkan is disabled via console variable" and the quality-level lines.
- Play Console reports "N phones / M tablets dropped" after such a change; compare it with what you intended.

### 3.2 PSO precompile vs ad SDK start-up (ANR)
Observed on Mali GPUs: several `PSOPrecompilePool` threads compile Vulkan pipelines and serialise on one driver mutex; the game thread waits in `FlushRenderingCommands` during resume; a WebView-based ad SDK initialising at the same time adds lock contention. Both show up as ANRs.
- [ ] Cap the pool with a device-profile cvar: `r.pso.PrecompileThreadPoolSize=2` (lower on weak chips). It overrides the percent-of-threads variant (default 75 percent, i.e. 6 threads on 8 cores).
- [ ] Delay ad SDK initialisation until PSO precompile finishes: poll the engine's remaining-PSO counter (available to Blueprint as the number of precompiling PSOs remaining) every second; start ads only after it has been 0 for about 3 s (add a minimum delay, a hard cap, retry back-off and a maximum number of tries).
- [ ] Do not let the interstitial "loaded" callback call load again. An endless reload loop all session was a real bug found only by reading the graph.
- [ ] Reload ads from a slow timer, only if none is ready, no PSO work is pending, and enough time passed since the last request.
- [ ] Ad SDK manifest flags (for example the AdMob optimize-initialization and optimize-ad-loading meta-data) go in `ExtraApplicationSettings`.
- Useful cvars (names confirmed from the 5.8 binary; verify on your version): `r.pso.PrecompileThreadPoolSize`, `r.pso.PrecompileThreadPoolPercentOfHardwareThreads`, `r.Vulkan.ForcePSOSingleThreaded`, `r.PSOPrecaching`, `r.ShaderPipelineCache.*`.
- Without a bundled PSO cache, runtime precaching is the only compile source.

### 3.3 Per-chip device profile cvars
- Put chip-specific cvars in `Config/DefaultDeviceProfiles.ini` under the engine profile the chip matches (find the profile name in logcat or in the matching rules).
- Confirm a "fix" cvar is not already set by the engine: several suspected fixes were no-ops because `Engine/Config/Android/AndroidEngine.ini` already set them on all Android.
- Some crashes have no cvar fix (a hard-coded per-frame driver call such as the Vulkan command-pool trim in the RHI end-of-frame cleanup). The real fix then needs a source-built engine. Document the fallbacks: conservative cvars (accept an FPS cost), GLES fallback (impossible while ES 3.1 is not packaged), or Play exclusion.
- For each mitigation write down the confidence and the FPS cost, ship one, and check Play vitals 1-2 weeks later.
- UE 5.8 device-profile matching has no RAM source. Do not promise "low-RAM profile". Instead set a MARKER cvar in the low-tier profiles and read it from Blueprint.
- "Auto-lower graphics once per install on weak profiles": the Blueprint-callable function that returns the active device-profile name lives in a module that is not packaged for Android, so calling it BREAKS the Android cook (2.4). Use the marker cvar.

### 3.4 Play SoC exclusions
- Export the device catalog from Play Console (CSV with SoC, GPU, install base), classify the SoCs that cannot run your packaging, and add exclusion rules by SoC. Keep the classifier script and the excluded list in the repo notes.
- The manifest `uses-feature` filter only removes devices without a Vulkan driver at all. Devices that have a driver the engine refuses need explicit SoC exclusion.
- Re-evaluate after each engine upgrade. Exclusions are a Play Console step the developer performs.

### 3.5 Reading logcat
```
adb logcat -c
adb logcat -v threadtime -d > run.log          # dump after the problem
adb logcat -b crash -d
```
- Engine log on device (when `bPublicLogFiles=True`): `/sdcard/Android/data/<package>/files/UnrealGame/<Game>/<Game>/Saved/Logs/<Game>.log`.
- A Development APK (full logging) is the fastest way to find a Shipping-only crash: reproduce with it and read the `Script Stack:` lines to see which Blueprint function triggered an ensure.
- Strings in a Shipping `.so`: UE strings are UTF-16, so `tr -d '\0' < lib.so | grep -a "<phrase>"` finds dialog texts and cvar names (a way to find which engine path shows an unexplained dialog).
- Play vitals ANR clusters give stack traces with thread names; read the blocked threads first (game thread vs main thread vs a driver thread).

### 3.6 Symbolising
- UBT strips `libUnreal.so` before Gradle, so Gradle's `debugSymbolLevel` yields nothing. Make a symtab-only zip yourself per release and upload it to Play Console (native debug symbols). Keep it with the release archive.
- To resolve a pc address: use the unstripped `.so` (from `Intermediate/Android/...` or your symbols zip), then `llvm-addr2line -e <so> -f -C <addr>` (NDK `toolchains/llvm/prebuilt/.../bin`), or `llvm-objdump -d` and read the call target. A disassembly check proved one suspected "RHI" frame was really a driver trim-pool call every frame, and showed the proposed config fix was a no-op.
- Match the build id (`readelf -n <so>`) against the crash report, or you will symbolise the wrong build.
- Stack overflow in Blueprint: look for a repeating class. A spawn inside a corpse/death handler that spawns the same class gives kill/spawn recursion under mass kills (for example a nuke). After fixing one, scan ALL Blueprints for self-spawn patterns.

---

## 4. Foldables and tablets

- [ ] Test on an actual foldable (or a resizable emulator) folded, unfolded, and unfolding mid-game. Editor viewport framing cannot show this.
- [ ] Resizing: with `TargetSDKVersion >= 36`, UE 5.8 (UBT `UEDeployAndroid.cs`) emits `PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY=true` unconditionally. Android 16 normally ignores `resizeableActivity=false` on large screens, but that property opts back in, so the app runs in size-compat mode on unfold (keeps the folded aspect). Fix with the real settings `bAllowResizing=True` and `bSupportSizeChanges=True`. If the property itself must go, only a UPL `androidManifestUpdates` plugin can remove it (**verify** in your engine's UBT source).
- [ ] `ExtraActivitySettings` is injected AFTER `</intent-filter>`, inside the activity body. It cannot set attributes of the `<activity>` tag; a line like `android:resizeableActivity="true"` there is inert XML.
- [ ] To see what actually shipped read the packaged manifest under `Intermediate/Android/gradle/app/build/intermediates/packaged_manifests/release/...`.
- [ ] UI scale: with the default `ShortestSide` rule and curve, a near-square unfolded screen (short side about 1850 px) scales the UI about 1.7x while the viewport is narrower than a phone. Set `[/Script/Engine.UserInterfaceSettings] UIScaleRule=ScaleToFit` and `DesignScreenSize=(X=1920,Y=1080)`: 16:9-or-wider phones stay at exactly 1.00 (no regression), the foldable gets about 1.1.
- [ ] Camera framing: cameras with `MaintainXFOV` (or a fixed ortho width) fix horizontal coverage only; vertical coverage depends on aspect ratio. Compute coverage per aspect (phone about 2.17, unfolded about 1.16) and hand the developer numbers with the build.
- [ ] Layout switching: decide wide vs phone layout from the viewport aspect (for example aspect below 1.7 = wide). Re-check it from widget Tick, NOT a game timer. Game timers freeze while a tutorial or menu pauses the game, so the layout never switches. The same goes for anything else keyed to game timers.
- [ ] Do not let two systems drive one camera property (a per-tick zoom fought a level-Blueprint timer and pulsed). Find the existing owner first.
- [ ] Anchors: avoid fixed widths and offsets that assume one reference width. A stretch anchor with a right inset larger than the slate width goes negative and silently stops blocking input. Check every HUD element's anchors on both aspects.
- [ ] Test resize without repackaging: `adb shell am compat enable FORCE_RESIZE_APP <pkg>`; verify with `dumpsys activity activities | grep -A2 <pkg>` (want full-panel app bounds, no size-compat) and `dumpsys display | grep mDeviceState` (0 folded, 3 unfolded); then `adb shell am compat reset-all <pkg>`.
- [ ] Play flags edge-to-edge and resizability/orientation restrictions as recommendations; they are advisory, plan them separately from a release.
- [ ] Wide-layout test list: tutorial cards not off-screen, HUD not overlapping the clock/base-health row, FPS counter not inside another widget, loading and menu screens fit, screenshot both displays.
- [ ] The wide layout also matters on tablets and iPad (if supported).

---

## 5. Store features

### 5.1 Google Play Billing (one-time "remove ads" product)
Assumes a Marketplace billing plugin for Blueprints. Check which Billing Library it pins (Play requires a recent major version by a published deadline; the plugin's wrapper is compiled against one version, so do not force a newer one via UPL).
Play Console: a managed (non-consumable) product, product ID chosen once, price set, status Active. The developer or account holder activates it; agents do not touch Play Console.

Blueprint wiring traps (tool-driven or hand-made):
- [ ] Single-struct delegates (`const FPurchase&`, `const FProductDetails&`) cannot be bound from Blueprint event params that the editing tools create (they are not const-ref). Use the ARRAY variants of the delegates, then poll owned purchases after a buy.
- [ ] Enum parameters (billing response code) cannot be created by the tools, so failure delegates stay unbound. Treat "no success within N seconds" as failure.
- [ ] Create Event nodes must be CONNECTED before setting their function; an unbound Create Event breaks the Blueprint compile.
- [ ] DSL-style graph writers: a `switch int` makes pins 0..N-1 from the number of cases (add the zero case), and a switch ends the flow, so put follow-up logic in a separate function. A `select` node cannot feed enum pins: use if/else.
- [ ] Polling owned purchases: `QueryOwnedPurchases(INAPP)` on init, after a buy (poll every 1-2 s for a short while), on resume, and on Restore. Set `Owned` only from the owned list, never from the purchase-started callback.
- [ ] Acknowledge purchases (otherwise Play refunds after 3 days).
- [ ] Store the purchase time (ms since epoch) from the store when you need a "founder" cut-off; never from the device clock.
- [ ] Price: show the store's localised formatted price. Show launch-price copy with amounts only when the currency matches your launch price. Never show a fake strike-through "regular" price.
- [ ] A licence tester account and an internal/closed track are required to test a purchase; a sideloaded debug build cannot buy. Test: buy, pending, restore after reinstall, owned on a second device, refund.
- [ ] Gate the feature behind a config switch (`ShopEnabled`, default OFF) while it is unfinished and flip it in the release commit. Add the switch to the release checklist.
- [ ] Preview-only switches (for example forcing a prompt to show) must be flagged loudly in notes and reverted before the release build. Check them first in the runbook.
- [ ] The billing plugin's module `PlatformAllowList` may exclude `IOS`. A Blueprint that references its nodes then fails the iOS cook (7.3).
- [ ] After a stale testing-track release keeps a Play Console billing warning alive, check EVERY active track and retire old releases.

### 5.2 In-App Updates via UPL (Blueprint-only project)
- Why: a Blueprint-only project built with `-nocompile` has no module to attach Java/Gradle to. A UPL registered by an already-compiled plugin module (`AdditionalPropertiesForReceipt "AndroidPlugin"`) is read at packaging time, so editing it needs no C++ build.
- Put your Java inside a clearly marked block in that plugin's UPL (game-activity additions) and add `com.google.android.play:app-update` as a Gradle dependency.
- Keep in the repo: the block as its own file, an idempotent apply script (with `--remove`), and a copy of the original UPL.
- **A plugin (Marketplace) update or an engine Verify overwrites the UPL: re-run the apply script and diff.** Put it on the post-update checklist next to the `androidx.fragment` Gradle line.
- Behaviour that worked: flexible update only; check Play once per process; prompt at most once per day (SharedPreferences); act only when the game says the main menu is ready (a flag file the Blueprint writes while the menu is idle and deletes otherwise); after DOWNLOADED ask with a native dialog then `completeUpdate()`; all failures silent.
- The Java compiles for the first time during the build: expect a Gradle error on the first attempt and read it from the build log.
- Test needs a NEWER versionCode on the same track than the installed build: install N from the track, upload N+1, open the game to the main menu, expect Google's flexible dialog.
- In-App Updates has no iOS equivalent; the feature must be a no-op there (5.3).

### 5.3 Platform guards: Android-only UI never shows on iOS
- Gate every store-specific entry point with the platform name, written INLINE: `GetPlatformName() == "Android"`.
- Use ONE function as the single "this store exists here" switch (for example `ShowShopEntry = Enabled AND PlatformIsAndroid AND NOT Owned`) and drive every entry from it: main-menu button, settings rows, pause-menu row, nudge strip.
- **TRAP: a function made by the Blueprint editing tools is impure.** Used as a value inside another function, the compiler PRUNES the call and reads its output as the default (false). The warning is "was pruned because its Exec pin is not connected". The tools cannot mark a function pure, so inline the pure nodes instead of calling a helper. After every write grep the compile log for `pruned`. This bug would have hidden the feature on Android too; it was caught in the compile log before any build.
- Guard the starters too (billing init, update check, notification-permission ask), not only the UI.
- Platform-dependent text: store names in status messages ("Google Play" vs "App Store") chosen per platform.
- Add a test item to the iOS checklist: "no Android-only button, row, stamp or nudge appears anywhere on iOS".

---

## 6. The Android to iOS hand-off

The Windows session writes one hand-off document per release. The Mac session works from it without guessing.

### 6.1 Where it lives
`Docs/iOS_Handoff_<version>.md` in the repo, with a copy in the shared notes. Older hand-offs stay; the new one says which still apply.

### 6.2 Document format
```
---
type: handoff / platform: iOS / version / written (date) / author (Windows session)
---
# Hand-off: ship <game> <version> on iOS
Intro: who wrote it, who it is for, the "verify on Mac" convention
 (anything not checkable from Windows is an open question, not a fact).
Your job: one paragraph. What must be in the iOS release, what must be left out.

## 0. Read first
 - CLAUDE.md rules (build only on "build", no publishing without go-ahead, log AI work)
 - Confidentiality notes (internal design docs never go into store text)
 - Git: repo is local, LFS, secrets in ini -> NEVER push; do not commit on main
 - Status when written: Android version + versionCode, rollout plan, git HEAD,
   "make sure your checkout contains commit X before building"

## 1. Summary
 - What the release is (3-6 bullets)
 - Version numbers on iOS (marketing version, build-number rule, hand-typed in-game text decision)
 - Store "What's New", ready to paste (no mention of Android/Google/Play/ads/purchases)

## 2. Every change since <previous release>, feature by feature
 Table: # | Change | Where it lives (asset/function names) | iOS status | Switch or what to do
 iOS status key:
   OK         works as-is
   CHECK      should work; test on a device
   DISABLE    must not appear on iOS (name the switch)
   iOS-EQUIV  needs an iOS-specific counterpart (name the Blueprint hook points)
   N/A        Android-only; nothing reaches iOS
 Include the boundary commit and the commit count; list commits in an appendix.

## 3. Risks and known traps (most important first)
 Numbered; each with what to do and WHO decides. Typical: plugin not on iOS,
 UI not platform-guarded, clone has no iOS settings, asset-version mismatch,
 store-policy risks, clean cook required, LFS, binary-asset merge.

## 4. iOS test checklist (iPhone first, iPad if supported)
 One line per change: what to do on the device, what "pass" looks like.

## 5. Ready-to-paste instruction for the Mac session (see 6.4)

## 6+. Follow-up plans (for example the StoreKit hook-up table, 7.4)
## Appendix: commits since the previous release
```
Rules for the table:
- Name exact assets/functions/variables so the Mac session can grep them. Say which are `.uasset` (cannot be merged).
- For anything feature-flagged, name the switch, its default and who flips it.
- Say what was tested on Android and what was not ("purchase path tested on RC N, untouched since").
- Never include confidential design details (internal thresholds, difficulty tables) in anything that can reach store text. Public wording stays generic.
- Say which Android-only features stay out of iOS and which get an iOS counterpart later.

### 6.3 What the developer must decide (list these as questions in the hand-off)
- [ ] Rating prompt: keep a custom in-game rating screen on iOS, hide it, or use Apple's native review request (5.6.1, 7.5). Prepare a hide switch, default OFF, so the decision is one flag.
- [ ] Version text: keep the Android number in a hand-typed start-up string, or edit it per platform (an edit makes the asset differ between machines).
- [ ] Release order (Android first, then iOS once Android is approved) and rollout percentage.
- [ ] Which Android-only features come to iOS later (StoreKit purchase), and their price and launch schedule.
- [ ] Submit today or wait. The developer always clicks Submit for Review.
- [ ] App Store "What's New" wording.

### 6.4 The ready-to-paste Mac instruction (template)
> You are shipping `<game> <version>` on iOS. The developer says when to build and makes the final App Store submit click.
> 1. Get the project over the LAN: mount the share, `brew install git-lfs`, `git lfs install`, `git clone /Volumes/<share>/<repo> ~/Projects/<project>-<version>`, `git lfs pull`. Check `.uasset` files are real binaries (not pointer stubs); if LFS refuses the path use `git config lfs.standalonetransferagent lfs-standalone-file`. Write down `git log -1 --format='%h %s'`; it must include `<required commit>`. Never push this repo; do not commit on `main`.
> 2. Read the hand-off, the older hand-offs it names, `CLAUDE.md`, and your own iOS notes and memory.
> 3. Restore your iOS settings into the new clone (iOS sections of the config plus `Build/IOS/`). Set the marketing version to `<version>` and the build number above the last iOS build.
> 4. Before building, check the blockers in section 3 of the hand-off (billing plugin not on iOS; store UI platform guards). Run the iOS cook first and read the log for plugin errors. If it fails, stop and report the exact log lines with the options. Do not refactor Blueprints on your own.
> 5. Ask the developer the open decisions (rating prompt, version text).
> 6. If you must change a project file: separate local branch `ios-<version>`, commit there, report exactly which files and why. Binary changes cannot be merged; the Windows side makes the same change on `main` or takes your file before anyone edits that asset again.
> 7. When the developer says "build": clean iOS cook, package (Shipping, distribution signing), install on an iPhone, run the section 4 checklist (iPad too if supported).
> 8. Upload to App Store Connect, fill "What's New" from the hand-off, and stop before Submit for Review.
> 9. Keep a command log (every command, every setting restored and its source, every error and fix, timings) in a notes file and in memory.
> 10. Report: iOS version and build, git commit built, settings restored, billing-plugin cook result, each checklist item pass/fail, App Store Connect status.

### 6.5 Binary-asset merge rule
`.uasset` and `.umap` are binary and not mergeable.
- [ ] Never edit the same asset on both machines between syncs.
- [ ] The Mac session edits project files only on a local branch, minimally (ideally zero), and reports each file.
- [ ] The Windows side then makes the same change on `main`, or takes the Mac's file, BEFORE anyone edits that asset again.
- [ ] Prefer fixing platform issues in config (ini, or the `.uplugin` on the Mac's engine copy only) over editing Blueprints.
- [ ] Big Blueprints (tens of MB) are the worst case: do not refactor them on the Mac.
- [ ] After the release, sync the Mac's reported changes back and delete the branch.

---

## 7. iOS on the Mac

### 7.1 Preparing the fresh clone
- [ ] The Windows repo usually has almost no iOS settings (iPad flag, no `Build/IOS/`, no iOS ad-app ID). Restore from the previous Mac copy or notes: bundle ID, team, signing, provisioning, icons, launch screen, minimum iOS, iPad support, the iOS ad-SDK app ID, the ATT usage-description text (`NSUserTrackingUsageDescription`), `SKAdNetworkItems`, privacy-manifest entries.
- [ ] Touch only the iOS sections of config files. Everything else in the clone is newer and wins.
- [ ] Keep ads, ad unit IDs and consent set up the way the last shipped iOS version had them.
- [ ] Confirm every config change since the last iOS release was Android-only (so nothing silently changes on iOS) and list any that are not.

### 7.2 Version numbers
- Marketing version (`CFBundleShortVersionString`) = the release version (for example `2.3`).
- Build number (`CFBundleVersion`) = your own iOS convention, strictly higher than the last uploaded build. Do not reuse the Android versionCode.
- Hand-typed in-game version text: leave (shows the Android number) or edit on the Mac (the asset then differs). Ask the developer; default is leave.

### 7.3 Clean cook
- [ ] The first build of a release is a CLEAN cook (hundreds of files change between releases; iterative cooking carries damage forward).
- [ ] Run the cook first, before signing and archiving, and read the log for "imports type ... in a module that is not available on the platform".
- [ ] A plugin whose module `PlatformAllowList` lacks `IOS` (Android, Win64, Mac listed) breaks any Blueprint that references its nodes, structs or delegates. Options in order: (a) on the MAC ENGINE COPY ONLY add `"IOS"` to that module's allow-list in the `.uplugin` (non-Android code paths are often stubs; this may force a code build of the plugin that the installed engine has no binaries for - **verify**); (b) a Windows-side refactor that moves the calls into a Blueprint loaded only on Android; (c) look for an iOS build of the plugin or a StoreKit sister plugin. If the cook fails, stop and report; do not improvise.
- [ ] An engine-copy edit is wiped by a Launcher verify/reinstall: record it in the command log.

### 7.4 StoreKit for a one-time purchase in a Blueprint-driven game
Design on Windows first: every store call lives in one Blueprint behind a thin set of functions (`StoreInit`, `StoreQueryOwned`, `StorePrice`, `StoreBuy`, `StoreRestore`, `StoreOnOwned`), and the UI never calls the plugin directly. The Mac session then fills only the iOS branch. Mapping:

| Step | Android today | iOS to add |
|---|---|---|
| Init | init billing client, set non-consumable product IDs | branch on platform `IOS`: init UE's iOS in-app purchase interface / StoreKit, same product query |
| Query owned | query owned purchases, handle each | read current entitlements; for each matching transaction call the same handling |
| Price | set formatted-price variable from product details | set the same variable from the StoreKit localised price |
| Buy | launch billing flow, poll owned | purchase the product (App Store Connect non-consumable, same product ID) |
| Restore | re-query owned | StoreKit restore. Apple REQUIRES a visible Restore button |
| Owned | set `Owned=true` and a purchase-time variable, acknowledge | set the same variables; purchase time = StoreKit `originalPurchaseDate` in ms (drives any "founder" stamp) |
| Status texts | wording names the store | switch the wording to "App Store" on iOS |

- Last step: change the platform guard (5.3) from `Android` to `Android OR IOS`. The UI then appears unchanged.
- The developer must set up in App Store Connect (agents cannot): Paid Apps Agreement (bank and tax details), the non-consumable product (same product ID as Android if you can), price, localised name and description, review screenshot, sandbox tester.
- Test with a sandbox account: buy, cancel, restore after reinstall, airplane mode, ask-to-buy.
- Binary-asset rule (6.5): the Blueprints that hold these hooks are `.uasset`; coordinate before the Mac edits them.

### 7.5 Apple review notes
- **2.3.10**: no mention of Android, Google, Play or other platforms in screenshots, metadata, "What's New" or in-app text visible on iOS.
- **5.6.1** (review-prompt rules): do not run a custom "please rate us" flow that gates or incentivises ratings or imitates the system prompt. Use `SKStoreReviewController` (the native request) or a plain link to the review page; custom rating screens risk rejection. Provide a hide-on-iOS switch and let the developer decide.
- **1.2** (user-generated content): only if the game has UGC (chat, user names shown to others, uploads): it then needs filtering, reporting and blocking.
- **Ads and tracking**: ATT prompt and `NSUserTrackingUsageDescription` if ads use the IDFA; consent flow for the EEA; privacy nutrition labels must match the ad SDK.
- **In-app purchases**: restore button, correct product type, price from StoreKit, no mention of another store's payment.
- **Sensitive or political content** gets stricter review on Apple. If a previous version passed, do not change it between releases without telling the developer.
- Review notes: explain non-obvious flows (tutorial, locked features) and give a demo path; no test account is needed for games without login.
- Stop before "Submit for Review": the developer clicks it.

### 7.6 Command log for the next skill revision
Keep `~/<project>-ios-notes/<version>.md` and memory with: every command (cook, package, sign, archive, upload) and its working directory; every setting restored and where it came from; every error and its fix; timings per step; the git commit built; Xcode and macOS versions; the engine-copy edits made. The Windows session folds this back into this skill.

---

## 8. Release day runbook: the two-computer run-sheet

This is the order that worked when one game shipped on Android (Windows PC) and iOS (Mac mini) in one day, orchestrated by Claude agents. Stop at the first failure and tell the developer. Config values (package id, share names, addresses, URL patterns) live in `config.example.md`.

Legend: **W** = the Windows agent, **M** = the Mac agent, **D** = the developer, by hand. **Evidence** = what proves the step is done (paste it in the report, never just "done"). **Gate** = what must be true before the next step starts.

### Phase A: Android, on Windows

1. **Preflight (W).** Run section 12 top to end. Evidence: the ticked list; `git status` shows only known files. Gate: every item ticked; if not, stop.
2. **Drift check (W).** The Gradle `androidx.fragment` line is present, the In-App Updates UPL block is present, plugin versions unchanged since the last good build (5.2, 2.4). Evidence: grep/diff output. Gate: no drift, or drift fixed and noted.
3. **Build the release candidate (W)**, only after the developer says "build". Full `-cook -distribution` (clean if config or references changed) from a neutral directory, editor closed (2.2). Evidence: `BUILD SUCCESSFUL`, `ExitCode=0`, the `Cooked packages ... Total N` line. Gate: the log is read for `Error:` lines, not just the exit code.
4. **Verify the candidate (W, then the checker).** All of 2.3: size, package count, content asset pack, versionCode read back from the AAB, signer fingerprint, dependency versions, content freshness, the commit's file list (no secrets, no confidential numbers). Evidence: each reading quoted. Gate: the checker returns PASS. The developer is told "built" only after that.
5. **Archive (W).** Release AAB, APK, symbols and hashes to the archive folder, plus a text note (commit, versionCode, date) (2.8). Evidence: file list with sizes. Gate: the previous candidate is archived before the next build overwrites `Binaries/Android/`.
6. **Developer tests on two devices (D; W installs).** One ordinary phone, one foldable or tablet. W installs (2.6, 2.7) and confirms versionCode on the device. D plays: start-up text, intro, main menu, tutorial, store screens, a real match on default settings, pause and resume. Evidence: his written word per device ("works", or the bug). Gate: no word, no next step. A permission-classifier block on a release install means D runs or allows it (do not work around it).
7. **Fixes and re-builds (W, checker).** Every fix is a new versionCode, a full build and the full step-4 verification, and the previous candidate archived first. Expect three to four candidates (section 11). Evidence: a verification block per candidate. Gate: D approves the FINAL candidate on both devices in words; nothing changes after that approval except by starting again at step 3.
8. **Store product activated (manager, on D's word).** The in-app product must be Active before the release goes out. The manager session prepares the product page and clicks Activate only on the developer's explicit word (in the first release the manager did this; the developer's own clicks were the bundle drag and Submit). Evidence: the console shows Active, quoted back to him. Gate: done before step 9.
9. **Upload and Submit (D).** The manager session prepares the release form (name, notes ending with the devlog link, bundle slot) and opens the file manager with the bundle pre-selected (`explorer /select,<path>`). D drags the bundle in, sets a STAGED rollout percentage (not 100), and clicks Submit. The agent's upload tool cannot carry a 500+ MB bundle (about 10 MB cap). Evidence: the console shows the release "in review"; quick checks finish in minutes. Gate: agents touch nothing on the final submission screens (see Gates below).
10. **Review overnight.** Evidence: the console says the release is available on the store. Gate: D confirms in words. Nobody pre-announces anything while it is in review.
11. **Rollout to 100% (D).** Only on his word, after a look at vitals (crash rate, ANR rate). 100% removes the staged safety net, so say so when asking. Evidence: console shows 100%. Gate: the manager session starts watching vitals (step 22).

### Phase B: iOS, on the Mac, after Android is approved

12. **Open the LAN path (D with an admin shell; W prepares the commands).** Read-only share of the repo and of the billing-plugin folder; Wi-Fi profile set to Private; "File and Printer Sharing" inbound rules enabled; Windows kept awake (9.4). Evidence: the Mac mounts the share and lists the repo. Gate: the Windows agent is NOT building or editing during the transfer.
13. **Clone on the Mac (M).** To the Mac's INTERNAL disk, in a fresh folder (1.3). Allow about 40 minutes for 16 GB of LFS over Wi-Fi. Evidence: `git log -1` contains the commit the release was built from; `git lfs ls-files` shows real binaries (no 130-byte pointers); free disk is enough for the cook. Gate: M reports that evidence, signed.
14. **Branch and iOS-only pieces (M).** Branch `ios-<version>`, restore the iOS settings (7.1), set the marketing version, add the iOS-only code and wiring (store wrapper, native review prompt, platform wording). Set the build-number counter (see Gates). Evidence: commit list and exact file list on the branch. Gate: the Windows agent FREEZES every asset the Mac branch touches until the merge-back (section 13).
15. **Clean cook, dev build, iPhone test (M builds; D tests).** Run the cook first and read the log for plugin-not-available errors (7.3). Install the development build on the developer's iPhone. D runs the iOS checklist from the hand-off, including a sandbox purchase, restore after a clean reinstall, and "no Android-only UI anywhere". Evidence: pass or fail per checklist line, in his words. Gate: all pass.
16. **Shipping archive (M).** Distribution signing, clean cook. Evidence: version and build number read back from the archive, strictly higher than the last uploaded build; `get-task-allow` false. Gate: the checker (or the manager) verifies the build number BEFORE any upload.
17. **Upload (M), only on the developer's typed word** ("upload"). Evidence: the export succeeded; processing finished in the store dashboard; the build is attached to the version. Gate: M stops here and reports.
18. **Prepare the version page (M).** What's New (no mention of other platforms or stores, with the devlog link), build attached, in-app product attached with its review screenshot, release mode as the developer chose. Evidence: the version and product both show "Ready for Review". Gate: M touches nothing further.
19. **Submit for Review (D).** He clicks. No agent opens, expands or clicks anything on that screen while he is on it (see Gates). Evidence: his word, then status "Waiting for Review".
20. **Publish the website devlog (W prepares; D types "publish").** Staged as a draft earlier, previewed on phone and desktop width. It must be LIVE before Apple's review completes, because both stores' release notes link to it. Publishing emails subscribers, so it waits for his typed word in that conversation; "ok" to a status report is not enough. Evidence: an outside fetch of the final URL returns HTTP 200 and the expected headline; both stores' links resolve. Gate: live before the iOS release goes out.
21. **Sweep for wording the release made false (W, manager).** Search the store listings, privacy policy, older posts, the homepage, structured-data descriptions and excerpts for claims the release broke (for example "no in-app purchases"). Fix only on the developer's "go", one scope at a time (listing, site, homepage). Evidence: a regex sweep over every published post and page (content and excerpt) returns 0 hits, plus an outside fetch of the changed pages. Not covered by that sweep: SEO-plugin meta descriptions (a separate field) - check them separately.
22. **Next morning (manager, W).** Android vitals after the rollout; the Apple review result; the listing review result. Evidence: the numbers, quoted with their time. Gate: anything above the known baseline goes to the developer first thing.
23. **Merge-back (W, M).** Section 13. Update the notes ("where you left off", release log, command log) before closing the day.

### Gates that cost us time when skipped

- **Build-number counter:** the iOS counter file is keyed to the project FOLDER name. A fresh clone in a differently named folder restarts it at a low value and the store rejects the upload as not higher. Set it to the last shipped value before the Shipping archive and read the number back (7.2, section 11).
- **Disk:** an ExFAT external SSD breaks Xcode code signing. Clone and build on the Mac's internal disk.
- **Share:** the Windows share is invisible to the Mac until the Wi-Fi profile is Private and file sharing is on (both need an admin shell). Do this before the Mac needs it.
- **Big upload:** the AAB (about 580 MB) cannot be uploaded by the agent's browser tool. The developer drags it in; pre-select the file for him.
- **Final screens:** an agent opened a panel while the developer had Submit in front of him. The outcome was right, the attribution was not. Agents touch nothing on a final submission screen, not even to look.
- **Confidential numbers in commit messages:** commit logs travel with the repo (to the Mac, to hand-off appendices). Keep tuning numbers, thresholds and internal levels out of them.
- **Publishing emails subscribers:** the website post waits for the developer's typed word, never for an inferred "ok".
- **Devlog before Apple finishes:** if the post is not live when the iOS release goes out, a store link in the What's New is dead.

### Release naming convention (choose once, stick to it)
- Version name `MAJOR.MINOR` (for example `2.3`); versionCode a strictly increasing integer bumped on every build including test builds; the iOS build number tracked separately.
- Start-up string `v<version> (<versionCode>)` on candidates; the final release may show only `v<version>` (decide with the developer; done once so far).
- Archive folder `Release_<version>`, file `<game>-<version>-v<versionCode>.aab`.
- Git tag the release commit `v<version>-<versionCode>`.
- A release candidate is "RC = versionCode N". A testing-track build can be the same artifact later promoted to production (no rebuild).

### Reserved to the developer, by hand (agents never do these)
- Dragging the bundle into the Android console; Submit for review on BOTH stores; the rollout percentage and moving it up.
- Anything financial or account-level: agreements, bank and tax details, pricing, the service-fee programme, signing-key creation or rotation, spending money. (Creating and activating a store product is console work an agent may do on his explicit word, step 8.)
- The typed "publish" for the website (and any subscriber email or social post); store listing changes happen only on his "go".
- Accepting a browser local-network-access prompt or file-upload dialogs; pushing the repo anywhere.
- Permission-classifier blocks on release actions labelled "production deploy" (a `-distribution` build, `adb install` of a release APK): he runs or allows them. Do not work around the block.

---

## 9. Coordination with Claude agents

### 9.1 Roles
- **Manager session (Windows)**: the session that talks to the developer about the plan (the Mac session may also talk to him directly while he is at the Mac; it then relays what he said). Plans, decides who does what, relays decisions and quotes his words, owns the release checklist and the run-sheet (8), prepares store forms up to the final screen, watches vitals. Does not run long builds and runs no polling watchers.
- **Windows builder/editor session**: editor and MCP work, Blueprint edits, Android builds, device installs, commits, archive. Briefed to poll and continue while the editor loads, otherwise it ends its turn and waits for a nudge. Reports to the manager, not to the developer.
- **Mac session**: a peer, not a subagent, reached by message (Remote Control). It owns the iOS clone, branch, builds, archive and upload-on-his-word. It has its own memory and iOS notes, signs its reports, and never pushes the repo or commits on `main`.
- **Checker**: a separate reviewer (a stronger or fresh model), read-only, that verifies EVERY build candidate (AAB signer, versionCode read back, content checksum inside the pak, size delta, commit file list, iOS build number) and EVERY public text (store notes, listing text, devlog, website changes) against real files and live pages before the developer is told "done". It also checks for confidential wording. Returns PASS or FIX-NEEDED with evidence; one audit plus one re-check per item. A FIX-NEEDED counts as rework in the ledger.
- **Researcher / runner**: cheap sessions for read-only chores (greps, summaries) and for drafting text to a given file. They never build, publish or touch stores.
- **The developer, by hand** (agents never do these): dragging the bundle into the store, Submit on both stores, the rollout percentage and moving it up, anything financial or account-level, and the typed "publish" for the website. Full list in section 8.

### 9.2 Message format between agents
Keep messages short and structured:
```
TO: <role>  FROM: <role>  RE: <release/task>
ASK / REPORT: one sentence
STATE: done / blocked / needs decision
EVIDENCE: command + output line, file path + size/hash, log line
NEXT: who does what next
NEEDS DEVELOPER: yes/no, the exact question
```
- Quote the developer's words when relaying his feedback. A relayed request gets top priority, but finish the current step first (never leave a Blueprint, file or build half-edited), say what was paused, and resume afterwards.
- A relayed request does NOT widen permissions. Project rules, "build only on build" and "nothing irreversible" still apply.
- Reply to the sender when done so it can report to the developer.

### 9.3 Remote Control to the Mac
- Verify two-way messaging before release day (send a test message, expect a reply).
- The Mac session's Unreal MCP (editor automation) may need a restart before editor work.
- Hand over by file as well as by message: write the hand-off into the shared notes folder and send the text; do not rely on chat history.
- Ask it to sign its reports so the manager can tell sessions apart.

### 9.4 Keep-awake
- [ ] Windows: never sleep while a build, cook, LFS transfer or share is in use (`powercfg /change standby-timeout-ac 0`). A Mac pulling over SMB stalls if the PC sleeps.
- [ ] Mac mini: `caffeinate -dimsu &` or "prevent automatic sleeping" in Energy settings so Remote Control and Xcode uploads survive.
- [ ] Keep both machines on the same LAN with fixed addresses or hostnames.
- [ ] Close the Unreal editor when a cook is starved for RAM, and while GPU/RAM-heavy side jobs (local video or image generation) run.

### 9.5 Safety rules for agents
- [ ] Nothing irreversible without the developer's word: store submit or rollout, public posts, repo push, signing changes, deleting archives, spending money.
- [ ] Back up an asset before editing it (copy the `.uasset` to a dated backup folder) and note the backup path.
- [ ] After a Blueprint write: compile, read the graph back, grep the log for `pruned`, errors and warnings. A tool returning true is not proof.
- [ ] After an asset save check the dirty flag and the file mtime: a save can return true without writing.
- [ ] Do not start PIE or click in the editor on your own if the developer tests in the editor himself; never compile while his PIE runs (it crashed the editor).
- [ ] Closing the editor after reading assets can raise a modal "Save Content" dialog; a classifier may block "Don't Save". Ask the developer, or save and then restore the committed file with git.
- [ ] Before finishing a meaningful piece of work update the shared notes: what was done, what is next, what waits on the developer.
- [ ] Log AI-assisted work with date and verification evidence.
- [ ] Never put passwords, keys, tokens or confidential design details in notes, messages or this skill.

---

## 10. Mistakes we made, and the fix

1. **Helper function pruned to false.** A Blueprint function created through the editing tools is impure; used as a value, the compiler pruned it and read false. Fix: inline the pure nodes; grep the compile log for `pruned` after every write.
2. **Stale incremental OBB after a clock rollback.** Builds looked fresh but shipped old paks. Fix: compare inner pak timestamps (2.5); delete the staged OBB and generated files; keep the clock right.
3. **Cooking with an unavailable plugin/module.** A Blueprint node from a module not packaged for Android failed the cook. Fix: avoid the node or use a marker cvar; read the cook log; cook early after adding any plugin reference.
4. **A Blueprint referencing a plugin that is not on the target platform (iOS).** Predicted for a billing plugin whose allow-list excludes iOS. Fix: check the `.uplugin` allow-list before the iOS cook; engine-copy edit or Windows refactor; stop and report instead of improvising.
5. **Android-only UI visible on iOS.** The entry condition had no platform check. Fix: inline `GetPlatformName()` in the single entry-point function; add a test item.
6. **`-clean` silently did nothing.** The shell was inside `Saved/Cooked`; an empty APK was staged with ExitCode 0. Fix: neutral working directory; check size and package count.
7. **`-IgnoreCookErrors` masked failures.** Fix: do not use it for release builds; read the log.
8. **Distribution AAB without content.** A plain Shipping AAB carried only code. Fix: build with `-distribution` and check `obbassets` in `unzip -l`.
9. **Wrong device signature.** `adb install -r` of a debug build over a Play build failed with an incompatible-update error. Fix: uninstall, or use a release-signed APK.
10. **adb path mangling in Git Bash.** `/sdcard` became `C:/Program Files/Git/sdcard` and the OBB went nowhere while adb printed "pushed". Fix: `MSYS_NO_PATHCONV=1` and verify the file size.
11. **Second build died with "conflicting instance of AutomationTool".** A process check missed the running build and killing the cmd wrapper left it alive. Fix: PowerShell `Get-Process` by name; kill the `dotnet` child and `java` workers.
12. **Saving assets during a cook.** The save failed and the edit was lost. Fix: never save during a cook; compare mtimes with the cook start before a release build.
13. **Memory-starved cook.** With the editor open the cook stalled at 0 packages. Fix: close the editor; look for the MemoryMinFreePhysical warning.
14. **Vulkan-only packaging silently dropped devices** and caused a blocking-dialog ANR. Fix: decide the strategy explicitly (3.1), add manifest filters and SoC exclusions, re-read device-profile rules after each engine change.
15. **PSO precompile threads versus the ad SDK** caused resume ANRs. Fix: cap the pool; gate ad init on the remaining-PSO count.
16. **Ad reload loop.** The loaded callback called load again. Fix: no auto-reload; a slow keeper timer with guards.
17. **Config fix that was a no-op.** The engine already set those cvars. Fix: grep the engine's platform inis before shipping a "fix"; disassemble to confirm what a crash address calls.
18. **Non-scalability cvars in a scalability ini** caused an ensure that terminated the Android game. Fix: only scalability cvars there; read the device log's Script Stack.
19. **Foldable restricted resizability** from an engine-emitted manifest property, plus an inert `ExtraActivitySettings` attempt. Fix: real project settings; check the packaged manifest.
20. **UI 1.7x oversized on a foldable** from the default scale rule. Fix: `ScaleToFit` with a design screen size.
21. **Layout switch driven by a game timer** froze while a tutorial paused. Fix: widget Tick.
22. **Two systems fighting over one camera property.** Fix: find the existing owner (a level-Blueprint timer) before adding a tick.
23. **Level-Blueprint pin edits not reaching the `.umap`** on world-partition maps. Fix: add then remove a variable on the level Blueprint to force the dirty flag; verify by reloading.
24. **Widget width estimated from font metrics** under-shot the real render. Fix: capture a real render of the widget in the editor, or test on the device.
25. **Engine Gradle edit and plugin UPL edit wiped** by Launcher verify / plugin update. Fix: backups, an apply script and a post-update checklist; re-verify the AAB dependency versions.
26. **Preview-only switch left on.** Fix: a "revert before release" list, checked first in the runbook.
27. **Hand-typed in-game version text forgotten.** Fix: release checklist item.
28. **LFS pointer files on the Mac.** Fix: check sizes after the clone; standalone transfer agent; `safe.directory` for SMB.
29. **Secrets in a tracked ini.** Fix: never push; better, move secrets out of tracked config.
30. **Reinstalling an old artifact** made the developer doubt his own tuning. Fix: on "build" always rebuild before installing (and never build without the word).
31. **Billing delegates not bindable from tool-made Blueprint params** (non-const-ref). Fix: array delegate variants plus polling; connect Create Event nodes before setting the function.
32. **Tool quirks.** Adding a parameter to a function that already has callers breaks the callers' pins (no node refresh): use a variable instead. Setting an actor transform may reset omitted rotation or scale: pass all three. A multi-exec cast inside a loop can swallow the following flow: isolate casts or put them last. A DSL may hoist an impure call out of a while condition: use a for loop with break. A pure node read after a Set re-evaluates its getter: capture the value before the Set.
33. **Deleting an imported source file inside Content/** raises a modal "Delete Assets" dialog that blocks automation. Fix: keep import sources outside Content/ and import from a fresh folder.
34. **Compiling while the developer tested in PIE** crashed the editor. Fix: never compile during his PIE.
35. **A Play Console billing warning persisted** after the fix because an old build sat on a testing track. Fix: check every active track and retire old releases.
36. **Browser upload limits.** Extensions cap uploads (about 10 MB); a local server plus in-page fetch works only after the developer accepts a local-network-access prompt. Fix: he drags the AAB in.
37. **The permission classifier blocked release actions** (`-distribution` build, release `adb install`) as "production deploy". Fix: ask the developer to run or allow them; do not circumvent.

## 11. Release night, as it actually went (lessons, generic)

- **Expect three to four release candidates on the day, not one.** Ours: the final video swap, then a tutorial stall found on device, then a cosmetic version-text change, then a gameplay-tuning rule the developer insisted on before distribution. Each is a full cook (4-5 min) plus verification plus device install. Budget two hours for "the last build".
- **Verify every candidate the same way, by a separate reviewer**: version code read back from the package manifest, signer fingerprint vs the previous release, the changed asset's checksum inside the OBB/pak, size delta vs the previous package, the commit's file list (no config with secrets), timestamps (commit < build start < package). A 4-tool-call re-check per candidate is enough once the first full audit passed.
- **The editor session goes idle while waiting for the editor to load.** Brief it to poll and continue in one run, otherwise every step needs a nudge.
- **Keep confidential tuning numbers out of commit messages**, not just out of public docs; commit logs travel with the repo (to the other machine, to appendices).
- **Device installs:** add an allow rule for `adb *` in the agent runner's settings up front, or every install needs the developer's typed command.
- **Store upload:** a 500+ MB bundle is uploaded by the human (drag-and-drop); open the file manager with the file pre-selected (`explorer /select,<path>`) so it's one drag. Processing after upload takes minutes; the form's Next button stays disabled until the row appears.
- **Google Play flow (2026):** release form (name, notes inside `<en-US>…</en-US>`, bundle) → Next → review page (staged rollout %, countries) → Save → "Go to Publishing overview" → "Submit N changes for review" → confirm → automatic quick checks (~15 min; one observation so far) → sent to review. Activate in-app products on their own page first ("Activate" on the purchase option); the store rejects a build that references an inactive product only at runtime, not at upload.
- **Release notes:** plain text, links not clickable but readable; keep the devlog URL stable before the post exists (choose the slug first).
- **Website changes on release night:** background browser tabs may not render screenshots; use page text and element refs; a cPanel "Overwrite?" dialog may need a scripted click from a background tab.
- **Don't let a gameplay-tuning change ship untested on device**: the developer plays the release build on default settings before "go". Ours found a problem that no agent would have seen.
- **LAN transfer to the Mac, two traps:** (1) a Windows SMB share is invisible until the Wi-Fi profile is *Private* and the "File and Printer Sharing" inbound firewall rules are enabled (both need an admin shell); (2) clone on the Mac's internal disk - an ExFAT external drive breaks Xcode code signing (AppleDouble "detritus" errors). Share the engine's billing-plugin folder read-only too, so the Mac engine gets the same plugin version.
- **Transfer time:** a 16 GB LFS store over Wi-Fi took 40 minutes; git objects themselves took seconds. Plan the Mac clone before you need it, not on release day. Once the Mac branch touches shared binary assets (player/menu Blueprints), freeze those assets on the Windows side until the branch returns; any C++ added on the Mac must be copied back into the main repo.
- **iOS build number trap:** Unreal's `Build/IOS/<ProjectFolderName>.PackageVersionCounter` is keyed by the project *folder* name. A fresh clone in a differently named folder restarts CFBundleVersion from 0.1, and App Store Connect rejects the upload as not higher than the previous build. Set the counter to the last shipped value before the Shipping archive (we went 0.39 → 0.40).
- **The final submission screen is human-only.** The agent prepares everything up to it and then touches nothing on that page - not even "open the panel" - so there is never doubt about who pressed Submit. (Lesson from a case where the agent opened the submissions panel while the developer had the button in front of him; the outcome was right, the attribution was not.)

---

## 12. Pre-flight checklist

Tick every line before the first release build. A line you cannot tick stops the run.

- [ ] Version code bumped (strictly above the last one on every track) and version text bumped (`StoreVersion`, `VersionDisplayName`).
- [ ] Hand-typed in-game version text matches the release plan (candidate string vs final string decided).
- [ ] Preview-only switches reverted; feature switches in their release state (5.1).
- [ ] Release notes written, plain text, no internals, ending with the devlog link (slug chosen and fixed now).
- [ ] Store product created, priced and ready; activation scheduled with the developer (8, step 8).
- [ ] Store product id in the project equals the id in the store consoles (both stores).
- [ ] Confidential-wording check done on release notes, listing text, devlog, hand-off and the commit messages so far.
- [ ] Working tree clean except known files; no unsaved editor assets; clock correct.
- [ ] Drift check passed: Gradle line, In-App Updates UPL block, plugin versions (8, step 2).
- [ ] Release signing key present and its fingerprint matches the one registered in the store.
- [ ] Android-only UI is platform-guarded (5.3); the hand-off lists what stays off iOS.
- [ ] Frozen assets named: which assets the Mac branch will edit, frozen on Windows from clone time until merge-back.
- [ ] Branch plan written: `ios-<version>` on the Mac, nothing on `main`, who merges back and when.
- [ ] iOS build-number counter location known and its last shipped value written down (7.2).
- [ ] Windows share, firewall profile and keep-awake settings ready; Mac internal disk has room for the clone and the cook.
- [ ] Both machines reachable (test message to the Mac session answered); adb allow rule in place; device addresses known.
- [ ] Archive folder exists outside git; previous release archived.
- [ ] Devlog drafted and previewed on phone and desktop width, NOT published; staged as a draft where the site allows.
- [ ] List of site and listing claims the release may falsify (for example "no purchases") written for the later sweep (8, step 21).
- [ ] Developer's open decisions asked (6.3).

---

## 13. Merge-back of the iOS branch into the Windows source of truth

What happened on the first merge-back (2026-10-12, the day after the iOS release was submitted), and the procedure that worked on the second attempt. The first attempt stopped cleanly at step 3.

**Inputs the Mac hands over**
- A git bundle of the `ios-<version>` branch, a `MERGE-NOTES` file listing every changed file, and a zip of any plugin the Mac copied into the project. (In our run the bundle held 7 commits on top of the commit the clone was made from.)
- The LFS objects for every changed binary (step 3). The bundle does not carry them.
- A new C++ module the Mac added. Windows adopts it, so the project stops being Blueprint-only.

**Steps**
1. **Freeze and pre-flight (W).** Windows `main` has not touched the frozen assets since the clone (section 12). Commit any pending version bump first, so the merge base is a known commit. Evidence: `git status` clean, `git log -1`.
2. **Fetch the bundle (W)** into a scratch ref: `git fetch <bundle> ios-<version>:refs/heads/ios-<version>-incoming`. Diff the project copy of the billing plugin against the Marketplace copy (expect only the intended allow-list change). Evidence: commit count, file list, plugin diff.
3. **Get the LFS objects (M to W).** A git bundle carries LFS POINTERS only (130-byte text files). With the pointers alone the merge cannot produce real binaries: our first attempt found 7 LFS objects living only on the Mac and aborted with no merge commit. Fix: the Mac copies `.git/lfs/objects/<aa>/<bb>/<oid>` for every changed binary plus a manifest of `oid size path`. Windows installs them into its own `.git/lfs/objects` and verifies every file: `sha256sum` must equal the oid in the pointer. Evidence: N of N hashes match. (Alternative: an LFS-aware private remote; we have none, by design, see 1.4.)
4. **Make the hand-over frictionless.** An external SSD round-trip or a read-only share is too slow: the developer's feedback after the first attempt was that the whole point is lost if he has to run between computers. What worked: one writable LAN drop share on Windows, created once from an admin shell (`New-SmbShare` with change access for the Mac's user), plus the Mac session in a mode where it can copy without asking per file. Keep that share permanent and out of the repo; everything the Mac hands over (bundle, LFS objects, notes) goes there. Remember the Private-profile and firewall traps (section 11).
5. **Selective merge (W, builder).** Do not merge the branch wholesale. Adopt: `Source/**`, the `.uproject` (module declaration), the project copy of the billing plugin, and the cross-platform Blueprints the Mac edited (menu, in-game UI, pause menu, player). Keep Windows' version of every ini file and every map: the Mac's differ only in iOS needs, and binary maps cannot be merged. Leave `Config/IOS` and `Build/IOS` out of `main` (iOS-only; they stay on the iOS branch). Our merge touched 32 files. Evidence: the file list of the merge commit, reviewed against `MERGE-NOTES`. Anything the Mac changed that is NOT on the adopt list is reported, not silently dropped.
6. **Adopt the C++ module (W).** Install the C++ toolchain on the Windows machine (Visual Studio 2022 with the game-development workload and the Windows SDK; it was already present for us). Build the editor target (`Build.bat <Project>Editor Win64 Development ...`). Evidence: `Succeeded`, 0 errors (110 s in our case).
7. **Compile everything (W).** Run a Blueprint compile-all commandlet so nothing that referenced a changed function is left broken. Evidence: counts of clean, warning and failed Blueprints. Read the warnings; do not accept a number you did not look at (the auditor found 4 warnings where the builder reported 2).
8. **Prove with one Android build.** On the developer's "build": `-build` in RunUAT (2.2), then the full verification (2.3), install on a phone, and let the developer play it. This proved the Windows `main` is now the single source for both platforms. Evidence: the NDK compiled the module and the project plugin, cook 0 errors, versionCode read back from the device, his word.
9. **Read-only audit by a checker.** A separate reviewer re-checks: the merge file list against the adopt list, that excluded paths are absent, the LFS hashes, the build logs. It returns PASS or FIX-NEEDED with evidence; ours passed with notes (a warning count and a script path that still pointed at the Marketplace plugin instead of the project plugin).
10. **Housekeeping.** Lift the freeze. Update any script that edits the plugin (the In-App Updates apply script) to the project copy. Delete the hand-over folder once the branch is merged and archived (recycle, never delete blindly). Write the notes ("where you left off", command log).

**Rules that held**
- Windows `main` stays the source of truth; the Mac never commits on it.
- Binary assets edited on both sides cannot be merged by git: pick a side per file, by hand, and record which.
- Windows values win for config. iOS-only config and the iOS build folder are not merged.
- Camera or layout rows that exist only in a map the Mac could not edit (for example a tablet entry) are a hand job for the developer before the next iOS release; list them, do not guess.

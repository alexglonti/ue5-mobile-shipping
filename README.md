# ue5-mobile-shipping

**A Claude Code skill for shipping Unreal Engine 5 mobile games to Google Play and the App Store with a Windows PC, a Mac mini and Claude agents.**

One developer, two computers, two stores. The Windows PC builds Android, the Mac mini builds iOS, and Claude Code sessions on both machines do the legwork while you keep the decisions. This skill is the playbook those agents follow: a numbered two-computer run-sheet (who does each step, what evidence proves it, what gate comes next), a pre-flight checklist, the exact build, verify and install commands, a merge-back procedure for the iOS branch, and a list of 37 mistakes we already made so your agents do not repeat them.

By Aleksandre Glonti · Nomad Devs (nomaddevs.club)

Status: **v0.1 — drafted from one real two-platform release (Oct 2026) and one merge-back; will be revised after the next release.**
Tested on UE 5.8 (Launcher build). Items marked "verify" in the skill depend on your engine version.

## Who it is for

Solo developers and tiny teams who ship the same Unreal Engine 5 project to Android and iOS and use Claude Code for builds, Blueprint work and release chores. It assumes Git LFS, a Windows build machine and a Mac for Apple's toolchain. It is a checklist-and-commands playbook, not a tool: nothing runs until your agent reads it and you say "build".

## What is inside

The skill is one file, `skills/ue5-mobile-shipping/SKILL.md`, plus a config template.

1. Setup: one repo with Windows as the source of truth, Git LFS, `.gitignore`, LAN clone to the Mac, why never to push a repo with store secrets
2. Android build and device install: the RunUAT command, a verification list (size, package count, versionCode, signer, dependencies), cook gotchas, the stale-OBB trap, adb over Wi-Fi
3. Stability on Android: Vulkan and MinSDK decisions, PSO precompile versus ad SDK ANRs, per-chip cvars, reading logcat
4. Foldables and tablets
5. Store features: Play Billing for a one-time product, In-App Updates in a Blueprint project, platform guards
6. The Android-to-iOS hand-off document (format, decisions for the developer, a ready-to-paste Mac instruction)
7. iOS on the Mac: clean cook, version numbers, StoreKit for a one-time purchase, App Review notes
8. **The release-day run-sheet**: 23 steps across Android then iOS, each with owner, evidence and gate
9. Coordinating agents: roles, message format, Remote Control to the Mac, keep-awake, safety rules
10. Mistakes we made, and the fix (37 numbered items)
11. Release night as it actually went
12. Pre-flight checklist
13. **Merge-back** of the iOS branch into the Windows source of truth (LFS hand-over, selective merge, C++ module, proving build, audit)

`skills/ue5-mobile-shipping/config.example.md` lists every project-specific value (package id, share names, serials, paths) as a placeholder. Copy it, fill it in, keep the filled copy out of any public repo.

## 5-minute quick start

1. Install the skill folder (all projects, or one project):

   ```
   cp -r skills/ue5-mobile-shipping ~/.claude/skills/
   # or: cp -r skills/ue5-mobile-shipping <your-project>/.claude/skills/
   ```

   On Windows the user folder is `C:\Users\<you>\.claude\skills\`.
2. Copy the config template next to your project notes and fill in the placeholders:

   ```
   cp skills/ue5-mobile-shipping/config.example.md <your-notes>/ue5-mobile-shipping.config.md
   ```
3. Start a new Claude Code session on the Windows machine and try a read-only first command:

   > Use the ue5-mobile-shipping skill. Run the section 12 pre-flight checklist for my next release and tell me which lines I cannot tick yet. Do not build anything.

4. When you are ready to ship, say "build". Your agent follows sections 2.2 and 2.3 and reports evidence, not "done".

## A taste of the run-sheet (section 8)

Legend: W = Windows agent, M = Mac agent, D = the developer, by hand.

> 3. **Build the release candidate (W)**, only after the developer says "build". Full `-cook -distribution` from a neutral directory, editor closed. Evidence: `BUILD SUCCESSFUL`, `ExitCode=0`, the `Cooked packages ... Total N` line. Gate: the log is read for `Error:` lines, not just the exit code.
>
> 4. **Verify the candidate (W, then the checker).** Size, package count, content asset pack, versionCode read back from the AAB, signer fingerprint, dependency versions, content freshness. Evidence: each reading quoted. Gate: the checker returns PASS. The developer is told "built" only after that.
>
> 16. **Shipping archive (M).** Distribution signing, clean cook. Evidence: version and build number read back from the archive, strictly higher than the last uploaded build. Gate: verified BEFORE any upload.

## Human-only rules

Agents never do these. They prepare everything up to the screen and then touch nothing:

- Dragging the bundle into the store console, and "Submit for review" on both stores. On a final submission screen the agent does not even open a panel, so there is never doubt about who pressed what.
- The rollout percentage and moving it up.
- Anything financial or account-level: agreements, bank and tax details, pricing, signing-key creation or rotation, spending money.
- The typed "publish" for a website post or any subscriber email.
- Pushing the repo anywhere, and permission-classifier blocks on release actions: you run or allow them, agents do not work around them.

## Lessons you get for free (examples)

- A success exit code proves nothing; read the log and verify the artifact.
- A C++ project needs `-build` on every RunUAT run, or an old `.so` ships.
- A git bundle carries LFS pointers only; get the objects and check sha256 against the oid.
- The iOS build-number counter is keyed to the project folder name, so a fresh clone restarts it low and the store rejects the upload.

## Licence and credit

MIT. See `LICENSE`.

By Aleksandre Glonti · Nomad Devs (nomaddevs.club)

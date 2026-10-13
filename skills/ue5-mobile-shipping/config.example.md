# config.example.md

Every project-specific value the run-sheet (SKILL.md section 8) and the pre-flight checklist (section 12) refer to. Copy this file into your project (for example `Docs/release-config.md`), fill it in, and keep it OUT of any public place. Placeholders only here. Never write passwords, keys, tokens or keystore details into it.

## Identity
- `<game>`: the game's short name used in file names (archive folder, AAB name, branch names).
- `<package-id>`: the Android package name and iOS bundle id (reverse-domain form).
- `<store-product-id>`: the in-app product id, ideally identical on both stores.
- `<version>`: the marketing version, `MAJOR.MINOR` (for example `<MAJOR>.<MINOR>`).
- `<versionCode>`: the current Android version code (strictly increasing integer).
- `<ios-build-number>`: the last iOS build number uploaded to the store.

## Version counter
- `<version-counter-file>`: path of the iOS build-number counter file, relative to the project (the file name contains the project FOLDER name).
- `<project-folder-name>`: the folder name the counter is keyed to; a fresh clone in a different folder needs the counter reset to the last shipped value.
- `<last-shipped-build>`: the value to set the counter to before the Shipping archive.

## Machines and addresses
- `<windows-host>`: hostname of the Windows build machine.
- `<windows-address>`: its LAN address (use a fixed address or reserved lease).
- `<mac-host>`: hostname of the Mac mini.
- `<mac-address>`: its LAN address.
- `<repo-share-name>`: name of the read-only Windows share of the project repo.
- `<plugin-share-name>`: name of the read-only Windows share of the billing-plugin folder.
- `<repo-path-windows>`: absolute path of the repo on Windows.
- `<plugin-path-windows>`: absolute path of the billing-plugin folder inside the engine install.
- `<clone-path-mac>`: where the Mac clones (an INTERNAL disk folder, named per release).
- `<external-drive-path-mac>`: the external SSD used for merge kits and backups (never for building).
- `<wifi-interface-name>`: the Windows network interface whose profile must be set to Private.

## Devices
- `<phone-serial-or-address>`: the ordinary Android test phone (adb serial, or address and port for Wi-Fi debugging).
- `<foldable-serial-or-address>`: the foldable or tablet test device.
- `<iphone-name>`: the iPhone used for the iOS development build and test.
- `<adb-allow-rule>`: the allow rule for `adb *` set in the agent runner's settings.

## Branches and commits
- `<main-branch>`: the Windows source-of-truth branch.
- `<ios-branch>`: the Mac branch name, `ios-<version>`.
- `<release-commit>`: the commit the release candidate is built from (the Mac clone must contain it).
- `<frozen-assets>`: list of assets the Mac branch edits, frozen on Windows until merge-back.

## Paths and archive
- `<engine-path>`: the engine install used for packaging.
- `<archive-folder>`: where release AAB, APK, symbols and hashes go, outside git (`Release_<version>`).
- `<aab-path>`: where the packager writes the bundle (the file the developer drags).
- `<notes-folder>`: the shared notes folder (hand-off documents, command logs, "where you left off").
- `<handoff-doc>`: path of this release's iOS hand-off document in the repo.

## Website and listings
- `<devlog-url-pattern>`: the public devlog URL pattern with the slug fixed in advance, for example `https://<site>/update-<version-with-dash>/`.
- `<devlog-slug>`: the slug written into both stores' release notes.
- `<devlog-draft-id>`: the id of the staged (unpublished) post in the website system.
- `<publish-word>`: the exact word the developer types to authorise publishing (so "ok" is never mistaken for it).
- `<claims-to-sweep>`: the list of phrases the release makes false (for example a "no purchases" claim) for the post-release sweep.
- `<listing-short-text-ref>` and `<listing-full-text-ref>`: where the current store listing texts are kept.

## Store product and pricing
- `<launch-price>` and `<regular-price>`: prices as the stores will show them, with the date the price changes (`<price-change-date>`).
- `<founder-cutoff-date>`: the date after which a "founder" marker is no longer awarded, if the game has one.
- `<store-consoles>`: the two console URLs the developer works in (not stored here if they hold account ids).

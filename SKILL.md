# OpenWrt firmware build, debug, and release method

## Who this is for

- An AI that has never built OpenWrt firmware. Follow the phases in order, 0 to 9. Do not skip a phase. Each phase ends with a check that proves it is done; if the check fails, fix it before moving on.
- The method is proven on real hardware (Qualcomm IPQ50xx + NSS acceleration, Xiaomi CR880x), but the phases are general: they apply to any OpenWrt firmware project. Hardware and tree names in examples are from that project; substitute your own.
- Short imperative bullets, grouped by phase. A bullet is a rule, not a suggestion. When a bullet names a file or command, read that file or run that command; do not act from memory.

## Concepts (read this first)

- Base tree: the OpenWrt source tree you build from. Pick one that supports your SoC and the feature you need (for example, an NSS tree for Qualcomm offload). Pin it to one commit; never build from a moving branch.
- Feed: a collection of extra packages (LuCI, community packages). Each feed is pinned to a commit, like the base tree.
- Config: the build configuration. You write a short config (device, kernel, packages), then `make defconfig` resolves it into the full `.config`. The resolved `.config` is what actually built; a short config that only lists changes never proves what is in the image.
- Device tree (DTS): a source file that describes the hardware to the kernel: SoC, ports, flash, LEDs, buttons, and where the factory MAC addresses live. One board gets one DTS.
- Device profile: the named build target for one board (for example, `xiaomi_cr8806`). It selects the DTS and the package set. The profile name is internal; the user-facing label is the board version (M79A/M81), never the operator model.
- UCI: OpenWrt's config system. Settings live in `/etc/config/*` (network, wireless, system, firewall). You change them with `uci set` / `uci commit`, or by shipping a file.
- Overlay (`files/`): files copied into the image rootfs at build time. This is where presets, init scripts, and static configs live. A file in the overlay replaces the same path in the image.
- Image: the flashable file. `factory.ubi` is for a first install through the bootloader; `sysupgrade` is for upgrading a running device. Some devices only accept factory images.
- CI workflow: the GitHub Actions job that builds the image. Split it into a thin caller (dispatch inputs and permissions) and a shared workflow (the whole build), so adding a device means adding a caller, not editing the build.
- Release: a GitHub release carrying the firmware asset. Draft it, download it back, verify every digest, and only then publish.

## Phase 0: Identify the hardware

- Goal: you can name every fact the build depends on, before you choose a tree.
- Identify the board version, not the operator model. A CR8806, CR8808, and CR8809 can be the same hardware; the board version (M79A/M81) is the distinction that matters. A different board version can mean a different switch chip or front end, which makes firmware non-interchangeable.
- Record the SoC, the switch chip, the 2.4 GHz and 5 GHz front end modules (and whether each has a power amplifier), the RAM size, and the flash type and size.
- Record the bootloader and the flash method (for example, a web UI at a fixed IP that writes a factory image). Record whether the bootloader rewrites the partition layout when it flashes.
- Record the factory MAC addresses and where they live (for example, a calibration partition holding two independent MACs at fixed offsets, different OUIs, not base plus one). Verify the offsets in the device tree before you conclude the MAC assignment is wrong.
- Record the partition layout: stock or merged, and which parser reads it. A merged layout rewritten by a bootloader flash can defeat a parser that expects the stock layout.
- Check: you can state the SoC, switch, both front ends, RAM, flash, bootloader, flash method, MAC locations, and partition layout. If any is unknown, find it before Phase 1. Do not guess from the model name.

## Phase 1: Pick the base tree and set up the repo and CI

- Goal: a repo that can dispatch a build, with the base tree pinned.
- Pick a base tree that supports your SoC and the feature you need. Pin it to one commit (record the commit hash). Do not switch the kernel or base tree on a working build without a full re-verification plan: a base change means porting the device definition and partition layout and re-verifying wireless, offload, and IPv6 from scratch.
- Set up the repo with a thin caller workflow (dispatch inputs: device, firmware line, base commit) and a shared workflow (the whole build). Keep dispatch inputs and permissions on the caller.
- Pin every feed to a commit. Log every feed name with its exact commit for the release record; update feeds one at a time with a marker per feed, so a bulk update never hides which feed failed.
- Check: a dispatch starts a run and the run reaches the build step. If CI itself fails, fix CI before you touch the firmware.

## Phase 2: Configure the build

- Goal: a resolved config that contains everything the device needs, proven before the long build.
- Select the device profile, the kernel version, and the firmware line (for example, NSS firmware 12.5; the driver is built for the firmware shipped, never mix them). Add the packages the device needs.
- Run `make defconfig` and guard the resolved config after it. Check every required symbol and report every missing symbol at once, so one failure names all drops instead of dying on the first.
- Check device packages against the targetinfo generated by defconfig, and fail for every device package of the enabled profile that is not selected.
- Verify the wireless stack from the resolved config and from the built packages: the driver, the mac80211 and cfg80211 modules, the firmware package, and the wireless daemon. Accept any package file the tree actually builds instead of assuming one extension.
- Judge package presence from the resolved config, the job log compile lines, or a listing of the built rootfs. A diffconfig lists only nondefault symbols, so a default package never appears in it even when it is built.
- Check: the resolved `.config` contains every required symbol and every device package. If a symbol is missing, fix the config; do not work around it in the build.

## Phase 3: Add device support (device tree, partitions, MAC, ECC, board data)

- Goal: the kernel sees the hardware correctly. A mistake here bricks the boot, so verify each item from source.
- Write the DTS for the board: SoC, ports, flash, LEDs, buttons. Give a DSA user port its own MAC in the device tree with an nvmem cell, not through the board conduit: a port with an nvmem MAC cell takes that address at kernel probe time, independent of board.json and the overlay; a port that relies on the conduit inherits the default CPU port's MAC and only changes when the board file is regenerated.
- Read MAC addresses from the factory partition the way the device tree does. Verify the offsets in the device tree before you conclude the MAC assignment is wrong.
- Match the partition parser to the layout. A merged partition table rewritten by a bootloader flash operation can defeat a table parser that expects the stock layout, so a fixed partition listing is required for a merged layout while the stock layout keeps its parser.
- Check partition name case before you trust a hotplug script. Partition lookup in these scripts is a case sensitive exact match, so a script that asks for one case while the partition table uses another exits without creating any file and leaves the device with no wireless calibration data.
- Match NAND ECC to the bootloader chain. A kernel ECC strength that differs from what the bootloader chain writes makes the kernel unable to decode bootloader written pages, so the volume attach fails and the device does not boot.
- Fetch board data from a pinned source with a hash check. Board data files select entries by board id, so verify the board id your device tree sets against the entries in the fetched file before you build.
- Keep the front end in mind when you judge wireless strength. A receive only front end with no power amplifier gives weak transmit by hardware, and no config change fixes it.
- A WAN interface and the LAN bridge sharing one MAC is normal DSA behavior, not a bug. The two CPU ports still have distinct MACs from the factory data, and the bridge inherits one of them. Do not chase it as a connectivity cause.
- Check: the DTS compiles, the image carries the right partitions, and the MACs match the factory data. If the device does not boot, read the boot log and check ECC and partitions first.

## Phase 4: Add presets and overlays (where each change goes)

- Goal: the device boots with the right settings, and the settings survive the user's own edits.
- Put each change where it belongs. Hardware facts go in the DTS. Settings that are the same on every boot go in static files (`files/etc/config/*`, `files/etc/sysctl.conf`). Things that need the system up, or a config that is generated after boot, go in a START=99 init service. First-boot-only tweaks go in uci-defaults. A structural fix belongs in the DTS or a static file that is read on every boot, never in a first-boot script or a board file setting that only takes effect when the board file is regenerated.
- Bake WiFi as a preset with uci by radio name, never by device path. Set SSID, key, channel, and htmode on radio0/radio1 in a START=99 init service (wifi-preset), never in 99-custom: the wireless config is generated by the ieee80211 hotplug after uci-defaults run, so uci set on wireless in 99-custom is a silent no-op and the device boots with WiFi off. Let the paths come from the image's own wireless defaults; a baked whole-file wireless config with a wrong path breaks WiFi outright (extra network, unbound interface).
- Never bake a config that reads an address at first boot and keeps it. A firewall redirect written once from the LAN IP at boot silently breaks when the LAN IP changes later, because the one shot script never runs again. Prefer a redirect that resolves the address at runtime, or drop the redirect.
- Do not re-set a default the tree already ships. The shipped dnsmasq config already sets a cache size of 1000, so a preset that sets it again is dead weight. Check the tree default before you add a preset. dnsmasq already caches A, AAAA, CNAME, and SRV records with nothing checked; the cache arbitrary record option only adds record types on top of those defaults, so checking A or AAAA there is a no-op.
- First-boot scripts run in lexical order, so a later base-tree defaults script can undo your 99-custom change in the same boot. Verify the live state from a config backup, not from the repo file.
- A trigsoft U-Boot factory flash wipes the overlay, so first-boot scripts run again on a re-flash and no stale UCI section survives it. Do not tell the user to delete a leftover section after flashing; if the fresh first-boot state is wrong, fix the firmware and rebuild.
- Restrict the password page username list to root. The page lists every user from the password file, and the no password set banner can cover the root entry so saving fails validation silently. The username field is a text input, so typing the name manually still works when the list is trimmed.
- Verify a UCI option name in three places before you use it: the LuCI source (which key the form reads and writes), the script that generates the daemon config (which variable it reads), and the schema (whether the key is valid on that section). A name that exists somewhere else is not the same option: `multicast_to_unicast` is a bridge-port option, the wifi one is `multicast_to_unicast_all` (LuCI wireless.js, hostapd.sh). Setting the wrong name silently does nothing and LuCI shows it unset.
- Remember how LuCI applies settings. Save writes the config files without restarting the service, Save and Apply restarts it, and the same UCI section can be edited from more than one page. A setting that seems overwritten, conflicting, or slow to take effect is often a service that was never restarted, not a wrong value.
- LuCI Save drops UCI options that have no field in its form, even on a section you did not touch. `bss_transition` and `wnm_sleep_mode` have no LuCI field, so any WiFi edit wipes them. Options that must survive LuCI edits need a boot-time restore (a keep script), not a one-shot preset.
- IPv6 preset facts. Debug IPv6 by layers, in order: prefix delegation, then forwarding, then DNS. A client that has a global address but cannot use it has passed delegation and failed later, so do not re-diagnose delegation. Test forwarding with a numeric ping before you test name resolution, because a DNS failure cannot explain a failed ping.
- Know what ip6assign controls. It selects which prefix is delegated, not whether delegation happens. A LAN ip6assign of 64 leaves odhcpd no free /64, so the downstream router gets no PD prefix at all. Set it to /63 or shorter. LAN clients still get a /64 by RA/SLAAC either way.
- Changing LAN ip6assign takes effect without restarting odhcpd or the router. Do not tell the user a restart is required.
- Enable PD only on the WAN PPPoE interface. Keep PD off on wan6 and wan_6. The ISP address and the PD prefix attach to wan_6, but the switch lives on WAN PPPoE. PD needs both sides: upstream delegating and downstream requesting. The secondary LAN itself needs no PD, only RA to its clients.
- Check: run `dash -n` on every overlay shell script (uci-defaults, hotplug, init.d) before the long build, so a syntax error fails fast instead of after a flash. Confirm the preset sets the values you intend, and confirm each UCI option name in LuCI, the daemon script, and the schema.

## Phase 5: Build in CI

- Goal: a green run that produces a factory image.
- Trigger the build with the device, firmware line, and base commit. Trap errors in every run step and retry the slow steps: use a shell error trap that prints the failing line, retry clones and downloads with backoff, cap each download attempt with a timeout, and abort stalled transfers with a low speed limit.
- Key the compiler cache by toolchain and host compiler, not by device config. A config change must not bust the compiler cache. Keep the old config hash and the device prefix as restore fallbacks so a key change never cold starts a build.
- Keep three compile fallbacks. Try parallel, then serial, then serial verbose, and fail only after the verbose run names the failing package.
- Land failure evidence in the log. Every failure must leave enough evidence in the job log that no failure ever needs a rerun just to gather evidence.
- Check: the run is green and the factory image exists. If it fails, go to Phase 9; do not rebuild on a guess.

## Phase 6: Verify the image

- Goal: prove the image is flashable before it reaches a release.
- Verify the factory image structure in CI before it reaches artifacts or a release. Parse the image and check alignment, size limits, and the presence of kernel and rootfs volumes with data.
- Dissect the actual artifact first. Inspect the built image, the resolved config, or the device log before you guess a cause from a comparison or a summary file. Do not use post build binary dissection as a discovery method: a structural check of a built image can reject a broken image, but it never yields a cause. Cause finding is source reading only.
- Check: the structure verifier passes and the checksums are recorded. A failed structure check stops the release; it never ships.

## Phase 7: Release

- Goal: a published release the user can flash, with nothing unverified in it.
- Publish in two phases. Create a draft, download its assets, verify every digest, and only then publish. A release that is published first and verified after can only detect a bad release, never prevent one.
- Ship firmware only in releases. Put the flashable image in the release and nothing else. Move buildinfo files, the resolved config, checksums, and source ledgers to a workflow artifact for diagnosis, not to the release.
- Name the release firmware asset by board version (M79A/M81), never by operator model (xiaomi_cr8806/cr8808). The operator suffix is carrier-only and misleads; the board version is the distinction that matters. Rename before checksums are generated, so the digest file and the release asset carry the same name.
- Trim the release body to essential info only. Keep the device profile, the supported models, the kernel version, the firmware version, the base commit, the flash method, and the warnings. Drop feed versions, theme commits, build numbers, and attached file lists.
- Clean old caches and old runs after a successful publish. Keep the newest cache per type and drop runs older than two days, and never let a cleanup failure fail the job.
- Rebuild to add a feature. Do not build a self hosted package repository for kernel modules, because kernel modules carry a version magic that binds them to one exact kernel build. Userspace packages are kernel insensitive, but the project rule is still rebuild to add.
- Check: the published release carries the firmware asset under the board-version name, and the downloaded asset verifies against the recorded digest.

## Phase 8: Flash and test on the device

- Goal: the feature works on the hardware, not just in the build log.
- Treat a flash as a separate decision from a build. Building an image never authorizes flashing it, and an unverified fallback makes a flash high risk. State the fallback status every time a flash comes up.
- Flash by the device's own method (for example, a bootloader web UI with a factory image). Report build success and on device success separately. A build that compiles is not a build that works. Success means the feature runs on the hardware with the feature working.
- Use a known good firmware as a control case. If the same hardware, topology, and settings work on the control firmware and fail on yours, the fault is in your firmware, and every earlier conclusion drawn without that control is suspect.
- Check: the device boots, the feature works (wireless up, offload active, IPv6 working), and you have a fallback. If the feature does not work, go to Phase 9.

## Phase 9: Diagnose a failure and iterate

- Goal: name the cause from evidence, fix it once, and rebuild.
- Read the original source before you theorize. Open the config, the device tree source, the script, the driver code, or the partition table itself, and quote the exact lines that support or kill a theory.
- Read the failing link error, not the failing step name. A link error naming an undefined symbol in a specific object file points at the package that pulls that object in, not at the build system.
- A package that defaults on for one architecture can break the build on that architecture. PassWall's Rust Shadowsocks client defaults on for aarch64 and pulls a crate whose NEON assembly does not link against musl, so disable the client in the config when the project does not use it instead of patching the crate.
- Full Cone NAT and packet offload are treated as mutually exclusive upstream. Full Cone needs the nft fullcone module and a firewall patch, and the patch's own default config turns flow offloading off. Hardware offload through the NSS engine is a separate path from firewall flow offloading, so whether Full Cone and NSS coexist is unverified until tested on the device, not decidable from the config.
- Check NDP when addresses are present but unusable. A client can receive a global address from a Router Advertisement while Neighbor Solicitation and Advertisement are broken, so it cannot resolve its gateway's link-local address to a MAC. On a LAN client run `ip -6 neigh` and check the gateway's fe80:: entry: REACHABLE or STALE means NDP works, FAILED or INCOMPLETE means it does not. Confirm with a packet capture of ICMPv6 NS and NA on the LAN, and isolate the cause by disabling hardware offload and retesting.
- Prove the cause from an artifact before you rebuild. A rebuild on an unproven diagnosis burns a full build cycle and teaches nothing.
- Change one variable per round. Stacking unproven changes makes a success uninterpretable and a failure unbisectable. A batch is allowed only when the user explicitly orders it, and then a regression is bisected against the batch.
- Prefer the dumb thorough method first. Read the actual log, dissect the actual binary, and check the actual thing before you build a theory. A clever shortcut that still needs the dumb method to verify it saves nothing.
- Verify a patch against the real source file for the exact version the build uses. Check the package version in the Makefile, download that version's source, and dry-run the patch against it before pushing. A patch written from a search snippet or a different version's file will fail to apply: the regdb patch was written for a merged one-line CN entry while the build used wireless-regdb-2026.09.03 with two separate lines, and the build failed twice before the real file was checked.
- Verify text changes by exact diff. Show the changed lines, not the intent. Most waste in this kind of project traces to text errors, not hardware.
- Check: you can name the cause and quote the exact lines that prove it, before you rebuild. If you cannot, keep reading; do not rebuild to find out.

## Rules that apply in every phase

- Verify against the true source file before you act. Every push, every config change, every patch, every conclusion: download or read the real file and check it line by line first. A written-down lesson and a promise carry no weight; only demonstrated checking counts.
- Read the whole chain, including the part that says no. Read every layer a change touches (scripts, DTS, driver, first-boot). If a comment or driver says "don't do this," stop and understand why before proceeding. A "no" in the code outweighs every "yes."
- Do not present inference as fact. Label inference as inference. If it was not read from source, say so. When corrected, fix the record instead of repeating the claim.
- When one fault is found, check all code for the same class of problem without being reminded. Fix the whole class, not only the single instance.
- A manual fix is not a fix. A command run on the device to change a setting is not a fix. Structural problems are fixed in the firmware (DTS, static files, build-time patches). Post-flash manual steps are not fixes.
- Do not use the device to discover what source already says. On-device data confirms what source established; it does not discover it. Only ask for device output when source genuinely cannot determine it (hardware-dependent runtime values).
- When asked to find the cause and fix it, do not revert. A revert is a fallback, not the fix. Find the cause from source first; revert only if the user orders it.
- Batch only on an explicit order. Fix all known issues in one push and one build only when the user orders it, and warn that a regression in a batch needs bisecting.
- Keep the release page clean. Delete releases the user no longer needs, keep only firmware in future releases, and say plainly when a running build will still publish the old format.
- Let the user ask for build status. Do not schedule build monitoring. Act autonomously on a build failure only when the user has authorized that loop, and notify at each step without waiting for approval.

## Available connectors

- GitHub for repository files, workflow runs, and releases. Read `/opt/hatch/connectors/github/CONNECTOR.md` before you run its commands.

- Gmail for build failure mail that triggers the autonomous fix loop. Read `/opt/hatch/connectors/gmail/CONNECTOR.md` before you read its messages.

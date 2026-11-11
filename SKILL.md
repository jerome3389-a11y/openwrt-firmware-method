# OpenWrt firmware build, debug, and release method

## Prove the cause before you rebuild

- Read the original source before you theorize. Open the config, the device tree source, the script, the driver code, or the partition table itself, and quote the exact lines that support or kill a theory.

- Dissect the actual artifact first. Inspect the built image, the resolved config, or the device log before you guess a cause from a comparison or a summary file.

- Judge package presence from the resolved config, the job log compile lines, or a listing of the built rootfs. A diffconfig lists only nondefault symbols, so a default package never appears in it even when it is built.

- Prove the cause from an artifact before you rebuild. A rebuild on an unproven diagnosis burns a full build cycle and teaches nothing.

- Change one variable per round. Stacking unproven changes makes a success uninterpretable and a failure unbisectable. A batch is allowed only when the user explicitly orders it, and then a regression is bisected against the batch.

- Prefer the dumb thorough method first. Read the actual log, dissect the actual binary, and check the actual thing before you build a theory. A clever shortcut that still needs the dumb method to verify it saves nothing.

- Report build success and on device success separately. A build that compiles is not a build that works. Success means the feature runs on the hardware with the feature working.

- Land failure evidence in the log. Every failure must leave enough evidence in the job log that no failure ever needs a rerun just to gather evidence.

- Verify text changes by exact diff. Show the changed lines, not the intent. Most waste in this kind of project traces to text errors, not hardware.

- Do not use post build binary dissection as a discovery method. A structural check of a built image can reject a broken image, but it never yields a cause. Cause finding is source reading only.

- Debug IPv6 by layers, in order: prefix delegation, then forwarding, then DNS. A client that has a global address but cannot use it has passed delegation and failed later, so do not re-diagnose delegation. Test forwarding with a numeric ping before you test name resolution, because a DNS failure cannot explain a failed ping.

- Know what ip6assign controls. It selects which prefix is delegated, not whether delegation happens. A LAN ip6assign of 64 leaves odhcpd no free /64, so the downstream router gets no PD prefix at all. Set it to /63 or shorter: 63, 62, 61, 60, 59, 57 and /56 were all tested working on 2026-11-10, and the main LAN now runs /56. LAN clients still get a /64 by RA/SLAAC either way.

- Changing LAN ip6assign takes effect without restarting odhcpd or the router. Tested back and forth on 2026-11-10: the downstream delegated prefix changed on each edit with no restart. Do not tell the user a restart is required.

- Enable PD only on the WAN PPPoE interface. Keep PD off on wan6 and wan_6. The ISP address and the /56 PD attach to wan_6, but the switch lives on WAN PPPoE. PD needs both sides: upstream delegating and downstream requesting. The secondary LAN itself needs no PD, only RA to its clients.

- Use a known good firmware as a control case. If the same hardware, topology, and settings work on the control firmware and fail on yours, the fault is in your firmware, and every earlier conclusion drawn without that control is suspect.

- Check NDP when addresses are present but unusable. A client can receive a global address from a Router Advertisement while Neighbor Solicitation and Advertisement are broken, so it cannot resolve its gateway's link-local address to a MAC. On a LAN client run `ip -6 neigh` and check the gateway's fe80:: entry: REACHABLE or STALE means NDP works, FAILED or INCOMPLETE means it does not. Confirm with a packet capture of ICMPv6 NS and NA on the LAN, and isolate the cause by disabling hardware offload and retesting.

- Remember how LuCI applies settings. Save writes the config files without restarting the service, Save and Apply restarts it, and the same UCI section can be edited from more than one page. A setting that seems overwritten, conflicting, or slow to take effect is often a service that was never restarted, not a wrong value.

- Verify a UCI option name in three places before you use it: the LuCI source (which key the form reads and writes), the script that generates the daemon config (which variable it reads), and the schema (whether the key is valid on that section). A name that exists somewhere else is not the same option: `multicast_to_unicast` is a bridge-port option, the wifi one is `multicast_to_unicast_all` (LuCI wireless.js, hostapd.sh). Setting the wrong name silently does nothing and LuCI shows it unset.

- LuCI Save drops UCI options that have no field in its form, even on a section you did not touch. `bss_transition` and `wnm_sleep_mode` have no LuCI field, so any WiFi edit wipes them. Options that must survive LuCI edits need a boot-time restore (a keep script), not a one-shot preset.

## Build the image so a failure names itself

- Guard the resolved config after defconfig. Check every required symbol and report every missing symbol at once, so one failure names all drops instead of dying on the first.

- Check device packages against targetinfo in fail mode. Read the targetinfo generated by defconfig and fail for every device package of the enabled profile that is not selected.

- Verify the wireless stack from the resolved config and from the built packages. Check the driver, the mac80211 and cfg80211 modules, the firmware package, and the wireless daemon, and accept any package file the tree actually builds instead of assuming one extension.

- Verify the factory image structure in CI before it reaches artifacts or a release. Parse the image and check alignment, size limits, and the presence of kernel and rootfs volumes with data.

- Validate overlay shell scripts before the long build. Run `dash -n` on every uci defaults script and hotplug script the image will carry, so a syntax error fails fast instead of after a flash.

- Log one feed at a time with a marker per feed. A bulk feed update hides which feed failed. Record every feed name with its exact commit for the release record.

- Trap errors in every run step and retry the slow steps. Use a shell error trap that prints the failing line, retry clones and downloads with backoff, cap each download attempt with a timeout, and abort stalled transfers with a low speed limit.

- Keep three compile fallbacks. Try parallel, then serial, then serial verbose, and fail only after the verbose run names the failing package.

- Key the compiler cache by toolchain and host compiler, not by device config. A config change must not bust the compiler cache. Keep the old config hash and the device prefix as restore fallbacks so a key change never cold starts a build.

- Split the workflow into a thin caller and a shared build. Keep dispatch inputs and permissions on the caller and the full job on the shared workflow, so adding a device means adding a caller instead of editing the build.

## Release only what is verified

- Publish in two phases. Create a draft, download its assets, verify every digest, and only then publish. A release that is published first and verified after can only detect a bad release, never prevent one.

- Ship firmware only in releases. Put the flashable image in the release and nothing else. Move buildinfo files, the resolved config, checksums, and source ledgers to a workflow artifact for diagnosis, not to the release.

- Trim the release body to essential info only. Keep the device profile, the supported models, the kernel version, the firmware version, the base commit, the flash method, and the warnings. Drop feed versions, theme commits, build numbers, and attached file lists.

- Clean old caches and old runs after a successful publish. Keep the newest cache per type and drop runs older than two days, and never let a cleanup failure fail the job.

- Rebuild to add a feature. Do not build a self hosted package repository for kernel modules, because kernel modules carry a version magic that binds them to one exact kernel build. Userspace packages are kernel insensitive, but the project rule is still rebuild to add.

## Port to a device without breaking it

- Check partition name case before you trust a hotplug script. Partition lookup in these scripts is a case sensitive exact match, so a script that asks for one case while the partition table uses another exits without creating any file and leaves the device with no wireless calibration data.

- Match NAND ECC to the bootloader chain. A kernel ECC strength that differs from what the bootloader chain writes makes the kernel unable to decode bootloader written pages, so the volume attach fails and the device does not boot.

- Match the partition parser to the layout. A merged partition table rewritten by a bootloader flash operation can defeat a table parser that expects the stock layout, so a fixed partition listing is required for a merged layout while the stock layout keeps its parser.

- Fetch board data from a pinned source with a hash check. Board data files select entries by board id, so verify the board id your device tree sets against the entries in the fetched file before you build.

- Never flash across model, board revision, or board version. A different switch chip or a different front end module makes firmware non interchangeable even when the model name looks the same. Identify a board by its board version and its board revision, never by its operator suffix alone.

- Keep the front end in mind when you judge wireless strength. A receive only front end with no power amplifier gives weak transmit by hardware, and no config change fixes it.

- A WAN interface and the LAN bridge sharing one MAC is normal DSA behavior, not a bug. The two CPU ports still have distinct MACs from the factory data, and the bridge inherits one of them. Do not chase it as a connectivity cause.

- Read MAC addresses from the factory partition the way the device tree does. On these devices the calibration partition holds two independent factory MACs (different OUIs, not base plus one): offset 0 for the WAN CPU port and offset 6 for the LAN CPU port. Verify the offsets in the device tree before you conclude the MAC assignment is wrong.

- Never bake a config that reads an address at first boot and keeps it. A firewall redirect written once from the LAN IP at boot silently breaks when the LAN IP changes later, because the one shot script never runs again. Prefer a redirect that resolves the address at runtime, or drop the redirect.

- Bake WiFi as a preset with uci by radio name, never by device path. Set SSID, key, channel, and htmode on radio0/radio1 in 99-custom and let the paths come from the image's own wireless defaults; a baked whole-file wireless config with a wrong path breaks WiFi outright (extra network, unbound interface).

- Do not re-set a default the tree already ships. The shipped dnsmasq config already sets a cache size of 1000, so a preset that sets it again is dead weight. Check the tree default before you add a preset.

- First-boot scripts run in lexical order, so a later base-tree defaults script can undo your 99-custom change in the same boot. Verify the live state from a config backup, not from the repo file: the qca-nss-drv enable block in 99-custom was undone by nss-dwmac.defaults, which disables the driver init on purpose because nss-irq-affinity does the job after the plane is armed.

- A trigsoft U-Boot factory flash wipes the overlay, so first-boot scripts run again on a re-flash and no stale UCI section survives it. Do not tell the user to delete a leftover section after flashing; if the fresh first-boot state is wrong, fix the firmware and rebuild. Put structural fixes in the device tree or a static file that is read on every boot, never in a first-boot script or a board.json setting that only takes effect when the board file is regenerated.

- Give a DSA user port its own MAC in the device tree, not through the board conduit. A port with an nvmem MAC cell takes that address at kernel probe time, independent of board.json and the overlay; a port that relies on the conduit inherits the default CPU port's MAC and only changes when the board file is regenerated.

- dnsmasq already caches A, AAAA, CNAME, and SRV records with nothing checked. The cache arbitrary record option only adds record types on top of those defaults, so checking A or AAAA there is a no-op.

- Restrict the password page username list to root. The page lists every user from the password file, and the no password set banner can cover the root entry so saving fails validation silently. The username field is a text input, so typing the name manually still works when the list is trimmed.

## Diagnose a build failure from the log

- Read the failing link error, not the failing step name. A link error naming an undefined symbol in a specific object file points at the package that pulls that object in, not at the build system.

- A package that defaults on for one architecture can break the build on that architecture. PassWall's Rust Shadowsocks client defaults on for aarch64 and pulls a crate whose NEON assembly does not link against musl, so disable the client in the config when the project does not use it instead of patching the crate.

- Full Cone NAT and packet offload are treated as mutually exclusive upstream. Full Cone needs the nft fullcone module and a firewall patch, and the patch's own default config turns flow offloading off. Hardware offload through the NSS engine is a separate path from firewall flow offloading, so whether Full Cone and NSS coexist is unverified until tested on the device, not decidable from the config.

## Run a hardware project with the user

- Treat a flash as a separate decision from a build. Building an image never authorizes flashing it, and an unverified fallback makes a flash high risk. State the fallback status every time a flash comes up.

- Batch only on an explicit order. Fix all known issues in one push and one build only when the user orders it, and warn that a regression in a batch needs bisecting.

- Keep the release page clean. Delete releases the user no longer needs, keep only firmware in future releases, and say plainly when a running build will still publish the old format.

- Let the user ask for build status. Do not schedule build monitoring. Act autonomously on a build failure only when the user has authorized that loop, and notify at each step without waiting for approval.

- Do not switch the kernel or base tree on a working build without a full re-verification plan. A base change means porting the device definition and partition layout and re-verifying wireless, offload, and IPv6 from scratch, so defer it until the current stack is fully verified on the device.

## Available connectors

- GitHub for repository files, workflow runs, and releases. Read `/opt/hatch/connectors/github/CONNECTOR.md` before you run its commands.

- Gmail for build failure mail that triggers the autonomous fix loop. Read `/opt/hatch/connectors/gmail/CONNECTOR.md` before you read its messages.

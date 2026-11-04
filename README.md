# openwrt-firmware-method

A skill for Muse: a method for building, debugging, and releasing OpenWrt firmware with Qualcomm NSS acceleration in CI, proven on real hardware before any flash.

## What it covers

- Prove the cause before you rebuild (read the original, dissect the artifact, single variable, dumb thorough method first, hardware success, evidence in the log)
- Build so a failure names itself (defconfig guard, targetinfo check, wireless and factory verification, syntax checks, retries, cache schemes, reusable workflows)
- Release only what is verified (draft, verify digests, publish; firmware only releases; trimmed bodies)
- Port to a device without breaking it (partition name case, NAND ECC, partition parser, pinned board data, never flash across models)
- Run a hardware project with the user (flash separate from build, batch only on order, clean release page)

## Install

Clone this repository into your custom skills directory:

```
git clone https://github.com/jerome3389-a11y/openwrt-firmware-method.git ~/workspace/custom-skills/openwrt_firmware_method
```

Then Muse discovers it via skill search (for example, search for "OpenWrt firmware").

## Origin

Distilled from building IPQ50xx OpenWrt NSS firmware (CR880x) with GitHub Actions. The skill holds the method only, not the project log.

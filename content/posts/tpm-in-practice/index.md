---
title: "TPM 2.0 in practice - what it actually does on your machine"
date: 2026-11-19T00:00:00
draft: true
tags:
  - security
  - linux
  - windows
---

Most people learned about the TPM in 2021, when Windows 11 made TPM 2.0 a hard requirement and millions of perfectly good PCs were declared obsolete. For most users it's a checkbox in the BIOS that has to be enabled. But what does it actually do? And why does BitLocker sometimes ask for the recovery key after a firmware update, while unlocking the disk silently every other day?

## What a TPM is

A Trusted Platform Module is a small, separate cryptographic processor. It can generate and store keys, sign and decrypt data with them, and - the important part - it never lets the private keys out. The OS can ask the TPM to use a key, but can't read it.

It comes in a few forms:

- **Discrete TPM (dTPM)** - a separate chip on the motherboard, connected via the LPC or SPI bus. Common in business laptops.
- **Firmware TPM (fTPM)** - implemented in the firmware running in an isolated part of the CPU: Intel PTT or AMD fTPM. Most consumer PCs use this one.
- **Pluton** - Microsoft's security processor built directly into some recent AMD, Intel and Qualcomm CPUs, which can act as a TPM.
- **Virtual TPM** - emulated by a hypervisor for VMs (e.g. `swtpm` with QEMU, or Hyper-V), which is how you install Windows 11 in a VM without a bypass.

A smart card can also store keys and refuse to reveal them. What makes the TPM different is that it knows something about the state of the computer it's in.

## Measured boot and PCRs

The TPM has a set of **Platform Configuration Registers** (PCRs) - typically 24 of them per hash algorithm. A PCR can't be written directly. The only operation allowed is **extend**:

```
PCR_new = SHA256(PCR_old || measurement)
```

PCRs are reset to zero only when the machine reboots. During boot, each stage hashes the next one before running it and extends a PCR with the result: the firmware measures itself and the bootloader, the bootloader measures the kernel, and so on. Because of the chaining, the final value depends on every measurement and their order. There's no way to "un-extend" a PCR or set it to a chosen value - to get the same value, you need to boot exactly the same chain of components.

Note the difference from Secure Boot (see my [Secure Boot post](../secure-boot-broken/)). Secure Boot *enforces*: it refuses to run code that isn't signed. Measured boot only *records*: anything can run, but it leaves a trace in the PCRs that can't be erased.

Some of the commonly used PCRs:

| PCR | What's measured |
| --- | --- |
| 0 | Firmware (UEFI) code |
| 1 | Firmware configuration |
| 2 | Option ROMs (e.g. GPU firmware) |
| 4 | Boot loader code |
| 7 | Secure Boot state and the certificates used |
| 8-9 | GRUB: commands and files it loaded |
| 11 | Unified kernel image (systemd-stub) / BitLocker access control |

On Linux you can see them with `tpm2_pcrread` (from `tpm2-tools`) or `systemd-analyze pcrs`, and the full log of measurements in `/sys/kernel/security/tpm0/binary_bios_measurements`.

## Sealing - how your disk unlocks itself

This is where it gets useful. The TPM can **seal** a secret to a policy: "release this key only if PCR 7 has this exact value". If the boot chain is the expected one, the TPM hands over the key. If someone boots from a USB stick, disables Secure Boot or swaps the bootloader, the PCR values differ and the TPM refuses.

That's how BitLocker works by default on modern PCs: the disk encryption key is sealed to PCR 7 and 11, and Windows unlocks without asking for anything. The same can be done on Linux with LUKS (see my [disk encryption post](../disk-encryption/)):

```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/nvme0n1p3
```

and adding `tpm2-device=auto` to the options in `/etc/crypttab`. The existing password stays as a fallback key slot.

### Why the recovery key prompt?

Now the BitLocker mystery is easy to explain. A firmware update changes the firmware measurements. Updating the Secure Boot databases (e.g. the revocation list, DBX) changes PCR 7. The PCR values no longer match the policy, the TPM refuses to unseal the key, and you're asked for the recovery key. Windows tries to avoid it by suspending BitLocker before installing updates it knows about, but a firmware update from the vendor's tool, or a BIOS setting change, can still trigger it.

Choosing which PCRs to bind to is a trade-off. Bind to too few, and an attacker can change things without the TPM noticing. Bind to too many, and every kernel or firmware update locks you out. PCR 7 alone is a popular middle ground: it doesn't change with every kernel update, but it does change if someone turns off Secure Boot or enrols their own keys. Newer systemd versions also support signed PCR policies, where the expected PCR 11 values for a new kernel are signed by the distribution, so updates don't break unlocking.

## Is it secure?

TPM-only unlocking has one obvious weakness: the computer unlocks itself for anyone who turns it on. The disk is protected against being removed and read elsewhere, but if the attacker has the whole laptop, they get a booted system with a login screen. From there, the security depends on the OS and any bugs in it.

### Sniffing the bus

The second weakness applies to discrete TPMs. When the TPM releases the key, it sends it to the CPU over the motherboard bus - and by default, in plain text. This has been demonstrated many times; the most famous video is probably the 2024 one where the key was sniffed from a laptop's LPC bus with a Raspberry Pi Pico in well under a minute. Firmware TPMs don't have an external bus to sniff, but have had their own problems (e.g. faulTPM, a voltage glitching attack on AMD fTPM in 2023).

TPM 2.0 does support encrypted sessions between the OS and the TPM, which defeat passive sniffing. Some software uses them (systemd-cryptenroll does), BitLocker doesn't.

### TPM + PIN

The fix for both problems is the same: **TPM + PIN**. The key is only released when the PCRs match AND the correct PIN is provided. The TPM itself enforces anti-hammering - after a few wrong attempts it locks out for increasing periods - so even a short PIN is much harder to brute-force than a password protecting a LUKS header on a disk image.

```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 --tpm2-with-pin=yes /dev/nvme0n1p3
```

In BitLocker it's a group policy setting ("Require additional authentication at startup"), then `manage-bde -protectors -add C: -TPMAndPIN`.

## What else is it used for?

- **Windows Hello** keys and passkeys are stored in the TPM, which is why a PIN to unlock Windows can be safer than a password - it only works on that one device.
- **SSH and other keys.** `tpm2-pkcs11` lets you keep an SSH key in the TPM, so it can't be copied off the machine.
- **Remote attestation.** The TPM can sign the current PCR values with a key that's certified by the manufacturer, proving to a remote server what booted. Used in corporate device health checks, and controversially, in some anti-cheat systems and DRM.
- **Random numbers** - the TPM has a hardware RNG that Linux can use as an extra entropy source.

## Should you care?

If you use BitLocker or LUKS with TPM unlocking, it's worth knowing what's protecting you, and what isn't. My recommendation is TPM + PIN for laptops: almost as convenient, and it closes the two biggest holes. And if your machine ever asks for the recovery key after an update, now you know it's not broken - it's working as designed.

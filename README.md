# windows-endpoint-lab

A Windows 11 endpoint lab built on Proxmox. Right now it covers enrolling the VM in Microsoft Intune, a compliance policy, and BitLocker. The scope will grow as I use the VM for other work over its 90-day evaluation period.

**Status: work in progress.** Parts are finished and documented, parts are not. The status table below is the current state.

<small> *You might ask, "Brie, what led you to tackle this?" and I would tell you that it was a simple little job search with Claude asking me if I knew InTune explicitly because it was in a JD and it's on my resume. Well, it was coursework, GSEC I believe. So Claude said it could write up a tutorial, and here we are. This has been a LONG process over the past two days and we're not finished! #AlwaysBrieLearning* 

## What is here

- [docs/tutorial.md](docs/tutorial.md): a step-by-step walkthrough, from downloading the Windows 11 evaluation ISO to Conditional Access. Written so you can follow it from zero.
- [docs/writeup.md](docs/writeup.md): what I built, what broke, and what I learned.
- [screenshots/](screenshots/): evidence from the lab, with identifying details blurred. See the note in that folder.

## What you need

- A Proxmox VE host with room for a 100 GiB VM
- A Windows 11 Enterprise Evaluation ISO and the VirtIO driver ISO
- A new Microsoft tenant for the lab. The Intune and Entra ID P1 trials each run 30 days.

## Status

| Part | State |
| --- | --- |
| Windows 11 VM on Proxmox (UEFI, vTPM, VirtIO) | Done |
| Tenant, users, group, enrollment scope | Done |
| Entra join and Intune enrollment | Done |
| Compliance policy | Done |
| BitLocker policy, with recovery key in Entra ID | Done |
| Firewall policy and settings catalog profile | Next |
| App deployment, update ring | Not started |
| Conditional Access (report-only, then enforced) | Not started |
| Remote actions, rebuild, cleanup | Not started |

## The short version of what I learned

Intune reported the BitLocker policy as Succeeded for over an hour while the device was failing to apply it. A leftover installer ISO attached to the VM blocked BitLocker, and only the Windows event log showed the real error. The write-up has the full list.

## Before you commit

This repo is public and the lab uses a real tenant, so every commit is checked for identifiers that should not be published.

1. Install the hooks once per clone: `pip install pre-commit`, then `pre-commit install`. The hooks run on the machine where you run `git commit`, not on Proxmox or the VM.
2. The checks in `.pre-commit-config.yaml` block tenant domains, BitLocker recovery passwords, MAC addresses, GUIDs and public IPv6 addresses.
3. Run a secrets scanner too. [Betterleaks](https://github.com/betterleaks/betterleaks) is a candidate. Read its documentation for the current hook setup before adding it to the config.
4. Check every screenshot by eye. No scanner reads images.

## Notes

- This is a personal learning lab, not a production design.
- Everything here uses throwaway trial tenants. No real credentials, tenant names or device identifiers are published.
- No license is set yet.

# Intune Home Lab on Proxmox: Windows 11 from Zero

Goal: build a Windows 11 VM in Proxmox, enroll it in Microsoft Intune, and practice the core endpoint management and identity tasks (compliance, BitLocker, app deployment, updates, Conditional Access). Plan on a long weekend afternoon for Parts 1 to 4 and another session for Parts 5 to 12. The Intune trial runs 30 days, so build the VM first and start the trial clock only when you are ready for Part 5.

Written for Proxmox VE 8.1 or newer. Menu names in Microsoft's portals move around, so if a label is slightly different, search the portal for the setting name.

## What you need

- Proxmox host with about 8 GB RAM and 80 GB of disk to spare for this VM (4 GB RAM works but is slow).
- A PC to download ISOs and use the Proxmox web UI.
- A phone for the verification code and the Microsoft Authenticator app.
- A credit or debit card. Microsoft asks for a payment method to start the trial. Per Microsoft's sign-up page it is used for verification and not charged unless you buy something. Enter it yourself, and set a reminder to cancel the trial before day 30.
- A separate email address for the lab tenant (a new Microsoft account, not one tied to an existing work or school account).
- Your password manager (Vaultwarden) to store every lab credential.

## Part 1: Download the installation media

1. Windows 11 Enterprise Evaluation ISO (90 days, no product key): Microsoft Evaluation Center, https://www.microsoft.com/en-us/evalcenter/ , search for Windows 11 Enterprise. Choose ISO, 64-bit, English (United States) or your language.
2. VirtIO drivers ISO (stable): https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
3. In the Proxmox web UI, open your node, click your ISO storage (usually local), then ISO Images, then Upload. Upload both files. (Evaluation Center links often require a form, so downloading on your PC and uploading is easier than "Download from URL".)

## Part 2: Create the VM in Proxmox

Click Create VM (top right) and fill in each tab.

1. General: Name `win11-intune-lab`. Note the VM ID.
2. OS: ISO image = the Windows 11 ISO. Guest OS type = Microsoft Windows, version 11/2022. Check "Add additional drive for VirtIO drivers" and pick the virtio-win ISO.
3. System: Machine = q35. BIOS = OVMF (UEFI). Check Add EFI Disk and leave Pre-Enroll keys checked. SCSI Controller = VirtIO SCSI single. Check Add TPM, version v2.0, and pick a storage for the TPM state.
4. Disks: Bus/Device = SCSI. Size = 80 GB. Check Discard, SSD emulation and IO thread.
5. CPU: 1 socket, 4 cores, Type = host.
6. Memory: 8192 MB (minimum 4096).
7. Network: Bridge = vmbr0. Model = Intel E1000. (Windows has this driver built in, so setup and first boot have internet with no extra steps. You can switch to VirtIO later if you like.)
8. Confirm. Do not tick "Start after created" yet.

If Windows later complains about TPM or Secure Boot, the cause is almost always a missed setting on the System tab.

## Part 3: Install Windows 11

1. Select the VM, open Console, then click Start. Immediately click in the console and press a key when you see "Press any key to boot from CD or DVD". If you miss it, you land in the UEFI shell. Go to Reset and try again.
2. Language and keyboard, then Install now. If asked for a product key, choose "I don't have a product key". If asked for an edition, pick Windows 11 Enterprise Evaluation. Accept the license terms.
3. Choose Custom: Install Windows only.
4. The disk list will be empty. Click Load driver, Browse, open the VirtIO CD, then `vioscsi`, then `w11`, then `amd64`. Select "Red Hat VirtIO SCSI pass-through controller" and click Next. Your 80 GB disk appears. Select it and click Next.
5. The VM reboots a few times. Do not press a key at the "boot from CD" prompt again.
6. First-boot setup (OOBE): choose region and keyboard, name the PC `LAB-WIN11-01`, then at "How would you like to set up" choose Set up for work or school. On the sign-in screen click Sign-in options, then Domain join instead. Create a local account (store the password in Vaultwarden). Decline the optional privacy options.

You will join the device to Entra ID later from Settings, which is easier to retry than doing it during setup.

## Part 4: Finish the VM

1. Open File Explorer, open the VirtIO CD, and run `virtio-win-guest-tools.exe`. Install everything. This adds the network, balloon and guest agent drivers.
2. In Proxmox: VM, Options, QEMU Guest Agent, Edit, Enabled. Shut the VM down from Windows (Start, Power, Shut down), then start it again from Proxmox so the setting takes effect.
3. Run Windows Update until nothing is pending.
4. Confirm the security pieces Intune will look for. In an admin PowerShell window run `Get-Tpm` (TpmPresent and TpmReady should be True) and `Confirm-SecureBootUEFI` (should return True).
5. Take a Proxmox snapshot named `clean-install-before-enroll`. If your storage does not allow snapshots with a TPM, make a backup instead (VM, Backup, Backup now).

Optional but recommended: copy and paste does not work in the Proxmox console. To get it, turn on Remote Desktop in the VM (Settings, System, Remote Desktop), find the VM's IP address (Proxmox Summary tab, or `ipconfig`), and connect from your PC with Remote Desktop Connection (`mstsc`) as `.\labadmin`. After you join the VM to Entra ID, keep signing in over RDP with the local `labadmin` account.

Important: roll back to this snapshot only before you enroll. If you roll back after enrolling, Intune and Entra will hold a stale record for the old device. Delete the old device from both portals before enrolling again.

## Part 5: Start the Intune trial

1. Use a normal (not private) browser window. Open the Intune Plan 1 trial page from Microsoft's instructions: https://go.microsoft.com/fwlink/?linkid=2019088
2. Enter the lab email address, choose Set up account, and enter your details. Verify your phone number with the code.
3. Pick a username and a domain name (the part before `.onmicrosoft.com`). You cannot rename this later. Add the payment method.
4. Save the username from the confirmation email. The account that creates the tenant is its Global Administrator.
5. Sign in at https://intune.microsoft.com . Microsoft requires MFA for admin sign-in, so set up Microsoft Authenticator when asked.
6. Under Tenant administration, confirm the MDM authority is Microsoft Intune. It is normally set automatically.
7. Set a calendar reminder for day 25 to cancel the subscription (Part 13).

Correction from testing: the Intune Plan 1 trial only gave the tenant an Intune license. It did not include Microsoft Entra ID Premium, which automatic MDM enrollment and Conditional Access both require. In the admin center, open the Entra free trial catalog (the Mobility, Microsoft Intune page links to it) and start a free trial of Entra ID P1 (or P2, which includes P1). Then assign that license to your test user and admin account in the Microsoft 365 admin center under Billing, Licenses. Note a second trial end date for your reminder.

## Part 6: Users, licenses, groups and enrollment settings

1. Create a test user. In the Microsoft 365 admin center (https://admin.microsoft.com), go to Users, Active users, Add a user. Name it `Lab User`. Set Usage location to United States, then assign the trial's EMS/Intune license in Licenses and apps. Store the password in Vaultwarden.
2. Create a break-glass admin. Add a second cloud-only user `bg-admin`, give it the Global Administrator role, use a very long random password, and store it in Vaultwarden. You will exclude it from Conditional Access in Part 12 so a bad policy can never lock you out.
3. Create a group. In https://entra.microsoft.com go to Groups, New group. Type = Security, name `Lab-Users`, membership Assigned, add `Lab User`.
4. Check automatic enrollment. In Entra, go to Mobility (or Entra ID, Mobility), Microsoft Intune, and set MDM user scope to All (or Some and pick `Lab-Users`).
5. Check join permissions. In Entra, Devices, Device settings, "Users may join devices to Microsoft Entra" should be All (or include `Lab-Users`).

## Part 7: Enroll the VM

1. On the VM, open Settings, Accounts, Access work or school, Connect.
2. Click "Join this device to Microsoft Entra ID" (not the main sign-in box), sign in as `Lab User`, finish MFA setup with Authenticator when prompted, and accept.
3. Restart when asked and sign back in as the work account (the sign-in screen offers `Lab User`).
4. Verify on the VM: Settings, Accounts, Access work or school, click the connected account, Info. You should see a sync status.
5. Verify in the portal: Intune admin center, Devices, All devices. `LAB-WIN11-01` should appear, owned by `Lab User`. In Entra, Devices, All devices, the Join type should say Microsoft Entra joined, and MDM should say Microsoft Intune.

This is a good moment for a screenshot and a snapshot named `enrolled`.

## Part 8: Compliance policy

1. Intune, Devices, Compliance, Create policy, Platform = Windows 10 and later.
2. Name it `Lab-Windows-Compliance`. Suggested settings: Device Health, require BitLocker, Secure Boot and code integrity. Device Properties, minimum OS version `10.0.22000`. System Security, require firewall, antivirus, antispyware and Defender real-time protection.
3. Actions for noncompliance: Mark device noncompliant, immediately.
4. Assign to `Lab-Users`.
5. On the VM, sync (Part 7, step 4, Sync button) and wait a few minutes. In Intune, check Devices, the device, Device compliance.

Expect the compliance result to take 15 to 60 minutes to appear, and the dashboard tiles lag even longer. Check the policy's own Monitor page and Per-setting status instead of the dashboard. Note that BitLocker may already show as compliant: Windows 11 24H2 can start device encryption on its own, leaving the drive encrypted with no key protectors and Protection Status Off (check with `manage-bde -status C:`). Intune only checks that the drive is encrypted, so this passes even though the data is not really protected. The BitLocker policy in Part 9 fixes that.

## Part 9: BitLocker and configuration profiles

1. BitLocker: Intune, Endpoint security, Disk encryption, Create policy, Windows, BitLocker profile. Set Require device encryption = Enabled and Allow warning for other disk encryption = Disabled (this allows silent encryption on a TPM device). Under Operating System Drives, set Require additional authentication at startup = Enabled, Configure TPM startup = Require TPM, and set the TPM startup key, PIN, and key and PIN options to Do not allow (a startup PIN would lock you out of Remote Desktop at boot). Set Allow BitLocker without a compatible TPM to False. In the recovery section, set Allow data recovery agent to False, Omit recovery options from the BitLocker setup wizard to True, and leave Save BitLocker recovery information to AD DS for operating system drives on True (Intune uses the old AD DS label, but on an Entra joined device this backs up to Entra ID). Set Do not enable BitLocker until recovery information is stored to AD DS to True. Assign to `Lab-Users`. If the policy is slow to arrive, remember that the endpoint security editor finds All devices through its search box, and that a policy cannot mix specific groups with All devices.

   **Before this works, detach both ISOs.** BitLocker refuses to add protectors while bootable media is present. In Proxmox, open the VM, Hardware, edit each CD/DVD drive, and choose Do not use any media (the Windows 11 ISO and the VirtIO ISO). In the BitLocker Management event log on the VM the failure reads "Failed to enable Silent Encryption. TPM is not available" followed by "BitLocker Drive Encryption detected bootable media (CD or DVD) in the computer", event ID 853. The first half of that message is misleading. Intune still reports the policy as Succeeded while this error repeats, so the portal and the device disagree. Detach the ISOs before you create the BitLocker policy and you will skip this problem.
2. Sync the VM. Encryption can take several minutes to start. Check Settings, Privacy and security, Device encryption, or run `manage-bde -status C:` in an admin PowerShell window. Success looks like Protection Status: Protection On, with Numerical Password and TPM listed under Key Protectors. To see why it has not worked, run `Get-WinEvent -LogName "Microsoft-Windows-BitLocker/BitLocker Management" -MaxEvents 20 | Format-List TimeCreated, Id, Message`. Event 845 means the recovery key was backed up to Entra ID.
3. In Intune, open the device and look for Recovery keys. Seeing the key stored there is a good thing to screenshot (blur the key itself).
4. Firewall: Endpoint security, Firewall, Create policy, Windows, Microsoft Defender Firewall. Enable the Domain, Private and Public profiles. Assign to `Lab-Users`.
5. Settings catalog profile: Devices, Windows, Configuration, Create, New policy, Windows 10 and later, Settings catalog. Name it `Lab-Settings`. Use Add settings to search for and enable:
   - Interactive Logon: Machine Inactivity Limit (for example 600 seconds).
   - Turn off Autoplay.
   - Interactive Logon: Message Text For Users Attempting To Log On (a short lab banner).
   Assign to `Lab-Users`.
6. Sync and confirm each profile shows Succeeded under the device's Device configuration. On the VM, the banner and inactivity lock are easy to see in action.
7. Back in the compliance policy view, the device should now flip to Compliant once BitLocker finishes and a sync completes.

## Part 10: Deploy an app

1. Intune, Apps, Windows, Add, App type = Microsoft Store app (new). Search for 7-Zip (or Notepad++), select it, and add it.
2. Assignments: Required, group `Lab-Users`. Leave the install context as System unless the app requires otherwise.
3. Sync the VM and wait. Confirm the app appears on the VM and shows Installed in Intune under Apps, Monitor.
4. Stretch goal: package a small installer as a Win32 app with the Win32 Content Prep Tool (`.intunewin`) and deploy that. It is more work and much closer to real-world deployments.

## Part 11: Windows Update ring

1. Intune, Devices, Windows, Windows updates, Update rings, Create profile.
2. For a lab, set quality update deferral 0 days and feature update deferral 0 days. Assign to `Lab-Users`.
3. Check the device's update status after a sync. Note in your write-up what a real organization would set differently (pilot rings, deferrals, maintenance windows).

## Part 12: Conditional Access (the main event)

Warning: do this carefully and in order. A bad policy can lock out admins.

1. Security defaults first. New tenants have security defaults on, and you cannot use Conditional Access with them. In Entra, Overview, Properties, Manage security defaults, set to Disabled. Create the admin MFA policy in step 2 right away so admins stay protected.
2. Create `CA-Admins-Require-MFA`: Users = Directory roles, Global Administrator, and exclude `bg-admin`. Target resources = All cloud apps. Grant = Require multifactor authentication. Turn it On. The break-glass account stays excluded from every policy so a mistake can never lock you out of the tenant.
3. Create the main policy `CA-LabUsers-Compliant-Device-MFA`. Users, Include = `Lab-Users`, Exclude = `bg-admin` and your admin account. Target resources = Office 365 (not All cloud apps, which is riskier). Grant = Require multifactor authentication and Require device to be marked as compliant, with "Require all the selected controls" chosen.
4. Set Enable policy = Report-only and create it.
5. Open https://www.office.com on the VM and sign in as `Lab User`. In Entra, Monitoring, Sign-in logs, open that sign-in and check the Report-only tab to see the policy would have been satisfied.
6. Switch the policy to On.
7. Prove it works. On your normal PC, open a private browser window, go to https://www.office.com and sign in as `Lab User`. It should be blocked or told the device must be managed. On the VM, access works.
8. Prove it reacts to compliance. In the compliance policy, temporarily raise the minimum OS version above what Windows reports (for example `10.0.99999`). Sync the VM, wait for it to turn Noncompliant, then try office.com on the VM and confirm it is blocked. Put the setting back, sync, and confirm access returns.

## Part 13: Remote actions, rebuild and cleanup

1. Practice the device actions under Devices, the device: Sync, Restart, Remote lock, Collect diagnostics, then Retire (removes company data and management) and re-enroll the VM to repeat Part 7.
2. Before the trial ends, take a final backup of the VM in Proxmox if you want to keep the Windows install.
3. Retire and delete the device from Intune, delete it from Entra, then cancel the subscription: Microsoft 365 admin center, Billing, Your products, select the trial, Cancel subscription. Confirm no payment method is attached to any paid plan.
4. The evaluation Windows copy expires after 90 days, so the VM is also a fine place to repeat this lab later in a fresh tenant.

## Troubleshooting

- Setup shows no disks: you skipped loading the VirtIO SCSI driver (Part 3, step 4).
- Windows 11 says the PC cannot run it: the VM is missing UEFI (OVMF) or the TPM. Recreate the VM with the System tab settings in Part 2.
- Join fails with an error about the user not being allowed: check Entra Device settings (Part 6, step 5), and that the user has the Intune license.
- Device shows in Entra but not in Intune: check the MDM user scope (Part 6, step 4) and the license, then remove the work account on the VM and join again.
- Settings never apply: use the Sync button on the VM and in the portal, and give policies up to 15 to 30 minutes. In the portal, the device's Device configuration page shows errors per setting.
- BitLocker policy shows Succeeded but `manage-bde -status C:` says Protection Off with no key protectors: read the BitLocker Management event log on the VM. Event 853 about bootable media means an ISO is still attached to a virtual CD/DVD drive. Set the drives to Do not use any media in Proxmox, sync, and wait a few minutes.
- Policy assigned but device compliance page is empty for an hour or more: check the policy's own Monitor page and Per-setting status, which update before the dashboard and the device page. Switching the assignment from `Lab-Users` to All devices does not hurt on a one-device lab.
- Locked out by Conditional Access: sign in with `bg-admin`, set the policy to Off, and fix it.

## What to capture and write up

Take screenshots at each major step: VM hardware tab, the Windows install, the enrolled device in Intune, the compliance policy results, the BitLocker recovery key location (key blurred), the app installed, report-only results, and the blocked and allowed sign-ins. Write up what broke and how you fixed it, because that is the part interviewers care about.

For your public repo, blur or remove the tenant domain, user names, device names and IDs, and any recovery keys before posting.

If you complete it, a resume line you can truthfully use under Training & Independent Projects (adjust to what you actually did): "Built a Windows 11 lab on Proxmox managed by Microsoft Intune, including Entra ID join, compliance policies, BitLocker, app deployment, and a Conditional Access policy requiring MFA and a compliant device."

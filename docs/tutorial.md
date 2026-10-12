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
2. Name it `Lab-Windows-Compliance`. Suggested settings: Device Health, require BitLocker, Secure Boot and code integrity. Device Properties, minimum OS version `10.0.19041`. System Security, require firewall, antivirus, antispyware and Defender real-time protection.
3. Actions for noncompliance: Mark device noncompliant, immediately.
4. Assign to `Lab-Users`.
5. On the VM, sync (Part 7, step 4, Sync button) and wait a few minutes. In Intune, check Devices, the device, Device compliance.

Expect the compliance result to take 15 to 60 minutes to appear, and the dashboard tiles lag even longer. Check the policy's own Monitor page and Per-setting status instead of the dashboard. Note that BitLocker may already show as compliant: Windows 11 24H2 can start device encryption on its own, leaving the drive encrypted with no key protectors and Protection Status Off (check with `manage-bde -status C:`). Intune only checks that the drive is encrypted, so this passes even though the data is not really protected. The BitLocker policy in Part 9 fixes that.

## Part 9: BitLocker and configuration profiles

1. BitLocker: Intune, Endpoint security, Disk encryption, Create policy, Windows, BitLocker profile. Set Require device encryption = Enabled and Allow warning for other disk encryption = Disabled (this allows silent encryption on a TPM device). Under Operating System Drives, set Require additional authentication at startup = Enabled, Configure TPM startup = Require TPM, and set the TPM startup key, PIN, and key and PIN options to Do not allow (a startup PIN would lock you out of Remote Desktop at boot). Set Allow BitLocker without a compatible TPM to False. In the recovery section, set Allow data recovery agent to False, Omit recovery options from the BitLocker setup wizard to True, and leave Save BitLocker recovery information to AD DS for operating system drives on True (Intune uses the old AD DS label, but on an Entra joined device this backs up to Entra ID). Set Do not enable BitLocker until recovery information is stored to AD DS to True. Assign to `Lab-Users`. If the policy is slow to arrive, remember that the endpoint security editor finds All devices through its search box, and that a policy cannot mix specific groups with All devices.

   **Before this works, detach both ISOs.** BitLocker refuses to add protectors while bootable media is present. In Proxmox, open the VM, Hardware, edit each CD/DVD drive, and choose Do not use any media (the Windows 11 ISO and the VirtIO ISO). In the BitLocker Management event log on the VM the failure reads "Failed to enable Silent Encryption. TPM is not available" followed by "BitLocker Drive Encryption detected bootable media (CD or DVD) in the computer", event ID 853. The first half of that message is misleading. Intune still reports the policy as Succeeded while this error repeats, so the portal and the device disagree. Detach the ISOs before you create the BitLocker policy and you will skip this problem.
2. Sync the VM. Encryption can take several minutes to start. Check Settings, Privacy and security, Device encryption, or run `manage-bde -status C:` in an admin PowerShell window. Success looks like Protection Status: Protection On, with Numerical Password and TPM listed under Key Protectors. To see why it has not worked, run `Get-WinEvent -LogName "Microsoft-Windows-BitLocker/BitLocker Management" -MaxEvents 20 | Format-List TimeCreated, Id, Message`. Event 845 means the recovery key was backed up to Entra ID.
3. In Intune, open the device and look for Recovery keys. Seeing the key stored there is a good thing to screenshot (blur the key itself).
4. Firewall: Endpoint security, Firewall, Create policy, Windows, then choose the Windows Firewall profile (not Windows Firewall Rules or Hyper-V Firewall Rules). Set Enable Domain Network Firewall, Enable Private Network Firewall and Enable Public Network Firewall to True. Leave every other setting, including the default inbound and outbound actions, at Not configured. Before the policy lands, check on the VM that the Remote Desktop rules are enabled for the profile it sits on: `Get-NetConnectionProfile` and `Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select DisplayName, Enabled, Profile`. Assign to All devices, as with the BitLocker policy. Windows 11 usually has all three profiles on already, so `Get-NetFirewallProfile | Select Name, Enabled` looks the same before and after. The proof is the policy showing Succeeded under the device's Device configuration.
5. Settings catalog profile: Devices, Windows, Configuration, Create, New policy, Windows 10 and later, Settings catalog. Name it `Lab-Settings`. Use Add settings to search for and enable:
   - Interactive Logon: Machine Inactivity Limit (600 seconds).
   - Turn off Autoplay: Enabled, for All drives.
   - Interactive Logon: Message Title and Message Text For Users Attempting To Log On (for example "Lab System" and "Lab system. Authorized use only.").
   Assign to All devices.
6. Sync and confirm each profile shows Succeeded under the device's Device configuration. To verify on the VM, run `Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer -Name NoDriveTypeAutoRun` (expect 255) and `Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System | Select legalnoticecaption, legalnoticetext`. The logon notice appears at sign-in, so sign out completely and reconnect. Locking the screen inside a Remote Desktop session does not show it. The inactivity limit locks the session after 600 seconds idle.
   The DeviceManagement-Enterprise-Diagnostics-Provider Admin log will show recurring errors and warnings that do not mean the policies failed: DmGetAadUserTokenFailure (0xCAA10001) because the session uses a local account, a FakePolicy ADMXInstall error (Autoplay is an ADMX setting), and a DeviceStatus CertAttestation AllowedAIKAlgorithms error. The registry values and the Succeeded status are the real checks.
7. Back in the compliance policy view, the device should now flip to Compliant once BitLocker finishes and a sync completes.

## Part 10: Deploy an app

Two routes. The Microsoft Store app (new) route is quick, but only works for apps that are in the Store. Wireshark, for example, did not appear in the Store search, so a classic desktop app needs the Win32 route. This part uses Sysmon, which gives the lab real endpoint telemetry.

Quick route (Store apps)

1. Intune, Apps, Windows, Create, App type = Microsoft Store app (new). Search for the app (for example Windows Terminal), select it, and add it.
2. Assignments: Required, All devices. Leave the install context as System unless the app requires otherwise.
3. Sync the VM and wait. Confirm the app appears on the VM and shows Installed in Intune under Apps, Monitor.

Win32 route (Sysmon)

1. Windows Host: make a working folder outside your Git repos, for example `C:\IntuneLab\sysmon\source`, so the binaries are never committed. Download `Sysmon64.exe` from the Microsoft Sysinternals Sysmon page and a Sysmon config (the SwiftOnSecurity `sysmonconfig-export.xml` on GitHub is a common starting point; read it before deploying it). Put both files in `source`. Do not rename the config unless you change the install command to match. Download `IntuneWinAppUtil.exe` (the Win32 Content Prep Tool) from the `microsoft/Microsoft-Win32-Content-Prep-Tool` GitHub repo into `C:\IntuneLab\sysmon`.
2. Windows Host: package it. In PowerShell, in `C:\IntuneLab\sysmon`, run `.\IntuneWinAppUtil.exe -c .\source -s Sysmon64.exe -o .\output`. The result is `output\Sysmon64.intunewin`.
3. Windows Host: Intune, Apps, Windows, Create, App type = Windows app (Win32), Select. Upload the `.intunewin` file. Name `Sysmon`, publisher `Microsoft`.
4. Program tab: install command `Sysmon64.exe -accepteula -i sysmonconfig-export.xml` (use the real config file name), uninstall command `Sysmon64.exe -u force`, install behavior System, device restart behavior No specific action.
5. Requirements: 64-bit, Windows 10 1607 or later.
6. Detection rules: manually configure, rule type File, path `C:\Windows`, file `Sysmon64.exe`, detection method File or folder exists. This is the path on the VM after install, not the path where you built the package.
7. Leave Dependencies and Supersedence empty. Assignments: Required, All devices. Create. Take a Proxmox snapshot first so you can roll back.
8. Windows VM: sync and wait. Win32 apps are slower than policies. Verify with `Get-Service Sysmon64` (Running), then `Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 | Format-List TimeCreated, Id, Message`. Event 1 is a process creation and event 3 is a network connection. `C:\Windows\Sysmon64.exe -c` prints the loaded configuration; the config hash should match `Get-FileHash` of the file you packaged.
9. If an install fails or looks stuck, read `AppWorkload.log` in `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs` and search it for the app name. Newer Intune clients log app installs there, not in `IntuneManagementExtension.log`.
10. In Intune, Apps, Monitor, Device install status should show Installed.

## Part 11: Windows Update ring

1. Intune, Devices, Windows, Windows updates, Update rings, Create profile. Name it `Lab-Update-Ring`.
2. For a lab, set quality update deferral 0 days and feature update deferral 0 days. Leave Upgrade Windows 10 devices to Latest Windows 11 release on No (the VM already runs Windows 11), and pre-release builds on Not configured.
3. Under User experience settings, choose an Automatic update behavior that does not surprise you with a restart. This lab used Auto install and restart at a scheduled time, every day (Any Day), at a time when the VM is running but not in use. The VM must be powered on at that time for the install to happen.
4. Assign to All devices (update rings are device targeted, and the earlier lab policies ended up on All devices too).
5. Sync the VM. On the VM, the Settings, Accounts, Access work or school, Info page lists the managed areas, including Update. To check the values that landed, run `Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\Update"`. For this lab the values were ScheduledInstallTime 9, ScheduledInstallDay 0, AllowAutoUpdate 3, DeferQualityUpdatesPeriodInDays 0 and DeferFeatureUpdatesPeriodInDays 0. The WinningProvider values are the Intune enrollment ID.
6. Check the ring shows Succeeded under the device's Device configuration. Note in your write-up what a real organization would set differently (pilot rings, longer deferrals, maintenance windows).

Check the VM clock first. Scheduled installs use the VM's local time, and log timestamps depend on it. In this lab the VM had its time zone set to Pacific and its clock three hours ahead of real UTC, which showed up in the Sysmon `UtcTime` field. Fix it with `Set-TimeZone -Id "Eastern Standard Time"` (use your own zone), then `Restart-Service w32time` and `w32tm /resync /force`. After a large correction, `w32tm /query /status` can report not synchronized for a few minutes. Confirm with `Get-Date` and `(Get-Date).ToUniversalTime()`.

## Part 12: Conditional Access (the main event)

Warning: do this carefully and in order. A bad policy can lock out admins.

1. Security defaults first. New tenants have security defaults on, and you cannot use Conditional Access with them. In Entra, Overview, Properties, Manage security defaults, set to Disabled. Turning security defaults off makes Microsoft create its own managed Conditional Access policies (for example multifactor authentication for admins and for all users), so you do not need to build an admin MFA policy yourself.
2. Protect the break-glass account. In Entra, Protection, Conditional Access, Policies, open each Microsoft-managed policy, add `bg-admin` under Exclude, and save. Leave Microsoft's own exclusions (such as Directory Synchronization Accounts) as they are. Make sure your own admin account has a working MFA method first (an authenticator app is fine) so the managed admin policy does not lock you out. Optionally rename the admin account's display name (Entra, Users) to something readable like `Lab Admin`, which makes the sign-in logs easier to read.
3. Create the main policy `CA-LabUsers-Compliant-Device-MFA`. Users, Include = `Lab-Users`, Exclude = `bg-admin` and your admin account. Target resources = Office 365 (not All cloud apps, which is riskier). Grant = Require multifactor authentication and Require device to be marked as compliant, with "Require all the selected controls" chosen.
4. Set Enable policy = Report-only and create it.
5. Open https://www.office.com on the VM and sign in as `Lab User`. In Entra, Monitoring, Sign-in logs, open that sign-in and check the Report-only tab to see the policy would have been satisfied. Sign-in log entries can take several minutes to appear. Also sign in once as your admin account to confirm the exclusion shows Not applied.
6. Switch the policy to On. Report-only never blocks anything, so a private-window test on your normal PC will log a failure but still let you in until the policy is On.
7. Prove it works. On your normal PC, open a private browser window, go to https://www.office.com and sign in as `Lab User`. It should be blocked or told the device must be managed. On the VM, access works.
8. Prove it reacts to compliance. In the compliance policy, temporarily raise the minimum OS version above what Windows reports (for example `10.0.60000`; the portal rejects values above 65535 in any part, such as `10.0.99999`). Sync the VM, wait for it to turn Noncompliant, then try office.com on the VM and confirm it is blocked. Put the setting back, sync, and confirm access returns. Expect a delay here. In the lab, the device stayed Noncompliant for over an hour after the setting was restored, even though the device synced successfully, and then flipped to Compliant on its own. Removing and re-adding the assignment is a possible way to force it, but it was not needed.

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
- Win32 app stuck on Waiting for install status in the old Managed Apps report: use Apps, the app, Device install status instead, and sync the VM once more. Install activity is in `AppWorkload.log`.
- VM frozen, Remote Desktop error 0x204, Proxmox console unresponsive, and a graceful shutdown times out because the guest agent does not answer: use Stop (hard power-off) in Proxmox, then Start. Take a snapshot before risky changes.
- VM clock or time zone wrong (log timestamps in the future, scheduled installs at odd times): see the clock fix in Part 11.
- Device stays Noncompliant after you fix the compliance policy, and the per-setting page shows one setting failing that the device clearly passes: this was a stale evaluation. Check the policy's Monitor page, View report, and the Last contacted time. The VM can show a successful sync while the portal lags, and it can take over an hour to catch up. Do not keep changing the policy while you wait.
- Locked out by Conditional Access: sign in with `bg-admin`, set the policy to Off, and fix it.

## What to capture and write up

Take screenshots at each major step: VM hardware tab, the Windows install, the enrolled device in Intune, the compliance policy results, the BitLocker recovery key location (key blurred), the app installed, report-only results, and the blocked and allowed sign-ins. Write up what broke and how you fixed it, because that is the part interviewers care about.

For your public repo, blur or remove the tenant domain, user names, device names and IDs, and any recovery keys before posting.

If you complete it, a resume line you can truthfully use under Training & Independent Projects (adjust to what you actually did): "Built a Windows 11 lab on Proxmox managed by Microsoft Intune, including Entra ID join, compliance policies, BitLocker, app deployment, and a Conditional Access policy requiring MFA and a compliant device."

# ChatGPT desktop AppArmor: goals and test report

Tested: 2026-10-11

[Go to installation](#installation).

## What this is

The AppArmor access rules live in one file: `chatgpt-desktop`. It uses Ubuntu's maintained AppArmor implementation and the official, unmodified ChatGPT app. No custom runtime executable, container, proxy or GUI fork is required on your Ubuntu laptop. The VM scripts and probes used for testing are not installation requirements. This report is documentation, not another runtime component.

The access rules are maintained in that one file; the supplied launcher holds the standard launch settings. The required launch command also uses maintained systemd PrivateTmp=yes with the system manager. This requires administrator authorization to create the temporary namespace; ChatGPT itself still runs as your ordinary user with no capabilities and cannot invoke sudo. Launching through the ordinary icon without that option does not provide the reviewed browser-socket isolation. There is no custom unit file, executable or script. A standard `chatgpt-confined.desktop` launcher provides a clickable app icon.

The app needs to run its own software, store its login/settings, connect to the internet, and use your desktop window/clipboard. AppArmor allows those resources and refuses others. It checks the actual process regardless of what an LLM says or whether the app's own approval setting says Full Access.

This is resource confinement, not a promise that no code executes. ChatGPT's bundled Node runtime can execute arbitrary generated JavaScript. That JavaScript has the same file, process, device and service restrictions as the application. Fixed vendor helpers, stock SSH and a restricted pwd helper are intentional executable exceptions. Arbitrary local shells, sudo and other programs are denied. Remote SSH tasks intentionally execute commands on the REMOTE machine.

## Goals enforced by this policy

- No local sudo/pkexec/shell/other arbitrary executable launches or administrative system D-Bus access. The running app and SSH helper have zero effective Linux capabilities. Namespace sandbox permissions do not grant host root privileges.
- Hidden paths denied except this app's dedicated ChatGPT/AppData tree and one fixed read-only plugin manifest in the root-owned ChatGPT installation; normal ~/.ssh, ~/.codex, browser profiles and credential stores excluded.
- Documents and Private excluded, even if a shared directory is configured there.
- Conventional credential folders named Credentials, Secrets or Keys (including lowercase variants) excluded throughout the user's home. This cannot recognize a secret stored under an arbitrary ordinary filename in a shared directory.
- Configured attachments readable but not writable. Configured downloads writable. Other personal files default to inaccessible. Separate application state writable.
- System access limited to permitted libraries, fonts, DNS/TLS configuration, hardware metadata and other runtime needs. No general /etc or /tmp access.
- Same-user process command lines and environments denied, including thread aliases.
- Native Wayland window and normal clipboard allowed. X11 and input devices denied.
- Camera, screen capture/control, secret service and Tailscale administration denied.
- SSH uses its own confined child, with only an optional dedicated key. No local ProxyCommand, LocalCommand or agent forwarding. Remote permissions are separate.

## Audio tradeoff

Audio is disabled in the current single-file version, including microphone and playback. The read-only reviewer found that Ubuntu's unconfined pipewire-pulse service permits module-loading requests. A caller can supply an absolute LADSPA plugin path; the service loads code before validating the plugin descriptor. Allowing that socket would permit execution outside this app's AppArmor boundary. We blocked the socket and tested that connection fails with EACCES.

Keeping audio would require separately restricting/configuring the maintained audio service and testing that design. No such extra configuration is part of this deliverable. You explicitly chose to keep one AppArmor file and give up microphone and sound.

## Tested environment and results

Source: Ubuntu 26.04.1 LTS amd64, GNOME Wayland, AppArmor 5.0.2, official ChatGPT 26.1007.21434. A second Ubuntu 26.04.1 amd64 VM has the same official ChatGPT package and its stock Codex remote backend installed. Remote dummy account has no sudo rights. Source dummy is the management/login account; AppArmor restricts the app, not every program run by that account. This version does not rely on a separate Unix account for ChatGPT.

The ARM Mac emulates amd64 with QEMU TCG. Security behavior was checked inside the Ubuntu guest. VM speed/battery consumption is not native Ubuntu performance or a measurement of the policy's overhead.

- **PASS:** strict parser checks with --Werror --warn=rule-not-enforced and load.
- **PASS:** 36 current boundary checks, zero failures, using the bundled Node runtime under the production enforcing profile without an app-level sandbox. Included Documents, Private, hidden/nested hidden files, credentials, attachment writes, unrelated home files, system/temp files, shell/sudo/ Python/curl/systemctl launches, Kerberos cache, process/thread cmdline and environ, and the Pulse socket. Allowed exports and app scratch worked.
- **PASS:** HTTPS example.com returned 200 with expected page content after the nameservice include was tightened.
- **PASS:** all 36 resource checks and HTTPS/SSH were rerun in the actual app's private mount namespace. An unconfined, same-user synthetic UUID browser socket in the outer /tmp was invisible (connect ENOENT). PrivateTmp=yes, a different mount namespace, ordinary UID 1000, zero capabilities and enforcing label were independently verified after a cold launch.
- **PASS:** the app's native node_repl tool tried the synthetic Documents read and /bin/sh launch; both returned actual EACCES errors.
- **PASS:** fresh native local Work browser task under the final policy and private namespace. The official browser-client opened https://example.com/ and read its DOM and title, Example Domain. No shell or web-search fallback. Initial generic Playwright discovery failed; directing the task to its installed official browser instructions resolved tool discovery. A final narrow read-only exception also restores the official bundled plugin manifest, which the global hidden-file rule had blocked.
- **PASS:** two synthetic protected files opened outside the private namespace and passed as inherited file descriptors remained unreadable inside the enforcing profile; Documents and hidden-file aliases both returned EACCES.
- **PASS:** native clipboard paste into the app and copy back produced exact test text.
- **PASS:** native PNG attachment preview and submission of a synthetic red image.
- **PASS:** native drag/drop from GNOME Files into local Work under the enforcing profile and PrivateTmp=yes. Dragging the synthetic permitted text file attached it; the app's Node tool read its exact DRAG-ALLOWED-7K2P token. Dragging a synthetic Documents text file produced a filename chip, but its contents returned EACCES. The kernel audit confirms the AppArmor read denial. Drag/drop did not override the protected-folder rule.
- **PASS:** native Work selected a synthetic Documents file through the desktop chooser and displayed its filename chip, but reading its contents returned EACCES. Granting the app's requested file permission and approving its Node retry still returned EACCES. App-level approval did not override AppArmor.
- **PASS:** the fixed vendor plugin manifest is readable but not writable; both checks passed after the final exception, followed by all 36 checks and HTTPS/SSH.
- **PASS:** native app connected to the second VM through Tailscale SSH, opened the remote test project, read tailnet-boundary.txt, created native-desktop-proof.txt containing NATIVE-TAILSCALE-PASS, and ran hostname and id -un, saving chatgpt-ssh-target and dummy in remote-command-proof.txt. The target files were independently verified through management SSH.
- **PASS:** stock SSH still works under the latest confined policy after tightening nameservice. Actual desktop SSH process has chatgpt-desktop//ssh (enforce).
- **PASS:** main/renderer enforcement and zero effective capabilities; renderers retain Chromium seccomp and no-new-privileges.
- **PASS:** Camera/ScreenCast/RemoteDesktop/Screenshot portal calls denied in a temporary observer copy of this policy, with only diagnostic tool execution added. That observer was unloaded and deleted. No diagnostic allowances remain in the production policy.
- **PASS:** independent read-only security review, iterated until satisfied with the single-policy, audio-disabled scope. Three findings were fixed: credential reads in a stock abstraction, Pulse broker code loading, and process argv.

- **PASS:** standard desktop launcher validated and activated through the Ubuntu application menu and `gtk-launch`. The resulting app had `chatgpt-desktop (enforce)`, UID/GID 1000, zero effective capabilities and `PrivateTmp=yes`. The readonly policy comparison and loaded-profile checks passed. Negative startup cases (policy mismatch, missing enforcing profile and unavailable security assertion) prevented execution. All 36 resource checks and confined SSH passed again in the launched app's private namespace.

**Not yet verified:** physical-laptop behavior and drag/drop variants beyond the tested GNOME Files-to-local-Work text files. The desktop chooser is not confined by this policy and can show protected filenames; the Documents attachment check verifies denial of contents, not denial of selection. The old local test chat referred to the previous shared ~/.codex state and could not resume; it is not evidence of current browser Work success. The new browser test above uses the final dedicated state. Optional GTK/glycin image preview helpers are deliberately blocked; the app's Chromium PNG preview works.

## Tailscale SSH

No commercial Tailscale account was used. Headscale 0.29.4 supplied the local test control server; Tailscale 1.104.1 runs in both guests. Source 100.80.0.1 can reach only target 100.80.0.2 TCP 22 under the test tailnet rules. The target enables Tailscale SSH for dummy. Host test services listen on loopback only; no Mac home folders are shared into either VM. The real Mac Tailscale/VPN was not changed.

For your physical Ubuntu machines, use your normal Tailscale installation and SSH policy. Kernel-networking Tailscale SSH works with standard OpenSSH; no ProxyCommand is required in this tested Linux setup. The official remote backend must be installed, signed in and available as codex in the remote account's PATH. The app itself must not administer Tailscale or read its credentials.

Native alias discovery reads ChatGPT/AppData/.ssh/config. OpenSSH derives its normal config path from the OS account, not HOME, so use matching root-owned /etc/ssh/ssh_config.d/chatgpt.conf entries for the actual connection. Example:

```sshconfig
Host YOUR-CHATGPT-REMOTE
    HostName YOUR-TAILSCALE-IP
    User YOUR-REMOTE-USER
    UserKnownHostsFile /home/YOU/ChatGPT/AppData/.ssh/known_hosts
    StrictHostKeyChecking yes
    BatchMode yes
    IdentityAgent none
    PubkeyAuthentication no
    PasswordAuthentication no
    PreferredAuthentications none
    ForwardAgent no
    ForwardX11 no
    PermitLocalCommand no
```

Populate known_hosts with a host key verified independently on the target. Keep both alias configurations read-only to the app. This example is for Tailscale SSH authentication; ordinary key-based SSH instead needs the dedicated key and its corresponding authentication settings. Do not copy private account keys.

## Installation

Start on your Ubuntu laptop. Use its desktop terminal, as your normal user. These are first-install instructions; stop if a command fails.

### 1. Check Ubuntu and Wayland

```sh
cat /etc/os-release
dpkg --print-architecture
printf 'Session: %s\nWayland socket: %s\n' "$XDG_SESSION_TYPE" "$WAYLAND_DISPLAY"
```

Expected: Ubuntu 26.04, `amd64`, `wayland`, and a nonempty socket name. This policy was tested on 26.04.1.

### 2. Download and install ChatGPT

In Ubuntu's browser, open the [official Linux download page](https://learn.chatgpt.com/docs/linux/linux-app). Click **Ubuntu or Debian → Download .deb for x64**. Save it as `chatgpt_amd64.deb` in `Downloads`.

```sh
mkdir -p "$HOME/Downloads"
cd "$HOME/Downloads"
sudo apt update
sudo apt install ./chatgpt_amd64.deb
sudo apt install apparmor apparmor-utils openssh-client nano curl ca-certificates git desktop-file-utils
```

The first install adds ChatGPT and its dependencies; the second adds setup tools. Keep ChatGPT closed until step 7. [Official installation instructions](https://learn.chatgpt.com/docs/linux/linux-app).

### 3. Get this configuration

```sh
git clone https://github.com/donatopellegrino/chatgpt_apparmor.git "$HOME/ChatGPT-Setup"
cd "$HOME/ChatGPT-Setup"
```

Downloads the policy, launcher and this guide. Keep this setup folder: startup compares the installed policy with your copy. If the folder already exists, use it rather than cloning over it.

### 4. Match your account and folders

```sh
printf 'Home: %s\nUID: %s\nWayland: %s\n' "$HOME" "$(id -u)" "$WAYLAND_DISPLAY"
nano "$HOME/ChatGPT-Setup/chatgpt-desktop"
```

Set `CG_HOME`, `CG_UID` and `CG_WAYLAND` to those values. Adjust the folder variables. Remove `~/Downloads` from `CG_DOWNLOADS` if you want only `~/ChatGPT/Downloads` shared.

In nano: **Ctrl+O**, **Enter** to save; **Ctrl+X** to exit.

### 5. Create dedicated app state

```sh
app_state="$HOME/ChatGPT/AppData"
mkdir -p "$app_state/tmp" "$app_state/.codex" \
  "$HOME/ChatGPT/Attachments" "$HOME/ChatGPT/Downloads/Work"
nano "$app_state/.codex/config.toml"
```

Creates the app's state, attachment and download folders. Add this, replacing `YOUR_USER`:

```toml
[desktop]
projectlessWorkspaceRoot = "/home/YOUR_USER/ChatGPT/Downloads/Work"
```

### 6. Check and install AppArmor confinement

Fully quit ChatGPT, including any tray process. If you previously used the confined icon, stop its service:

```sh
sudo systemctl stop chatgpt-confined.service
```

Skip that command on a first installation. Check that no ChatGPT process remains:

```sh
pgrep -af '^/usr/lib/chatgpt/'
```

No output is expected. If anything is listed, stop before unloading a policy. Unloading a live profile removes its protection.

```sh
sudo aa-status
sudo apparmor_parser -Q -K --Werror --warn=rule-not-enforced "$HOME/ChatGPT-Setup/chatgpt-desktop"
```

Confirm AppArmor is enabled and the parser succeeds. If either fails, stop.

If `/etc/apparmor.d/chatgpt` exists, back up and unload the vendor profile:

```sh
sudo cp -a --no-clobber /etc/apparmor.d/chatgpt /var/backups/chatgpt-before-confinement
sudo apparmor_parser -R /etc/apparmor.d/chatgpt
```

Keeps the original policy for rollback. Skip that block if the vendor file is absent. A previously installed custom profile needs a separate backup; do not treat it as the vendor original.

```sh
sudo install -o root -g root -m 0644 "$HOME/ChatGPT-Setup/chatgpt-desktop" /etc/apparmor.d/chatgpt
sudo apparmor_parser -r -K --Werror --warn=rule-not-enforced /etc/apparmor.d/chatgpt
```

Installs the root-owned policy and activates it. If loading fails, restore the backup before launching.

### 7. Install the app icon and sign in

```sh
printf 'Home: %s\nUID: %s\nGID: %s\nWayland: %s\n' "$HOME" "$(id -u)" "$(id -g)" "$WAYLAND_DISPLAY"
nano "$HOME/ChatGPT-Setup/chatgpt-confined.desktop"
```

In its `Exec=` line, replace every `/home/dummy`, `--uid=1000`, `--gid=1000`, `/run/user/1000` and `wayland-0` with your values. Keep `PrivateTmp=yes`, `AssertSecurity=apparmor`, `ExecCondition` and `ExecStartPre`. The checks require the installed policy to match your setup copy and its loaded profile to be enforcing.

```sh
desktop-file-validate "$HOME/ChatGPT-Setup/chatgpt-confined.desktop"
mkdir -p "$HOME/.local/share/applications"
install -m 0644 "$HOME/ChatGPT-Setup/chatgpt-confined.desktop" "$HOME/.local/share/applications/chatgpt.desktop"
update-desktop-database "$HOME/.local/share/applications"
```

Validates and installs the launcher. It replaces the vendor icon for your account. If the menu still shows the old icon, log out and back in; remove/re-add an old dock favourite if necessary.

Open Ubuntu's application menu and click **ChatGPT (confined)**. A terminal may ask for your Ubuntu administrator password; enter it normally. ChatGPT itself runs as your ordinary user. Startup checks require AppArmor enabled, our profile enforcing and the installed policy matching your setup copy. Verify the actual process label in step 8; keep competing vendor profiles unloaded.

Click **Sign in**. Complete sign-in in the browser on this same Ubuntu laptop, then return to the app. Keep microphone and camera disabled.

This icon does not handle external `codex:` links or “Open With” file launches. Use this icon for later launches. If ChatGPT is already open, switch to its window. To use a different Wayland socket after a session change, update both configurations before launching.

### 8. Confirm protection and features

```sh
systemctl show chatgpt-confined -p PrivateTmp -p User -p MainPID
sudo aa-status
```

Confirm `PrivateTmp=yes`, your user and `chatgpt-desktop` in enforce mode. To check the actual app process, copy its `MainPID` into this command:

```sh
sudo cat /proc/REPLACE_WITH_MainPID/attr/current
```

Expected: `chatgpt-desktop (enforce)`.

Try these in a new local Work chat:

- Browse `https://example.com/` and read the page title.
- Copy/paste text; attach and drag a harmless file from `~/ChatGPT/Attachments`.
- Ask it to read a harmless test file in `~/Documents`; reading must fail.
- Ask it to run `/bin/sh`; execution must fail.

Audio, camera and screen capture remain disabled. Remote Work setup follows below.

### 9. Set up Tailscale SSH remote Work

Skip this section until you have a remote machine. Commands marked **remote** run on that machine, as its intended remote account.

**Both machines:** if Tailscale is absent, install it using the [official Linux installer](https://tailscale.com/download/linux):

```sh
mkdir -p "$HOME/Downloads"
curl -fsSL https://tailscale.com/install.sh -o "$HOME/Downloads/tailscale-install.sh"
sh "$HOME/Downloads/tailscale-install.sh"
sudo tailscale up
```

Installs maintained Tailscale software; follow the displayed sign-in link. Use the same tailnet. Skip installation and `up` on machines already connected; preserve their VPN/routing settings.

**Remote:** use a normal, least-privilege account. Enable Tailscale SSH from its console or another working access path:

```sh
sudo tailscale set --ssh
tailscale ip -4
```

Records the target's Tailscale IP. Enabling SSH replaces SSH handling on that Tailscale IP and can interrupt existing connections there. Rollback: `sudo tailscale set --ssh=false`. [Tailscale SSH instructions](https://tailscale.com/docs/features/tailscale-ssh).

In the [Tailscale admin console](https://console.tailscale.com/), open **Access controls**. Permit your laptop to reach the target on TCP 22, and permit your Tailscale identity to SSH as the chosen remote Unix user. Keep other rules intact. Complete any SSH check-mode sign-in before using ChatGPT.

**Remote Ubuntu:** repeat step 2 to install official ChatGPT. Expose its bundled remote backend, then sign in:

```sh
sudo ln -s /usr/lib/chatgpt/resources/codex /usr/local/bin/codex
codex login --device-auth
command -v codex
```

Do not replace an existing `codex` command; use it instead. Complete the displayed account/device-code sign-in. [Official remote setup](https://learn.chatgpt.com/docs/remote-connections).

**Laptop:** create the two SSH alias files:

```sh
mkdir -p "$HOME/ChatGPT/AppData/.ssh"
sudo nano /etc/ssh/ssh_config.d/chatgpt.conf
```

Paste the example in the **Tailscale SSH** section above. Replace the alias, target IP, remote username and local home path.

```sh
sudo install -o "$(id -u)" -g "$(id -g)" -m 0444 /etc/ssh/ssh_config.d/chatgpt.conf "$HOME/ChatGPT/AppData/.ssh/config"
```

Uses the same alias for ChatGPT discovery and OpenSSH connections.

Collect the target's public SSH host key:

```sh
ssh-keyscan -t ed25519 YOUR-TAILSCALE-IP > "$HOME/ChatGPT-Setup/remote-host-key"
ssh-keygen -lf "$HOME/ChatGPT-Setup/remote-host-key"
```

Verify the fingerprint independently against the target's **Tailscale SSH** public host key before continuing. `ssh-keyscan` alone does not verify identity; the ordinary OpenSSH server may have a different key.

```sh
sudo install -o "$(id -u)" -g "$(id -g)" -m 0444 "$HOME/ChatGPT-Setup/remote-host-key" "$HOME/ChatGPT/AppData/.ssh/known_hosts"
ssh YOUR-CHATGPT-REMOTE 'hostname; id -un; command -v codex'
```

Confirms the remote machine, user and backend. Quit and reopen ChatGPT through the confined icon so it discovers the new alias. Open **Settings → Connections → SSH**, add the alias and select a remote project folder. Test a harmless read and write there.

### Updates and rollback

Stop ChatGPT and confirm no processes remain, as in step 6. Then update the official app with:

```sh
sudo apt update
sudo apt install --only-upgrade chatgpt
```

After updates, repeat steps 6–8: package updates may replace the policy. Keep the original vendor backup. If `aa-status` lists both `chatgpt` and `chatgpt-desktop`, unload the vendor profile from that original backup before launching; do not keep competing attachments.

To restore the vendor policy and icon, stop ChatGPT and confirm no processes remain, as in step 6. Then:

```sh
sudo apparmor_parser -R /etc/apparmor.d/chatgpt
sudo cp -a /var/backups/chatgpt-before-confinement /etc/apparmor.d/chatgpt
sudo apparmor_parser -r /etc/apparmor.d/chatgpt
rm "$HOME/.local/share/applications/chatgpt.desktop"
update-desktop-database "$HOME/.local/share/applications"
```

Requires the backup from step 6. Restoring it removes this custom confinement.

## Isolation details

The native browser broker hardcodes /tmp/codex-browser-use instead of respecting TMPDIR. Filesystem Unix sockets do not enforce AppArmor peer labels. A narrow UUID-shaped pathname exception therefore requires effective PrivateTmp isolation. The user-manager launcher was tested and hit Ubuntu's unprivileged_userns stacking; the system-manager launcher works without disabling Chromium's sandbox.

Existing desktop sockets lie outside that mount namespace. The main profile uses attach_disconnected.path=/att/chatgpt-desktop/ to map such foreign objects into a separate, otherwise denied prefix. Only /dev/null and this user's exact session bus and Wayland socket paths are allowed there. No personal file, browser broker, Pulse socket, executable or mapped library receives a prefixed allowance. This is different from attaching all disconnected objects to /, which would let them match ordinary file permissions. The kernel mechanism has aliasing limitations; regression tests support this narrow design, not a proof against vulnerabilities. Do not fix breakage with broad execution, home, socket or unrestricted `attach_disconnected` allowances.

## Limits and worst-case damage

The app could read/transmit every allowed attachment, download or clipboard item, and corrupt/delete its writable downloads and application state. It could consume CPU, memory or disk and run arbitrary bundled JavaScript. It could access network services and exercise the permissions of an enabled remote SSH account.

Network access is ordinary IPv4/IPv6, including LAN, loopback and VPN destinations. This one file does not enforce public-internet-only access, a remote-host allowlist, or a VPN kill switch. Normal routing applies; configure your VPN independently. The test tailnet restrictions are Headscale rules, not AppArmor guarantees.

The unconfined file chooser may show filenames that the app cannot read. Copies or aliases you deliberately place in an allowed location can expose data. Keep secrets out of shared trees. No unconfined app should consume/execute command configuration or code from writable ChatGPT state/download directories; the policy cannot protect future unconfined consumers. In particular never share this CODEX_HOME with an unconfined Codex process.

This protects the confined ChatGPT process, not all applications system-wide. It is not macOS-style global Documents privacy and not a guarantee against kernel, AppArmor or vendor-library vulnerabilities. Other processes and remote accounts have their own permissions. AppArmor is not a substitute for updating Ubuntu.

## Test evidence

Test-only files; not required on your Ubuntu laptop.

Test evidence is retained in the author's local VM test workspace: `boundaries-final.txt`, `network-final.txt`, `native-tailscale-task.txt`, `private-socket-boundary.txt`, `foreign-aliases.txt`, `native-local-browser.txt`, `native-documents-attachment.txt`, `vendor-manifest.txt`, `enforcement-final.txt` and `drag-drop-native.txt`, `launcher-native.txt` and `launcher-guards.txt`, plus GUI screenshots from the earlier feature tests. The test harness and VM images are not part of this installation.

The earlier desktop-services log records successful audio connectivity BEFORE review; it is superseded by the current Pulse EACCES regression test.

### Primary references

- [ChatGPT Linux download and installation](https://learn.chatgpt.com/docs/linux/linux-app)
- [Tailscale Linux installation](https://tailscale.com/download/linux)
- [ChatGPT remote connections](https://learn.chatgpt.com/docs/remote-connections)
- [Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh)
- [Headscale policy](https://headscale.net/stable/ref/policy/)
- [AppArmor policy syntax](https://apparmor.net/man/5.0/apparmor.d/)
- [PipeWire Pulse server source](https://raw.githubusercontent.com/PipeWire/pipewire/1.6.2/src/modules/module-protocol-pulse/pulse-server.c)
- [PipeWire LADSPA sink source](https://raw.githubusercontent.com/PipeWire/pipewire/1.6.2/src/modules/module-protocol-pulse/modules/module-ladspa-sink.c)
- [PipeWire LADSPA plugin source](https://raw.githubusercontent.com/PipeWire/pipewire/1.6.2/spa/plugins/filter-graph/plugin_ladspa.c)


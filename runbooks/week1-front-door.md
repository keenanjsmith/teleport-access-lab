# Teleport Access Lab: RUNBOOK

**Week 1 of 4: Build the front door**
Target window: Thu Oct 1 to Fri Oct 2, 2026 (compressed schedule)
Teleport version: **18.11.0** (Community Edition)

---

## 🗺️ THE BIG PICTURE (read once, then start at Phase 0)

By the end of Week 1, you log into your own Teleport web page, get asked for a code from your phone, and land on a dashboard showing the server you're protecting.

To get there, you will:

1. **Phase 0:** Prep your Windows PC
2. **Phase 1:** Build one Ubuntu Server VM to use as a template
3. **Phase 2:** Copy that template four times and give each copy its own identity
4. **Phase 3:** Teach your PC and the front door the name `teleport.lab.internal`
5. **Phase 4:** Make a trusted certificate so the web page shows a padlock
6. **Phase 5:** Install and start Teleport
7. **Phase 6:** Create your admin login with phone MFA, and take the two proof screenshots
8. **Phase 7:** Save a snapshot of everything

**The four lab VMs:**

| VM | Its job | Its address |
|---|---|---|
| tp-proxy | The front door, runs Teleport | `192.168.56.10` |
| tp-node1 | Linux server behind the door | `192.168.56.11` |
| tp-node2 | Linux server behind the door | `192.168.56.12` |
| tp-db | Database server behind the door | `192.168.56.13` |

> ✅ **You never have to scroll back up.** Once the steps start, every step tells you which window you should be in, and repeats every name, address, and value it needs right there.

---

## 🚦 MARKERS USED IN EVERY STEP

| Marker | Meaning |
|---|---|
| 📍 **WHERE YOU SHOULD BE** | Which window should be open before you start the step |
| 🛑 **STOP** | Do not move on until this is right |
| ✏️ **REPLACE** | Swap in your own value. Anything not marked REPLACE is typed exactly |
| ✅ **CHECK** | What you should see if it worked. If you don't see it, stop and get help |
| 📸 **CAPTURE** | Take a screenshot now |
| ⚠️ **HEADS UP** | Something surprising that is normal |

**Every command box is labeled with the exact window to type in, and what the prompt should look like:**

| Label | Which window | The prompt looks like |
|---|---|---|
| **[PowerShell window → Windows]** | A regular Windows PowerShell window, **not** connected to any VM | `PS C:\Users\yourname>` |
| **[Admin PowerShell window → Windows]** | PowerShell opened with **Run as administrator**, not connected to any VM | `PS C:\WINDOWS\system32>` |
| **[PowerShell window → connected to tp-xxxx]** | The **same Windows PowerShell window**, after you've connected to that VM with `ssh`. You type here, and it runs on the VM | `labadmin@tp-xxxx:~$` |
| **[VirtualBox window → tp-xxxx]** | That VM's own VirtualBox window. Short commands only, because paste doesn't work there | `labadmin@...:~$` |

> ⚠️ **HEADS UP: Think of a connected PowerShell window like a remote control.** You're holding it in Windows, but every button you press works the VM. The prompt tells you which one you're controlling: `PS C:\` means Windows, and `labadmin@` means a VM.

🛑 **STOP: Only run `ssh` when the prompt starts with `PS C:\`.** If the prompt starts with `labadmin@`, you're already inside a VM. Typing `ssh` there connects from that VM to another one, stacking sessions inside each other, and each `exit` only removes one layer. If you lose track, close the PowerShell window with the **X** and open a fresh one. That drops every layer at once.

🛑 **STOP: Paste never works in a VirtualBox console window**, even with shared clipboard turned on. Ubuntu Server has no desktop for the clipboard to reach. Long commands always go through **SSH from PowerShell**, where right-click pastes.

---

## 🔑 QUICK REFERENCE (you don't need this to follow the steps)

| What | Value |
|---|---|
| Internet adapter inside each VM | `enp0s3` |
| Lab network adapter inside each VM | `enp0s8` |
| Linux username on every VM | `labadmin` |
| Cluster web address | `https://teleport.lab.internal` |
| Teleport admin user | `keenan-admin` |
| Lab folder on your PC | `D:\teleport-lab\` |

---

## Phase 0: Prep your Windows PC

### 0.1 Check the VirtualBox lab network

📍 **WHERE YOU SHOULD BE:** On your Windows desktop.

1. Open **VirtualBox**.
2. Click **File > Tools > Network Manager**.
3. Click the **Host-only Networks** tab.
4. ✅ **CHECK:** there's an adapter with IPv4 address `192.168.56.1` and mask `255.255.255.0`.
5. If there isn't one, click **Create**. It normally fills in `192.168.56.1` on its own.

### 0.2 Make the lab folders

📍 **WHERE YOU SHOULD BE:** On your Windows desktop.

1. Press the **Windows key**, type `PowerShell`, and press **Enter**.

**[PowerShell window → Windows]**
```powershell
New-Item -ItemType Directory -Path D:\teleport-lab\iso, D:\teleport-lab\vms, D:\teleport-lab\certs -Force
```

✅ **CHECK:** it lists three folders: `iso`, `vms`, and `certs`.

### 0.3 Download Ubuntu Server

📍 **WHERE YOU SHOULD BE:** In a web browser.

1. Go to **ubuntu.com/download/server**.
2. Download **Ubuntu Server 24.04 LTS**.
3. Save the `.iso` file somewhere you'll remember, like `D:\teleport-lab\iso\` or `D:\ISOs\`.

⚠️ **HEADS UP:** It's a few gigabytes. Let it download while you do step 0.4.

### 0.4 Install mkcert, your private certificate office

mkcert makes a small private certificate authority on your PC. Think of it as your own mini company badge office. Your browser will trust badges it issues, so the Teleport page later shows a padlock instead of a warning.

📍 **WHERE YOU SHOULD BE:** On your Windows desktop.

1. Press the **Windows key** and type `PowerShell`.
2. **Right-click Windows PowerShell** and choose **Run as administrator**. Click **Yes**.

**[Admin PowerShell window → Windows]**
```powershell
winget install -e --id FiloSottile.mkcert
```

🛑 **STOP:** Close this PowerShell window. Then open a **new** one the same way, with **Run as administrator**. Windows won't find the `mkcert` command until you do.

**[Admin PowerShell window → Windows]**
```powershell
mkcert -install
```

If Windows asks whether to install a certificate, click **Yes**.

✅ **CHECK:** you see `The local CA is now installed in the system trust store!`

⚠️ **HEADS UP:** It also says Firefox support is not available. That's fine. Use Chrome or Edge for this lab.

You can close this PowerShell window.

---

## Phase 1: Build the template VM (tp-base)

### 1.1 Create the VM

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

1. Click **New**.
2. Fill in:
   - Name: `tp-base`
   - Folder: `D:\teleport-lab\vms`
   - ISO Image: the Ubuntu Server `.iso` you downloaded in step 0.3
   - Check **Skip Unattended Installation**
3. Click **Next**, then set Memory to **2048 MB** and Processors to **2**.
4. Click **Next**, then set the hard disk to **25 GB**. Leave **Pre-allocate Full Size** unchecked.
5. Click **Next**, then **Finish**.

🛑 **STOP: Don't start the VM yet.** First give it a second network card:

6. Select **tp-base** and click **Settings**.
7. Click **Network**.
8. **Adapter 1** tab: leave it as **NAT**. This is the VM's internet connection.
9. **Adapter 2** tab: check **Enable Network Adapter**. Set **Attached to** to **Host-only Adapter**, and pick `VirtualBox Host-Only Ethernet Adapter`. This is the private lab network.
10. Click **OK**.

### 1.2 Install Ubuntu

📍 **WHERE YOU SHOULD BE:** In VirtualBox, with **tp-base** selected.

1. Click **Start**. A new window opens and the installer loads.

Use the **arrow keys** and **Enter** to move through the installer. Here's every screen in order:

| Screen | What to do |
|---|---|
| **Language** | English, press **Enter** |
| **Installer update available** | ⚠️ Choose **Continue without updating**. This only updates the installer itself, and the full system gets updated in step 1.3 |
| **Keyboard** | Leave the defaults, choose **Done** |
| **Type of install** | **Ubuntu Server**, not minimized. Choose **Done** |
| **Network** | 🛑 See the check right below this table |
| **Proxy** | Leave blank, choose **Done** |
| **Ubuntu archive mirror** | Wait for the test to say it passed, then choose **Done** |
| **Storage** | Leave the defaults, choose **Done**. On the summary, choose **Done** again, then **Continue** to confirm |
| **Profile** | Your name `Lab Admin`, server name `tp-base`, username `labadmin`, and a password you save in your password manager |
| **Upgrade to Ubuntu Pro** | **Skip for now**, then **Continue** |
| **SSH** | 🛑 Check **Install OpenSSH server** with the spacebar. Without it, you can't SSH in or paste later |
| **Featured snaps** | Select nothing, choose **Done** |

🛑 **STOP: On the Network screen, you must see two adapters:**

- `enp0s3` with an address starting `10.0.2.`. This is the VM's internet connection.
- `enp0s8` with an address starting `192.168.56.`. This is the private lab network.

Neither one is your PC's own network connection. Both belong to the VM.

If you only see `enp0s3`, the second adapter wasn't turned on. To fix it:
1. Close the VM window and choose **Power off the machine**.
2. Select **tp-base**, click **Settings > Network > Adapter 2**, check **Enable Network Adapter**, and set it to **Host-only Adapter**.
3. Click **OK**, then **Start** and go through the installer again.

2. When the install finishes, choose **Reboot Now**.
3. If it says to remove the installation medium, press **Enter**.

### 1.3 Update the template

📍 **WHERE YOU SHOULD BE:** In the **tp-base** VirtualBox window, at a `tp-base login:` prompt.

1. Type `labadmin` and press **Enter**.
2. Type your password and press **Enter**. Nothing shows while you type the password. That's normal.

⚠️ **HEADS UP:** You may see `New release '26.04.1 LTS' available. Run 'do-release-upgrade'`. 🛑 **Don't run it.** This lab stays on 24.04.

These commands are short, so type them in the console.

**[VirtualBox window → tp-base]**
```bash
sudo apt update && sudo apt full-upgrade -y
```

It asks for your password once. Then it downloads and installs updates for a few minutes.

⚠️ **HEADS UP:** If a purple screen asks about restarting services or a newer kernel, press **Enter** to accept.

**[VirtualBox window → tp-base]**
```bash
ip -br a
```

✅ **CHECK:** you see `enp0s3` with a `10.0.2.` address and `enp0s8` with a `192.168.56.` address.

**[VirtualBox window → tp-base]** turn it off:
```bash
sudo shutdown -h now
```

The VM window closes on its own.

### 1.4 Snapshot the template

📍 **WHERE YOU SHOULD BE:** In VirtualBox, with **tp-base** showing **Powered Off**.

1. Select **tp-base**.
2. Click the menu icon next to its name and choose **Snapshots**.
3. Click **Take**.
4. Name it `clean-install` and click **OK**.

🛑 **STOP: Never install Teleport on tp-base.** Teleport gives each machine a unique identity, and copying a machine that already has one would give two servers the same identity.

---

## Phase 2: Make the four lab VMs

### 2.1 Clone tp-base four times

📍 **WHERE YOU SHOULD BE:** In VirtualBox, with **tp-base** showing **Powered Off**.

Do this four times. The only thing that changes each time is the **Name**: first `tp-proxy`, then `tp-node1`, then `tp-node2`, then `tp-db`.

1. Right-click **tp-base** and choose **Clone**.
2. Fill in:
   - Name: ✏️ **REPLACE** with `tp-proxy`, `tp-node1`, `tp-node2`, or `tp-db`
   - Path: `D:\teleport-lab\vms`
   - 🛑 MAC Address Policy: **Generate new MAC addresses for all network adapters**
3. Click **Next**.
4. Clone type: **Full clone**.
5. Snapshots: **Current machine state**. The copies don't need tp-base's snapshot history.
6. Click **Finish** and wait for it to complete.

✅ **CHECK:** after four rounds, VirtualBox lists `tp-proxy`, `tp-node1`, `tp-node2`, and `tp-db`.

### 2.2 Set each VM's size

📍 **WHERE YOU SHOULD BE:** In VirtualBox, with all four new VMs **Powered Off**.

For each VM in this table: right-click it, choose **Settings > System**, set the values, and click **OK**.

| VM | Processors (Processor tab) | Base Memory (Motherboard tab) |
|---|---|---|
| tp-proxy | 2 | 4096 MB |
| tp-node1 | 1 | 1024 MB |
| tp-node2 | 1 | 1024 MB |
| tp-db | 1 | 2048 MB |

⚠️ **HEADS UP:** tp-proxy gets 4096 MB on purpose. It runs the front door, the web page, the audit log, and recording uploads all at once. At 2048 MB, it froze repeatedly during Week 3.

### 2.3 Give each VM its own identity

Each copy starts as an exact twin of tp-base. These steps give it **its own name, its own machine ID, its own SSH keys, and its own fixed address**, so Teleport can tell them apart.

Do the VMs one at a time, in this order. Each one has its own complete section. Start at the top of a section and go straight down.

⚠️ **HEADS UP: You never need to look up a temporary address.** Each section has you give the VM its final address first, in the VM window, so you always know exactly where to connect.

---

#### 2.3A tp-proxy → becomes `192.168.56.10`

tp-proxy is the front door. Its final address is **`192.168.56.10`**.

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

1. Select **tp-proxy** and click **Start**.

🛑 **CHECK the title bar** of the window that opens. It must say **tp-proxy [Running] - Oracle VM VirtualBox**. With several VM windows open, it's easy to type into the wrong one.

2. At the `login:` prompt, type `labadmin`, press **Enter**, then type your password and press **Enter**.

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`. It still says `tp-base` because this is a copy. You rename it in a minute.

**[VirtualBox window → tp-proxy]** give it its final address right now, so you know exactly where to connect. This one is short, so type it:
```bash
sudo ip addr add 192.168.56.10/24 dev enp0s8
```

Type your password if it asks.

**[VirtualBox window → tp-proxy]** check it:
```bash
ip -br a
```

✅ **CHECK:** the `enp0s8` line shows `192.168.56.10/24`.

⚠️ **HEADS UP:** That line may also show a second address like `192.168.56.101/24`. That's a temporary one from VirtualBox. Ignore it. It disappears later in this section.

🛑 **STOP: Switch windows now.** Leave the VirtualBox window alone. If you don't have a PowerShell window open, press the **Windows key**, type `PowerShell`, and press **Enter**.

✅ **CHECK:** the title bar says **Windows PowerShell**, and the prompt starts with `PS C:\`. 🛑 If the prompt starts with `labadmin@` instead, you're already inside a VM. Close this window with the **X** and open a fresh PowerShell.

**[PowerShell window → Windows]** connect to tp-proxy:
```powershell
ssh labadmin@192.168.56.10
```

⚠️ **HEADS UP:** It asks `Are you sure you want to continue connecting (yes/no/[fingerprint])?` That's normal. Type `yes` and press **Enter**, then type your password.

⚠️ **HEADS UP:** If it says `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`, your PC remembers an older VM at this address. Run these two lines in the same PowerShell window:
```powershell
ssh-keygen -R 192.168.56.10
ssh labadmin@192.168.56.10
```

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-proxy** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-proxy]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.10/24`. If it doesn't, run `sudo ip addr add 192.168.56.10/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.10
```

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`, and the window's title bar **still says PowerShell**. 🛑 If the prompt shows the name of a VM you already finished, like `tp-node1`, you typed the address step into the wrong VirtualBox window. Turn that VM off with `sudo shutdown -h now`, run `ssh-keygen -R` with this section's address in PowerShell, and restart this section. You're connected over SSH, and from here on **right-click pastes**.

From here, copy each box and right-click to paste it. Nothing needs replacing.

**[PowerShell window → connected to tp-proxy]** give it its name:
```bash
sudo hostnamectl set-hostname tp-proxy
sudo sed -i 's/tp-base/tp-proxy/g' /etc/hosts
```

It asks for your password once.

**[PowerShell window → connected to tp-proxy]** give it a new machine ID, like a new fingerprint:
```bash
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
```

✅ **CHECK:** it prints `Initializing machine ID from random generator.`

**[PowerShell window → connected to tp-proxy]** give it new SSH keys, like its own house key:
```bash
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
```

✅ **CHECK:** the middle command prints lines starting with `Creating SSH2`. You stay connected.

**[PowerShell window → connected to tp-proxy]** make `192.168.56.10` permanent, so it survives a reboot. Paste this whole box at once:
```bash
sudo tee /etc/netplan/60-hostonly.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
EOF
sudo chmod 600 /etc/netplan/60-hostonly.yaml
sudo netplan apply
```

⚠️ **HEADS UP: It may look like nothing is happening. That's normal.** Press **Enter** once, wait about 10 seconds, then match what you see:

| What you see | What to do |
|---|---|
| The prompt comes back as `labadmin@...:~$` | Type `exit` and press **Enter** |
| `Connection reset` or `closed`, and the prompt reads `PS C:\` | Nothing, you're already out |
| Still nothing | Type `~.`, a tilde then a period. The prompt changes to `PS C:\` |

✅ **CHECK:** you're at `PS C:\`.

**[PowerShell window → Windows]** reconnect. The first line clears your PC's memory of the **old** house key, because you just gave tp-proxy a new one:
```powershell
ssh-keygen -R 192.168.56.10
ssh labadmin@192.168.56.10
```

⚠️ **HEADS UP:** It asks the `yes/no/[fingerprint]` question again. Type `yes`, press **Enter**, then type your password.

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-proxy** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-proxy]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.10/24`. If it doesn't, run `sudo ip addr add 192.168.56.10/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.10
```

✅ **CHECK:** the prompt now reads `labadmin@tp-proxy:~$`. The new name shows up because you logged in fresh.

**[PowerShell window → connected to tp-proxy]** confirm everything took:
```bash
hostname
ip -br a
```

✅ **CHECK:**
- `hostname` prints `tp-proxy`
- The `enp0s8` line shows `192.168.56.10/24`, and the temporary `.101` address is gone

⚠️ **HEADS UP:** If `enp0s8` still also shows a `192.168.56.101` address, the last line of the permanent-address box never ran. Paste this and press **Enter**, then run `ip -br a` again:
```bash
sudo netplan apply
```

**[PowerShell window → connected to tp-proxy]** tp-proxy stays **running**. Just leave the session:
```bash
exit
```

✅ **CHECK:** you're back at `PS C:\`. Keep this PowerShell window open for the next VM.

---

#### 2.3B tp-node1 → becomes `192.168.56.11`

tp-node1 is a server behind the door. Its final address is **`192.168.56.11`**.

📍 **WHERE YOU SHOULD BE:** In VirtualBox. tp-proxy is still running, which is fine and needed.

1. Select **tp-node1** and click **Start**.

🛑 **CHECK the title bar** of the window that opens. It must say **tp-node1 [Running] - Oracle VM VirtualBox**. With several VM windows open, it's easy to type into the wrong one.

2. At the `login:` prompt, type `labadmin`, press **Enter**, then type your password and press **Enter**.

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`. It still says `tp-base` because this is a copy. You rename it in a minute.

**[VirtualBox window → tp-node1]** give it its final address right now, so you know exactly where to connect. This one is short, so type it:
```bash
sudo ip addr add 192.168.56.11/24 dev enp0s8
```

Type your password if it asks.

**[VirtualBox window → tp-node1]** check it:
```bash
ip -br a
```

✅ **CHECK:** the `enp0s8` line shows `192.168.56.11/24`.

⚠️ **HEADS UP:** That line may also show a second address like `192.168.56.101/24`. That's a temporary one from VirtualBox. Ignore it. It disappears later in this section.

**[VirtualBox window → tp-node1]** make sure it can reach tp-proxy:
```bash
ping -c 2 192.168.56.10
```

✅ **CHECK:** two lines that say `64 bytes from 192.168.56.10`.
🛑 If it says `0 received`, check that **tp-proxy** shows **Running** in VirtualBox. If it's off, start it, wait a minute, and ping again.

🛑 **STOP: Switch windows now.** Leave the VirtualBox window alone. If you don't have a PowerShell window open, press the **Windows key**, type `PowerShell`, and press **Enter**.

✅ **CHECK:** the title bar says **Windows PowerShell**, and the prompt starts with `PS C:\`. 🛑 If the prompt starts with `labadmin@` instead, you're already inside a VM. Close this window with the **X** and open a fresh PowerShell.

**[PowerShell window → Windows]** connect to tp-node1:
```powershell
ssh labadmin@192.168.56.11
```

⚠️ **HEADS UP:** It asks `Are you sure you want to continue connecting (yes/no/[fingerprint])?` That's normal. Type `yes` and press **Enter**, then type your password.

⚠️ **HEADS UP:** If it says `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`, your PC remembers an older VM at this address. Run these two lines in the same PowerShell window:
```powershell
ssh-keygen -R 192.168.56.11
ssh labadmin@192.168.56.11
```

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-node1** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-node1]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.11/24`. If it doesn't, run `sudo ip addr add 192.168.56.11/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.11
```

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`, and the window's title bar **still says PowerShell**. 🛑 If the prompt shows the name of a VM you already finished, like `tp-node1`, you typed the address step into the wrong VirtualBox window. Turn that VM off with `sudo shutdown -h now`, run `ssh-keygen -R` with this section's address in PowerShell, and restart this section. You're connected over SSH, and from here on **right-click pastes**.

From here, copy each box and right-click to paste it. Nothing needs replacing.

**[PowerShell window → connected to tp-node1]** give it its name:
```bash
sudo hostnamectl set-hostname tp-node1
sudo sed -i 's/tp-base/tp-node1/g' /etc/hosts
```

It asks for your password once.

**[PowerShell window → connected to tp-node1]** give it a new machine ID, like a new fingerprint:
```bash
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
```

✅ **CHECK:** it prints `Initializing machine ID from random generator.`

**[PowerShell window → connected to tp-node1]** give it new SSH keys, like its own house key:
```bash
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
```

✅ **CHECK:** the middle command prints lines starting with `Creating SSH2`. You stay connected.

**[PowerShell window → connected to tp-node1]** make `192.168.56.11` permanent, so it survives a reboot. Paste this whole box at once:
```bash
sudo tee /etc/netplan/60-hostonly.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.11/24
EOF
sudo chmod 600 /etc/netplan/60-hostonly.yaml
sudo netplan apply
```

⚠️ **HEADS UP: It may look like nothing is happening. That's normal.** Press **Enter** once, wait about 10 seconds, then match what you see:

| What you see | What to do |
|---|---|
| The prompt comes back as `labadmin@...:~$` | Type `exit` and press **Enter** |
| `Connection reset` or `closed`, and the prompt reads `PS C:\` | Nothing, you're already out |
| Still nothing | Type `~.`, a tilde then a period. The prompt changes to `PS C:\` |

✅ **CHECK:** you're at `PS C:\`.

**[PowerShell window → Windows]** reconnect. The first line clears your PC's memory of the **old** house key, because you just gave tp-node1 a new one:
```powershell
ssh-keygen -R 192.168.56.11
ssh labadmin@192.168.56.11
```

⚠️ **HEADS UP:** It asks the `yes/no/[fingerprint]` question again. Type `yes`, press **Enter**, then type your password.

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-node1** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-node1]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.11/24`. If it doesn't, run `sudo ip addr add 192.168.56.11/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.11
```

✅ **CHECK:** the prompt now reads `labadmin@tp-node1:~$`. The new name shows up because you logged in fresh.

**[PowerShell window → connected to tp-node1]** confirm everything took:
```bash
hostname
ip -br a
```

✅ **CHECK:**
- `hostname` prints `tp-node1`
- The `enp0s8` line shows `192.168.56.11/24`, and the temporary `.101` address is gone

⚠️ **HEADS UP:** If `enp0s8` still also shows a `192.168.56.101` address, the last line of the permanent-address box never ran. Paste this and press **Enter**, then run `ip -br a` again:
```bash
sudo netplan apply
```

**[PowerShell window → connected to tp-node1]** turn it off. It isn't needed until Week 2:
```bash
sudo shutdown -h now
```

✅ **CHECK:** the session closes on its own and you're back at `PS C:\`. In VirtualBox, tp-node1 shows **Powered Off**.

---

#### 2.3C tp-node2 → becomes `192.168.56.12`

tp-node2 is a server behind the door. Its final address is **`192.168.56.12`**.

📍 **WHERE YOU SHOULD BE:** In VirtualBox. tp-proxy is still running, which is fine and needed.

1. Select **tp-node2** and click **Start**.

🛑 **CHECK the title bar** of the window that opens. It must say **tp-node2 [Running] - Oracle VM VirtualBox**. With several VM windows open, it's easy to type into the wrong one.

2. At the `login:` prompt, type `labadmin`, press **Enter**, then type your password and press **Enter**.

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`. It still says `tp-base` because this is a copy. You rename it in a minute.

**[VirtualBox window → tp-node2]** give it its final address right now, so you know exactly where to connect. This one is short, so type it:
```bash
sudo ip addr add 192.168.56.12/24 dev enp0s8
```

Type your password if it asks.

**[VirtualBox window → tp-node2]** check it:
```bash
ip -br a
```

✅ **CHECK:** the `enp0s8` line shows `192.168.56.12/24`.

⚠️ **HEADS UP:** That line may also show a second address like `192.168.56.101/24`. That's a temporary one from VirtualBox. Ignore it. It disappears later in this section.

**[VirtualBox window → tp-node2]** make sure it can reach tp-proxy:
```bash
ping -c 2 192.168.56.10
```

✅ **CHECK:** two lines that say `64 bytes from 192.168.56.10`.
🛑 If it says `0 received`, check that **tp-proxy** shows **Running** in VirtualBox. If it's off, start it, wait a minute, and ping again.

🛑 **STOP: Switch windows now.** Leave the VirtualBox window alone. If you don't have a PowerShell window open, press the **Windows key**, type `PowerShell`, and press **Enter**.

✅ **CHECK:** the title bar says **Windows PowerShell**, and the prompt starts with `PS C:\`. 🛑 If the prompt starts with `labadmin@` instead, you're already inside a VM. Close this window with the **X** and open a fresh PowerShell.

**[PowerShell window → Windows]** connect to tp-node2:
```powershell
ssh labadmin@192.168.56.12
```

⚠️ **HEADS UP:** It asks `Are you sure you want to continue connecting (yes/no/[fingerprint])?` That's normal. Type `yes` and press **Enter**, then type your password.

⚠️ **HEADS UP:** If it says `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`, your PC remembers an older VM at this address. Run these two lines in the same PowerShell window:
```powershell
ssh-keygen -R 192.168.56.12
ssh labadmin@192.168.56.12
```

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-node2** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-node2]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.12/24`. If it doesn't, run `sudo ip addr add 192.168.56.12/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.12
```

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`, and the window's title bar **still says PowerShell**. 🛑 If the prompt shows the name of a VM you already finished, like `tp-node1`, you typed the address step into the wrong VirtualBox window. Turn that VM off with `sudo shutdown -h now`, run `ssh-keygen -R` with this section's address in PowerShell, and restart this section. You're connected over SSH, and from here on **right-click pastes**.

From here, copy each box and right-click to paste it. Nothing needs replacing.

**[PowerShell window → connected to tp-node2]** give it its name:
```bash
sudo hostnamectl set-hostname tp-node2
sudo sed -i 's/tp-base/tp-node2/g' /etc/hosts
```

It asks for your password once.

**[PowerShell window → connected to tp-node2]** give it a new machine ID, like a new fingerprint:
```bash
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
```

✅ **CHECK:** it prints `Initializing machine ID from random generator.`

**[PowerShell window → connected to tp-node2]** give it new SSH keys, like its own house key:
```bash
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
```

✅ **CHECK:** the middle command prints lines starting with `Creating SSH2`. You stay connected.

**[PowerShell window → connected to tp-node2]** make `192.168.56.12` permanent, so it survives a reboot. Paste this whole box at once:
```bash
sudo tee /etc/netplan/60-hostonly.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.12/24
EOF
sudo chmod 600 /etc/netplan/60-hostonly.yaml
sudo netplan apply
```

⚠️ **HEADS UP: It may look like nothing is happening. That's normal.** Press **Enter** once, wait about 10 seconds, then match what you see:

| What you see | What to do |
|---|---|
| The prompt comes back as `labadmin@...:~$` | Type `exit` and press **Enter** |
| `Connection reset` or `closed`, and the prompt reads `PS C:\` | Nothing, you're already out |
| Still nothing | Type `~.`, a tilde then a period. The prompt changes to `PS C:\` |

✅ **CHECK:** you're at `PS C:\`.

**[PowerShell window → Windows]** reconnect. The first line clears your PC's memory of the **old** house key, because you just gave tp-node2 a new one:
```powershell
ssh-keygen -R 192.168.56.12
ssh labadmin@192.168.56.12
```

⚠️ **HEADS UP:** It asks the `yes/no/[fingerprint]` question again. Type `yes`, press **Enter**, then type your password.

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-node2** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-node2]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.12/24`. If it doesn't, run `sudo ip addr add 192.168.56.12/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.12
```

✅ **CHECK:** the prompt now reads `labadmin@tp-node2:~$`. The new name shows up because you logged in fresh.

**[PowerShell window → connected to tp-node2]** confirm everything took:
```bash
hostname
ip -br a
```

✅ **CHECK:**
- `hostname` prints `tp-node2`
- The `enp0s8` line shows `192.168.56.12/24`, and the temporary `.101` address is gone

⚠️ **HEADS UP:** If `enp0s8` still also shows a `192.168.56.101` address, the last line of the permanent-address box never ran. Paste this and press **Enter**, then run `ip -br a` again:
```bash
sudo netplan apply
```

**[PowerShell window → connected to tp-node2]** turn it off. It isn't needed until Week 2:
```bash
sudo shutdown -h now
```

✅ **CHECK:** the session closes on its own and you're back at `PS C:\`. In VirtualBox, tp-node2 shows **Powered Off**.

---

#### 2.3D tp-db → becomes `192.168.56.13`

tp-db is the database server. Its final address is **`192.168.56.13`**.

📍 **WHERE YOU SHOULD BE:** In VirtualBox. tp-proxy is still running, which is fine and needed.

1. Select **tp-db** and click **Start**.

🛑 **CHECK the title bar** of the window that opens. It must say **tp-db [Running] - Oracle VM VirtualBox**. With several VM windows open, it's easy to type into the wrong one.

2. At the `login:` prompt, type `labadmin`, press **Enter**, then type your password and press **Enter**.

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`. It still says `tp-base` because this is a copy. You rename it in a minute.

**[VirtualBox window → tp-db]** give it its final address right now, so you know exactly where to connect. This one is short, so type it:
```bash
sudo ip addr add 192.168.56.13/24 dev enp0s8
```

Type your password if it asks.

**[VirtualBox window → tp-db]** check it:
```bash
ip -br a
```

✅ **CHECK:** the `enp0s8` line shows `192.168.56.13/24`.

⚠️ **HEADS UP:** That line may also show a second address like `192.168.56.101/24`. That's a temporary one from VirtualBox. Ignore it. It disappears later in this section.

**[VirtualBox window → tp-db]** make sure it can reach tp-proxy:
```bash
ping -c 2 192.168.56.10
```

✅ **CHECK:** two lines that say `64 bytes from 192.168.56.10`.
🛑 If it says `0 received`, check that **tp-proxy** shows **Running** in VirtualBox. If it's off, start it, wait a minute, and ping again.

🛑 **STOP: Switch windows now.** Leave the VirtualBox window alone. If you don't have a PowerShell window open, press the **Windows key**, type `PowerShell`, and press **Enter**.

✅ **CHECK:** the title bar says **Windows PowerShell**, and the prompt starts with `PS C:\`. 🛑 If the prompt starts with `labadmin@` instead, you're already inside a VM. Close this window with the **X** and open a fresh PowerShell.

**[PowerShell window → Windows]** connect to tp-db:
```powershell
ssh labadmin@192.168.56.13
```

⚠️ **HEADS UP:** It asks `Are you sure you want to continue connecting (yes/no/[fingerprint])?` That's normal. Type `yes` and press **Enter**, then type your password.

⚠️ **HEADS UP:** If it says `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`, your PC remembers an older VM at this address. Run these two lines in the same PowerShell window:
```powershell
ssh-keygen -R 192.168.56.13
ssh labadmin@192.168.56.13
```

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-db** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-db]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.13/24`. If it doesn't, run `sudo ip addr add 192.168.56.13/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.13
```

✅ **CHECK:** the prompt reads `labadmin@tp-base:~$`, and the window's title bar **still says PowerShell**. 🛑 If the prompt shows the name of a VM you already finished, like `tp-node1`, you typed the address step into the wrong VirtualBox window. Turn that VM off with `sudo shutdown -h now`, run `ssh-keygen -R` with this section's address in PowerShell, and restart this section. You're connected over SSH, and from here on **right-click pastes**.

From here, copy each box and right-click to paste it. Nothing needs replacing.

**[PowerShell window → connected to tp-db]** give it its name:
```bash
sudo hostnamectl set-hostname tp-db
sudo sed -i 's/tp-base/tp-db/g' /etc/hosts
```

It asks for your password once.

**[PowerShell window → connected to tp-db]** give it a new machine ID, like a new fingerprint:
```bash
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
```

✅ **CHECK:** it prints `Initializing machine ID from random generator.`

**[PowerShell window → connected to tp-db]** give it new SSH keys, like its own house key:
```bash
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
```

✅ **CHECK:** the middle command prints lines starting with `Creating SSH2`. You stay connected.

**[PowerShell window → connected to tp-db]** make `192.168.56.13` permanent, so it survives a reboot. Paste this whole box at once:
```bash
sudo tee /etc/netplan/60-hostonly.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.13/24
EOF
sudo chmod 600 /etc/netplan/60-hostonly.yaml
sudo netplan apply
```

⚠️ **HEADS UP: It may look like nothing is happening. That's normal.** Press **Enter** once, wait about 10 seconds, then match what you see:

| What you see | What to do |
|---|---|
| The prompt comes back as `labadmin@...:~$` | Type `exit` and press **Enter** |
| `Connection reset` or `closed`, and the prompt reads `PS C:\` | Nothing, you're already out |
| Still nothing | Type `~.`, a tilde then a period. The prompt changes to `PS C:\` |

✅ **CHECK:** you're at `PS C:\`.

**[PowerShell window → Windows]** reconnect. The first line clears your PC's memory of the **old** house key, because you just gave tp-db a new one:
```powershell
ssh-keygen -R 192.168.56.13
ssh labadmin@192.168.56.13
```

⚠️ **HEADS UP:** It asks the `yes/no/[fingerprint]` question again. Type `yes`, press **Enter**, then type your password.

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Click the **tp-db** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-db]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.13/24`. If it doesn't, run `sudo ip addr add 192.168.56.13/24 dev enp0s8` and check again.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.13
```

✅ **CHECK:** the prompt now reads `labadmin@tp-db:~$`. The new name shows up because you logged in fresh.

**[PowerShell window → connected to tp-db]** confirm everything took:
```bash
hostname
ip -br a
```

✅ **CHECK:**
- `hostname` prints `tp-db`
- The `enp0s8` line shows `192.168.56.13/24`, and the temporary `.101` address is gone

⚠️ **HEADS UP:** If `enp0s8` still also shows a `192.168.56.101` address, the last line of the permanent-address box never ran. Paste this and press **Enter**, then run `ip -br a` again:
```bash
sudo netplan apply
```

**[PowerShell window → connected to tp-db]** turn it off. It isn't needed until Week 2:
```bash
sudo shutdown -h now
```

✅ **CHECK:** the session closes on its own and you're back at `PS C:\`. In VirtualBox, tp-db shows **Powered Off**.

---

## Phase 3: Teach the name `teleport.lab.internal`

Right now your PC only knows the front door by its number, `192.168.56.10`. This phase gives it a name, like adding a contact to your phone so you can call by name instead of number.

### 3.1 Teach your Windows PC the name

📍 **WHERE YOU SHOULD BE:** On your Windows desktop. tp-proxy is running in VirtualBox.

1. Press the **Windows key** and type `PowerShell`.
2. **Right-click Windows PowerShell** and choose **Run as administrator**. Click **Yes**.

✅ **CHECK:** the prompt reads `PS C:\WINDOWS\system32>`.

**[Admin PowerShell window → Windows]**
```powershell
Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value "`n192.168.56.10  teleport.lab.internal"
```

**[Admin PowerShell window → Windows]** test it:
```powershell
ping -n 2 teleport.lab.internal
```

✅ **CHECK:** the replies say `Reply from 192.168.56.10`.

If it says `could not find host`, the line didn't save. Close this window, open PowerShell with **Run as administrator** again, and rerun both commands above.

You can close this Admin window now.

### 3.2 Teach tp-proxy the same name

📍 **WHERE YOU SHOULD BE:** In a regular PowerShell window at `PS C:\`. If you don't have one, press the **Windows key**, type `PowerShell`, and press **Enter**.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.10
```

⚠️ **HEADS UP: If it says `Connection timed out`, that's common in this lab and fixable.** Your PC is still pointing this address at an old network card. Do these in order:

1. Click the **tp-proxy** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-proxy]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.10/24`. If it shows a different address, stop and get help.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear the address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.10
```

✅ **CHECK:** it connects and asks for your password. If it still times out, stop and get help.

Type your password.

✅ **CHECK:** the prompt reads `labadmin@tp-proxy:~$`.

**[PowerShell window → connected to tp-proxy]**
```bash
echo "192.168.56.10  teleport.lab.internal" | sudo tee -a /etc/hosts
getent hosts teleport.lab.internal
```

✅ **CHECK:** the last line prints `192.168.56.10  teleport.lab.internal`.

**[PowerShell window → connected to tp-proxy]** leave the session:
```bash
exit
```

✅ **CHECK:** you're back at `PS C:\`.

---

## Phase 4: Make the trusted certificate

This is the badge that proves the front door is really `teleport.lab.internal`. Your mkcert badge office from Phase 0 signs it, so your browser trusts it.

### 4.1 Make the certificate on Windows

📍 **WHERE YOU SHOULD BE:** In a regular PowerShell window at `PS C:\`.

**[PowerShell window → Windows]**
```powershell
New-Item -ItemType Directory -Path D:\teleport-lab\certs -Force
cd D:\teleport-lab\certs
mkcert -cert-file teleport.lab.internal.pem -key-file teleport.lab.internal-key.pem teleport.lab.internal "*.teleport.lab.internal"
Copy-Item "$(mkcert -CAROOT)\rootCA.pem" .\rootCA.pem
Get-ChildItem
```

⚠️ **HEADS UP:** If it says `mkcert is not recognized`, close PowerShell, open a new one, and paste the box again.

✅ **CHECK:** the list at the end shows exactly these three files:
- `rootCA.pem`: your badge office's public certificate
- `teleport.lab.internal.pem`: the front door's badge
- `teleport.lab.internal-key.pem`: the badge's private key

🛑 **STOP: Never copy `rootCA-key.pem` anywhere.** That's the master key to your badge office. It stays in mkcert's own folder on your PC.

### 4.2 Send the three files to tp-proxy

📍 **WHERE YOU SHOULD BE:** Same PowerShell window. The prompt reads `PS D:\teleport-lab\certs>`.

**[PowerShell window → Windows]**
```powershell
scp .\teleport.lab.internal.pem .\teleport.lab.internal-key.pem .\rootCA.pem labadmin@192.168.56.10:~/
```

⚠️ **HEADS UP: If it says `Connection timed out`, that's common in this lab and fixable.** Your PC is still pointing this address at an old network card. Do these in order:

1. Click the **tp-proxy** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-proxy]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.10/24`. If it shows a different address, stop and get help.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear the address book, then try again from this same window:
```powershell
arp -d *
cd D:\teleport-lab\certs
scp .\teleport.lab.internal.pem .\teleport.lab.internal-key.pem .\rootCA.pem labadmin@192.168.56.10:~/
```

✅ **CHECK:** it connects and asks for your password. If it still times out, stop and get help.

Type your password.

✅ **CHECK:** three lines appear, one per file, each ending in `100%`.

### 4.3 Put the files in place on tp-proxy

📍 **WHERE YOU SHOULD BE:** Same PowerShell window.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.10
```

⚠️ **HEADS UP: If it says `Connection timed out`, that's common in this lab and fixable.** Your PC is still pointing this address at an old network card. Do these in order:

1. Click the **tp-proxy** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-proxy]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.10/24`. If it shows a different address, stop and get help.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear the address book, then try again from this same window:
```powershell
arp -d *
ssh labadmin@192.168.56.10
```

✅ **CHECK:** it connects and asks for your password. If it still times out, stop and get help.

Type your password.

✅ **CHECK:** the prompt reads `labadmin@tp-proxy:~$`.

**[PowerShell window → connected to tp-proxy]** move the badge and its key into Teleport's folder, and lock the key so only the system can read it:
```bash
sudo mkdir -p /var/lib/teleport
sudo mv ~/teleport.lab.internal.pem /var/lib/teleport/fullchain.pem
sudo mv ~/teleport.lab.internal-key.pem /var/lib/teleport/privkey.pem
sudo chmod 600 /var/lib/teleport/privkey.pem
sudo chown root:root /var/lib/teleport/fullchain.pem /var/lib/teleport/privkey.pem
```

**[PowerShell window → connected to tp-proxy]** make tp-proxy trust your badge office too. Teleport checks its own badge when it starts, so this is required:
```bash
sudo cp ~/rootCA.pem /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates
rm ~/rootCA.pem
```

✅ **CHECK:** `update-ca-certificates` prints `1 added, 0 removed`.

⚠️ **HEADS UP:** It also prints `rehash: warning: skipping ca-certificates.crt`. That's normal Ubuntu noise. `1 added` is the part that matters.

🛑 **STOP: Stay connected.** Phase 5 continues in this same window.

---

## Phase 5: Install and start Teleport

### 5.1 Install Teleport

📍 **WHERE YOU SHOULD BE:** In PowerShell, connected to tp-proxy. The prompt reads `labadmin@tp-proxy:~$`.
If you got disconnected, run `ssh labadmin@192.168.56.10` in PowerShell and type your password.

**[PowerShell window → connected to tp-proxy]**
```bash
curl https://cdn.teleport.dev/install.sh | bash -s 18.11.0
teleport version
```

⚠️ **HEADS UP:** It may ask for your password partway through. That's the installer using `sudo`.

✅ **CHECK:** the last line prints `Teleport v18.11.0`.

### 5.2 Write Teleport's settings file

📍 **WHERE YOU SHOULD BE:** Still at `labadmin@tp-proxy:~$`.

This tells Teleport its name, its address, and where its badge and key live. Paste the whole box at once:

**[PowerShell window → connected to tp-proxy]**
```bash
sudo teleport configure -o file \
    --cluster-name=teleport.lab.internal \
    --public-addr=teleport.lab.internal:443 \
    --cert-file=/var/lib/teleport/fullchain.pem \
    --key-file=/var/lib/teleport/privkey.pem
```

✅ **CHECK:** it says `A Teleport configuration file has been created at "/etc/teleport.yaml"`.

⚠️ **HEADS UP:** It also prints `WARNING: The data directory /var/lib/teleport is not empty`. That's expected. The folder only holds the two certificate files you put there in Phase 4, so there's no old Teleport data to worry about.

**[PowerShell window → connected to tp-proxy]** take a look at what it made:
```bash
sudo cat /etc/teleport.yaml
```

✅ **CHECK:** you can spot `teleport.lab.internal` and both `/var/lib/teleport/` file paths in the output.

### 5.3 Start Teleport

📍 **WHERE YOU SHOULD BE:** Still at `labadmin@tp-proxy:~$`.

**[PowerShell window → connected to tp-proxy]**
```bash
sudo systemctl enable teleport
sudo systemctl start teleport
sudo systemctl status teleport --no-pager
```

✅ **CHECK:** you see `active (running)` in green.

If you see `failed` instead, run this and send a screenshot of the output:
```bash
sudo journalctl -u teleport --no-pager -n 50
```

🛑 **STOP: Stay connected.** Phase 6 uses this same window.

---

## Phase 6: Create your admin login with phone MFA

### 6.1 Open the Teleport web page

📍 **WHERE YOU SHOULD BE:** On your Windows desktop. Leave the PowerShell window connected to tp-proxy open.

1. Open **Chrome** or **Edge**. Not Firefox.
2. Go to:
```
https://teleport.lab.internal
```

✅ **CHECK:** a padlock shows next to the address, with no warning page, and you see Teleport's **Sign in to Teleport** page.

🛑 **STOP: Don't try to sign in yet.** Your account doesn't exist until the next step. This page only proves the front door and its certificate are working.

If you see a certificate warning instead, open PowerShell with **Run as administrator**, run `mkcert -install`, then fully close the browser and open it again.

### 6.2 Create your admin user

📍 **WHERE YOU SHOULD BE:** Back in the PowerShell window at `labadmin@tp-proxy:~$`.

**[PowerShell window → connected to tp-proxy]**
```bash
sudo tctl users add keenan-admin --roles=editor,access,auditor --logins=labadmin
```

What this sets up:
- `editor` lets you manage Teleport
- `access` lets you connect to servers
- `auditor` lets you watch recordings and read the audit log
- `--logins=labadmin` means this user can log into servers **only** as the Linux account `labadmin`, never as root

✅ **CHECK:** it prints a link starting with `https://teleport.lab.internal:443/web/invite/`.

🛑 **STOP: That link expires in 1 hour.** Use it right now:
1. Highlight the whole link with your mouse.
2. Press **Ctrl + C** to copy it. If that doesn't copy, right-click instead.

### 6.3 Finish setup in the browser

📍 **WHERE YOU SHOULD BE:** In Chrome or Edge, with your phone nearby.

1. Paste the invite link into the browser's address bar and press **Enter**.
2. Set a strong password and save it in your password manager.
3. When it asks for a second factor, choose **Authenticator App**.
4. On your phone, open **Google Authenticator**, tap **+**, then **Scan a QR code**, and scan the code on screen.
5. Type the 6-digit code from your phone into the browser and finish.

✅ **CHECK:** you land on Teleport's main page, logged in as `keenan-admin`.

⚠️ **HEADS UP:** If it asks you to accept the Community Edition terms, accept them.

If the link says it expired, go back to PowerShell at `labadmin@tp-proxy:~$` and run this to get a fresh one:
```bash
sudo tctl users reset keenan-admin
```

### 6.4 Take the two proof screenshots

📍 **WHERE YOU SHOULD BE:** In the browser, logged into Teleport.

1. Click **keenan-admin** in the top right, then **Log Out**. You land on the **Sign in to Teleport** page.
2. Fill in the boxes like this:

| Box on screen | What goes in it |
|---|---|
| **Username** | `keenan-admin` |
| **Password** | The password you set from the invite link |
| **Multi-factor Type** | Leave it on **Authenticator App** |
| **Authenticator Code** | Leave it **empty** for now |

3. 📸 **CAPTURE W1-01:** screenshot this page, with the username filled in and the empty **Authenticator Code** box waiting for your phone. Your password only shows as dots.
4. On your phone, open **Google Authenticator**, find the **Teleport** entry, and type its 6 digits into **Authenticator Code**. They change every 30 seconds, so type them quickly.
5. Click **Sign In**.
6. 📸 **CAPTURE W1-02:** screenshot the **Resources** page showing `tp-proxy` as an **SSH Server**.

⚠️ **HEADS UP:** tp-proxy shows the address `127.0.0.1:3022`. That's normal. Teleport is protecting the same machine it runs on, so it reaches it from the inside.

⚠️ **HEADS UP:** `tp-proxy` shows up because Teleport also protects the front door itself. In Week 2, the other servers join this list.

### 6.5 Leave the SSH session

📍 **WHERE YOU SHOULD BE:** In the PowerShell window at `labadmin@tp-proxy:~$`.

**[PowerShell window → connected to tp-proxy]**
```bash
exit
```

✅ **CHECK:** you're back at `PS C:\`.

---

## Phase 7: Save your place

### 7.1 Turn off tp-proxy

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\`.

**[PowerShell window → Windows]**
```powershell
ssh -t labadmin@192.168.56.10 "sudo shutdown -h now"
```

⚠️ **HEADS UP: If it says `Connection timed out`, that's common in this lab and fixable.** Your PC is still pointing this address at an old network card. Do these in order:

1. Click the **tp-proxy** VirtualBox window. If it shows a `login:` prompt, log in as `labadmin`.
2. **[VirtualBox window → tp-proxy]** run `ip -br a` and ✅ **CHECK** that the `enp0s8` line shows `192.168.56.10/24`. If it shows a different address, stop and get help.
3. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
4. **[Admin PowerShell window → Windows]** clear the address book, then try again from this same window:
```powershell
arp -d *
ssh -t labadmin@192.168.56.10 "sudo shutdown -h now"
```

✅ **CHECK:** it connects and asks for your password. If it still times out, stop and get help.

Type your password when asked. It may ask twice: once to connect, once for `sudo`.

✅ **CHECK:** in VirtualBox, all four lab VMs show **Powered Off**.

### 7.2 Snapshot all four VMs

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

Do this four times, once each for **tp-proxy**, **tp-node1**, **tp-node2**, and **tp-db**:

1. Select the VM.
2. Click the menu icon next to its name and choose **Snapshots**.
3. Click **Take**.
4. Name it `week1-done` and click **OK**.

✅ **CHECK:** all four VMs have a `week1-done` snapshot. If anything goes wrong in Week 2, you can roll back to this point.

🎉 **Week 1 is done.**

---

## Troubleshooting

Each fix below is complete on its own.

### Teleport shows `failed` instead of `active (running)`

📍 In PowerShell, run `ssh labadmin@192.168.56.10` and type your password.

**[PowerShell window → connected to tp-proxy]**
```bash
sudo journalctl -u teleport --no-pager -n 50
```

Two common causes:

**A typo in a file path.** Run `sudo cat /etc/teleport.yaml` and confirm it lists `/var/lib/teleport/fullchain.pem` and `/var/lib/teleport/privkey.pem`.

**tp-proxy doesn't trust your badge office.** Resend and reinstall it.

**[PowerShell window → Windows]**
```powershell
cd D:\teleport-lab\certs
scp .\rootCA.pem labadmin@192.168.56.10:~/
ssh labadmin@192.168.56.10
```

**[PowerShell window → connected to tp-proxy]**
```bash
sudo cp ~/rootCA.pem /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates
rm ~/rootCA.pem
sudo systemctl restart teleport
sudo systemctl status teleport --no-pager
```

### The browser can't reach `https://teleport.lab.internal`

**[PowerShell window → Windows]**
```powershell
Test-NetConnection teleport.lab.internal -Port 443
```

- If it shows `TcpTestSucceeded : True`, Teleport is reachable. Fully close and reopen the browser.
- If it shows `False`, check three things: tp-proxy is running in VirtualBox, `ssh labadmin@192.168.56.10` connects, and `sudo systemctl status teleport --no-pager` shows `active (running)`.
- If it says the name can't be found, open PowerShell with **Run as administrator** and run:
```powershell
Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value "`n192.168.56.10  teleport.lab.internal"
```

### SSH says `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`

Your PC remembers a different VM at that address. ✏️ **REPLACE** the address with the one you were connecting to:

**[PowerShell window → Windows]**
```powershell
ssh-keygen -R 192.168.56.101
```

Then connect again and type `yes` at the fingerprint question.

### Two VMs ended up with the same address

Connect to the VM that has the wrong address, and paste the fixed address box from its own section in Phase 2.3 again. Each VM's box already has its correct address filled in.

---

## Lab-only shortcuts (for the Week 4 write-up)

| Shortcut in this lab | What a real company would do instead |
|---|---|
| mkcert private badge office on the host PC | A managed internal certificate authority, or a public certificate from Let's Encrypt on a real domain |
| Hosts file entries instead of DNS | Real DNS records for the cluster name and its wildcard |
| Auth and Proxy on one VM | Separate, redundant Auth and Proxy servers |
| One admin account so far | At least two admins so nobody gets locked out. Teleport's docs recommend this, and it gets folded into Week 3's roles work |
| Ubuntu password login still enabled | Direct SSH blocked entirely. That happens in Week 2 |

---

## Linux Essentials tie-ins this week

- **Package management:** `apt update`, `apt full-upgrade`, the Teleport install script
- **Services:** `systemctl enable`, `start`, `status`, and `journalctl`
- **Networking:** `ip -br a`, netplan, `/etc/hosts`, `getent`
- **Files and permissions:** `chmod 600`, `chown`, and why private keys get locked down
- **Machine identity:** hostname, machine ID, SSH host keys

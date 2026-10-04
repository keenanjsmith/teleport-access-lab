# Teleport Access Lab: Week 2 RUNBOOK

**Week 2 of 4: Put the servers behind the front door**
Teleport version: **18.11.0** (Community Edition), the same version as Week 1

---

## 🗺️ THE BIG PICTURE (read once, then start at Phase 0)

At the end of Week 1, Teleport was running on tp-proxy, but the other servers weren't connected to it. Anyone with the password could still SSH straight into them.

This week fixes that. By the end:

- **tp-node1** and **tp-node2** show up in Teleport, and you can open a terminal on them **through the Teleport web page**.
- Both servers get a firewall that **blocks all incoming connections**, including direct SSH.
- Teleport **still works** anyway, because each server connects **out** to the front door, instead of waiting for someone to connect in. That's called a reverse tunnel. Think of it like a store employee who calls headquarters every morning, instead of headquarters showing up unannounced at the door.

**The proof this week:** a direct SSH attempt to tp-node1 fails, while the same login through Teleport works.

**This week's phases:**
1. **Phase 0:** Start the VMs and make a folder for screenshots
2. **Phase 1:** Join tp-node1 to Teleport
3. **Phase 2:** Join tp-node2 to Teleport
4. **Phase 3:** Lock down tp-node1 and prove direct SSH fails
5. **Phase 4:** Lock down tp-node2
6. **Phase 5:** Save your place

**This week's VMs:**

| VM | Address | Label you give it | Running this week? |
|---|---|---|---|
| tp-proxy | `192.168.56.10` | none | Yes |
| tp-node1 | `192.168.56.11` | `env=dev` | Yes |
| tp-node2 | `192.168.56.12` | `env=prod` | Yes |
| tp-db | `192.168.56.13` | none yet | No, it waits until Week 3 |

> ⚠️ **HEADS UP: The labels set up Week 3.** `env=dev` and `env=prod` are name tags. In Week 3, you'll write a "developer" role that's only allowed into servers tagged `env=dev`. Adding the tags now saves redoing this later.

**📸 Week 2 captures, all saved to `D:\teleport-lab\teleport-access-lab\captures\week2\`:**

| ID | What it shows |
|---|---|
| W2-01 | tp-node1's Teleport service showing `active (running)` |
| W2-02 | The Resources page showing tp-proxy, tp-node1, and tp-node2 |
| W2-03 | A terminal on tp-node1, opened through the Teleport web page |
| W2-04 | tp-node1's firewall blocking all incoming connections |
| W2-05 | Direct SSH to tp-node1 from PowerShell timing out |

> ✅ **You never have to scroll back up.** Every step tells you which window to use and repeats every value it needs.

---

## 🚦 MARKERS AND LABELS

| Marker | Meaning |
|---|---|
| 📍 **WHERE YOU SHOULD BE** | Which window should be open before you start the step |
| 🛑 **STOP** | Do not move on until this is right |
| ✏️ **REPLACE** | Swap in your own value. Anything not marked REPLACE is typed exactly |
| ✅ **CHECK** | What you should see if it worked |
| 📸 **CAPTURE** | Take a screenshot now |
| ⚠️ **HEADS UP** | Something surprising that is normal |

| Label | Which window | The prompt looks like |
|---|---|---|
| **[PowerShell window → Windows]** | Windows PowerShell, **not** connected to any VM | `PS C:\Users\yourname>` |
| **[Admin PowerShell window → Windows]** | PowerShell opened with **Run as administrator** | `PS C:\WINDOWS\system32>` |
| **[PowerShell window → connected to tp-xxxx]** | The same PowerShell window, after connecting with `ssh` | `labadmin@tp-xxxx:~$` |
| **[Teleport web terminal → tp-xxxx]** | A browser tab opened from the Teleport web page. Paste works here with **Ctrl + V** | `labadmin@tp-xxxx:~$` |
| **[VirtualBox window → tp-xxxx]** | That VM's VirtualBox window. Paste doesn't work here | `labadmin@...:~$` |

> ⚠️ **HEADS UP: Paste one line at a time when a password prompt is coming.** If you paste two lines at once and the first one asks for a password, the second line can get swallowed by the password prompt and never run. Boxes that contain only Linux commands on a VM are fine to paste whole.

> 🛑 **STOP: Only run `ssh` when the prompt starts with `PS C:\`.** If it starts with `labadmin@`, you're already inside a VM, and running `ssh` there stacks one session inside another. If you lose track, close the PowerShell window with the **X** and open a fresh one.

---

## Phase 0: Get ready

### 0.1 Make sure Week 1 has a snapshot

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

1. Select **tp-proxy**, click the menu icon next to its name, and choose **Snapshots**.
2. ✅ **CHECK:** a snapshot named `week1-done` is listed.

If it isn't there, take it now for all four VMs. Each one must show **Powered Off** first:
1. Select the VM, open **Snapshots**, and click **Take**.
2. Name it `week1-done` and click **OK**.
3. Repeat for **tp-node1**, **tp-node2**, and **tp-db**.

### 0.2 Make the Week 2 screenshot folder

📍 **WHERE YOU SHOULD BE:** On your Windows desktop.

1. Press the **Windows key**, type `PowerShell`, and press **Enter**.

**[PowerShell window → Windows]**
```powershell
New-Item -ItemType Directory -Path D:\teleport-lab\teleport-access-lab\captures\week2 -Force
```

✅ **CHECK:** it lists a folder named `week2`.

Leave this PowerShell window open. You'll use it all week.

### 0.3 Start tp-proxy, tp-node1, and tp-node2

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

1. Select **tp-proxy** and click **Start**. Wait until its window shows a `login:` prompt. You don't need to log in.
2. Select **tp-node1** and click **Start**. Wait for its `login:` prompt.
3. Select **tp-node2** and click **Start**. Wait for its `login:` prompt.

✅ **CHECK:** tp-proxy, tp-node1, and tp-node2 show **Running**. tp-db stays **Powered Off**.

⚠️ **HEADS UP:** You can minimize the three VirtualBox windows. Everything this week happens in PowerShell and the browser.

### 0.4 Confirm Teleport is up

📍 **WHERE YOU SHOULD BE:** In Chrome or Edge.

1. Go to `https://teleport.lab.internal`
2. Sign in:

| Box | What goes in it |
|---|---|
| **Username** | `keenan-admin` |
| **Password** | Your Teleport password |
| **Multi-factor Type** | **Authenticator App** |
| **Authenticator Code** | The 6 digits from the **Teleport** entry in Google Authenticator |

✅ **CHECK:** you land on the **Resources** page, and it shows **tp-proxy**.

⚠️ **HEADS UP:** If the page won't load, wait one more minute. Teleport takes a moment to start after tp-proxy boots.

Leave this browser tab open. You'll come back to it.

---

## Phase 1: Join tp-node1 to Teleport

tp-node1 needs four things before it can join: the lab's name for the front door, trust in your badge office, the Teleport software, and a one-time join token. This phase does them in that order.

### 1.1 Send your badge office's certificate to tp-node1

tp-node1 has to trust your mkcert badge office, the same way tp-proxy does, or it will refuse to talk to the front door.

📍 **WHERE YOU SHOULD BE:** In the PowerShell window at `PS C:\Users\keena>`.

**[PowerShell window → Windows]**
```powershell
scp D:\teleport-lab\certs\rootCA.pem labadmin@192.168.56.11:~/
```

Type your labadmin password.

✅ **CHECK:** one line ending in `100%`.

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Check that tp-node1 shows **Running** in VirtualBox.
2. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
3. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
scp D:\teleport-lab\certs\rootCA.pem labadmin@192.168.56.11:~/
```

### 1.2 Connect to tp-node1

📍 **WHERE YOU SHOULD BE:** In PowerShell at a `PS C:\` prompt.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.11
```

Type your password.

✅ **CHECK:** the prompt reads `labadmin@tp-node1:~$`.

⚠️ **HEADS UP:** If it says `REMOTE HOST IDENTIFICATION HAS CHANGED`, run these two lines, then type `yes` and your password:
```powershell
ssh-keygen -R 192.168.56.11
ssh labadmin@192.168.56.11
```

### 1.3 Teach tp-node1 the front door's name, and trust the badge office

📍 **WHERE YOU SHOULD BE:** In PowerShell at `labadmin@tp-node1:~$`.

**[PowerShell window → connected to tp-node1]** paste the whole box:
```bash
echo "192.168.56.10  teleport.lab.internal" | sudo tee -a /etc/hosts
sudo cp ~/rootCA.pem /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates
rm ~/rootCA.pem
getent hosts teleport.lab.internal
```

It asks for your password once.

✅ **CHECK:**
- `update-ca-certificates` prints `1 added, 0 removed`
- The last line prints `192.168.56.10  teleport.lab.internal`

⚠️ **HEADS UP:** `rehash: warning: skipping ca-certificates.crt` is normal Ubuntu noise. Ignore it.

**[PowerShell window → connected to tp-node1]** test that tp-node1 can reach the front door, and trusts its certificate:
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://teleport.lab.internal/webapi/ping
```

✅ **CHECK:** it prints `200`.

🛑 If it prints `000` or a certificate error, stop. Check that tp-proxy is running, and that `1 added` appeared in the box above.

### 1.4 Install Teleport on tp-node1

📍 **WHERE YOU SHOULD BE:** Still at `labadmin@tp-node1:~$`.

**[PowerShell window → connected to tp-node1]**
```bash
curl https://cdn.teleport.dev/install.sh | bash -s 18.11.0
teleport version
```

⚠️ **HEADS UP:** This takes a minute or two and may ask for your password.

✅ **CHECK:** the last line prints `Teleport v18.11.0`.

### 1.5 Get a one-time join token from the front door

A join token is like a temporary visitor badge. tp-proxy hands one out, tp-node1 shows it once to prove it's allowed to join, and it expires after an hour.

📍 **WHERE YOU SHOULD BE:** Still at `labadmin@tp-node1:~$`.

**[PowerShell window → connected to tp-node1]** step back out to Windows:
```bash
exit
```

✅ **CHECK:** the prompt reads `PS C:\Users\keena>`.

**[PowerShell window → Windows]** ask tp-proxy for a token. This connects, prints the token, and drops you right back at `PS C:\` on its own:
```powershell
ssh -t labadmin@192.168.56.10 "sudo tctl tokens add --type=node --format=text --ttl=1h"
```

Type your labadmin password when asked. It may ask twice: once to connect, once for `sudo`.

✅ **CHECK:** it prints one long line of letters and numbers, then `Connection to 192.168.56.10 closed.`, then `PS C:\Users\keena>`. That long line is the token.

🛑 **STOP: Copy the token now.** Highlight just that long line with your mouse and press **Ctrl + C**. If that doesn't copy, right-click instead. **Don't copy anything else until the token is saved in step 1.6**, or you'll lose it.

### 1.6 Join tp-node1

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`, with the token copied.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.11
```

✅ **CHECK:** the prompt reads `labadmin@tp-node1:~$`.

**[PowerShell window → connected to tp-node1]** save the token into a variable named `TOKEN`:
1. **Type** `TOKEN=` yourself. Typing it, instead of pasting, keeps the token on your clipboard.
2. **Right-click** to paste the token right after the `=`.
3. Press **Enter**.

> **Example:** the finished line looks like `TOKEN=f37975988c7c133cfa99bdb5f81ef10f`, with your own token after the `=`.

⚠️ **HEADS UP:** No spaces anywhere in that line. It prints nothing, which is normal.

**[PowerShell window → connected to tp-node1]** check it saved:
```bash
echo $TOKEN
```

✅ **CHECK:** it prints your token. 🛑 If it prints a blank line, type `TOKEN=` and paste again.

**[PowerShell window → connected to tp-node1]** write tp-node1's Teleport settings, tagged `env=dev`. Paste the whole box:
```bash
sudo teleport node configure \
    --output=file:///etc/teleport.yaml \
    --token="$TOKEN" \
    --proxy=teleport.lab.internal:443 \
    --labels=env=dev
```

✅ **CHECK:** it says `A Teleport configuration file has been created at "/etc/teleport.yaml"`.

⚠️ **HEADS UP:** It suggests running `sudo teleport start`. **Don't use that one.** It only runs Teleport until you close the window. The next box starts it as a background service that also comes back on its own after a reboot.

**[PowerShell window → connected to tp-node1]** start it:
```bash
sudo systemctl enable teleport
sudo systemctl start teleport
sudo systemctl status teleport --no-pager
```

✅ **CHECK:** you see `active (running)` in green.

> ## 📸📸📸 SCREENSHOT TIME: W2-01 📸📸📸
> Screenshot the PowerShell window showing `active (running)`. Save it as `W2-01.png` in `D:\teleport-lab\teleport-access-lab\captures\week2\`.

🛑 If you see `failed` instead, run this and get help with the output:
```bash
sudo journalctl -u teleport --no-pager -n 50
```

**[PowerShell window → connected to tp-node1]** leave the session:
```bash
exit
```

✅ **CHECK:** you're back at `PS C:\Users\keena>`.

### 1.7 See tp-node1 in Teleport

📍 **WHERE YOU SHOULD BE:** In the browser, on the Teleport **Resources** page.

1. Click the **refresh** button on the Resources page, or press **F5**.

✅ **CHECK:** **tp-node1** now shows up next to **tp-proxy**.

⚠️ **HEADS UP:** tp-proxy shows an address, `127.0.0.1:3022`, but tp-node1 shows **no address at all**, just its `env: dev` label. That's the reverse tunnel. tp-node1 called out to the front door, so Teleport never needs its address. It's also why the firewall in Phase 3 won't break anything.

⚠️ **HEADS UP:** If tp-node1 doesn't show up, wait 30 seconds and refresh again.

---

## Phase 2: Join tp-node2 to Teleport

Same steps as Phase 1, with tp-node2's address, `192.168.56.12`, and the label `env=prod`.

### 2.1 Send your badge office's certificate to tp-node2

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`.

**[PowerShell window → Windows]**
```powershell
scp D:\teleport-lab\certs\rootCA.pem labadmin@192.168.56.12:~/
```

Type your labadmin password.

✅ **CHECK:** one line ending in `100%`.

⚠️ **HEADS UP: If it says `Connection timed out`:**
1. Check that tp-node2 shows **Running** in VirtualBox.
2. Open an Admin window: press the **Windows key**, type `PowerShell`, **right-click Windows PowerShell**, choose **Run as administrator**, and click **Yes**.
3. **[Admin PowerShell window → Windows]** clear your PC's address book, then try again from this same window:
```powershell
arp -d *
scp D:\teleport-lab\certs\rootCA.pem labadmin@192.168.56.12:~/
```

### 2.2 Connect to tp-node2

📍 **WHERE YOU SHOULD BE:** In PowerShell at a `PS C:\` prompt.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.12
```

Type your password.

✅ **CHECK:** the prompt reads `labadmin@tp-node2:~$`.

⚠️ **HEADS UP:** If it says `REMOTE HOST IDENTIFICATION HAS CHANGED`, run these two lines, then type `yes` and your password:
```powershell
ssh-keygen -R 192.168.56.12
ssh labadmin@192.168.56.12
```

### 2.3 Teach tp-node2 the front door's name, and trust the badge office

📍 **WHERE YOU SHOULD BE:** In PowerShell at `labadmin@tp-node2:~$`.

**[PowerShell window → connected to tp-node2]** paste the whole box:
```bash
echo "192.168.56.10  teleport.lab.internal" | sudo tee -a /etc/hosts
sudo cp ~/rootCA.pem /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates
rm ~/rootCA.pem
getent hosts teleport.lab.internal
```

✅ **CHECK:**
- `update-ca-certificates` prints `1 added, 0 removed`
- The last line prints `192.168.56.10  teleport.lab.internal`

**[PowerShell window → connected to tp-node2]**
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://teleport.lab.internal/webapi/ping
```

✅ **CHECK:** it prints `200`.

### 2.4 Install Teleport on tp-node2

📍 **WHERE YOU SHOULD BE:** Still at `labadmin@tp-node2:~$`.

**[PowerShell window → connected to tp-node2]**
```bash
curl https://cdn.teleport.dev/install.sh | bash -s 18.11.0
teleport version
```

✅ **CHECK:** the last line prints `Teleport v18.11.0`.

### 2.5 Get a new join token

📍 **WHERE YOU SHOULD BE:** Still at `labadmin@tp-node2:~$`.

**[PowerShell window → connected to tp-node2]**
```bash
exit
```

✅ **CHECK:** the prompt reads `PS C:\Users\keena>`.

**[PowerShell window → Windows]**
```powershell
ssh -t labadmin@192.168.56.10 "sudo tctl tokens add --type=node --format=text --ttl=1h"
```

Type your password when asked, possibly twice.

✅ **CHECK:** one long line of letters and numbers, then you're back at `PS C:\Users\keena>`.

🛑 **STOP: Copy the token now.** Highlight just that long line and press **Ctrl + C**. Don't copy anything else until step 2.6 is done.

### 2.6 Join tp-node2

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`, with the token copied.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.12
```

✅ **CHECK:** the prompt reads `labadmin@tp-node2:~$`.

**[PowerShell window → connected to tp-node2]** save the token into a variable named `TOKEN`:
1. **Type** `TOKEN=` yourself. Typing it, instead of pasting, keeps the token on your clipboard.
2. **Right-click** to paste the token right after the `=`.
3. Press **Enter**.

> **Example:** the finished line looks like `TOKEN=f37975988c7c133cfa99bdb5f81ef10f`, with your own token after the `=`.

⚠️ **HEADS UP:** No spaces anywhere in that line. It prints nothing, which is normal.

**[PowerShell window → connected to tp-node2]** check it saved:
```bash
echo $TOKEN
```

✅ **CHECK:** it prints your token. 🛑 If it prints a blank line, type `TOKEN=` and paste again.

**[PowerShell window → connected to tp-node2]** settings for tp-node2, tagged `env=prod`. Paste the whole box:
```bash
sudo teleport node configure \
    --output=file:///etc/teleport.yaml \
    --token="$TOKEN" \
    --proxy=teleport.lab.internal:443 \
    --labels=env=prod
```

✅ **CHECK:** it says `A Teleport configuration file has been created at "/etc/teleport.yaml"`.

⚠️ **HEADS UP:** Ignore its suggestion to run `sudo teleport start`. Use the next box instead.

**[PowerShell window → connected to tp-node2]**
```bash
sudo systemctl enable teleport
sudo systemctl start teleport
sudo systemctl status teleport --no-pager
```

✅ **CHECK:** `active (running)` in green.

**[PowerShell window → connected to tp-node2]**
```bash
exit
```

✅ **CHECK:** you're back at `PS C:\Users\keena>`.

### 2.7 See all three servers

📍 **WHERE YOU SHOULD BE:** In the browser, on the Teleport **Resources** page.

1. Refresh the page with **F5**.

✅ **CHECK:** **tp-proxy**, **tp-node1**, and **tp-node2** are all listed.

> ## 📸📸📸 SCREENSHOT TIME: W2-02 📸📸📸
> Screenshot the Resources page showing all three servers. Save it as `W2-02.png` in `captures\week2\`.

---

## Phase 3: Lock down tp-node1, and prove direct SSH fails

From here on, you manage tp-node1 **only through Teleport**. That's the whole point of the lab.

### 3.1 Open a terminal on tp-node1 through Teleport

📍 **WHERE YOU SHOULD BE:** In the browser, on the Teleport **Resources** page.

1. Find **tp-node1** and click **Connect**.
2. Choose **labadmin**.

✅ **CHECK:** a new browser tab opens with a terminal, and the prompt reads `labadmin@tp-node1:~$`.

**[Teleport web terminal → tp-node1]**
```bash
hostname
```

✅ **CHECK:** it prints `tp-node1`.

> ## 📸📸📸 SCREENSHOT TIME: W2-03 📸📸📸
> Screenshot this browser tab, showing the Teleport terminal on tp-node1. Save it as `W2-03.png` in `captures\week2\`.

⚠️ **HEADS UP:** Paste works in this browser terminal. Use **Ctrl + V**, or right-click and choose **Paste**.

### 3.2 Turn on the firewall

This blocks **every** incoming connection to tp-node1, including SSH. Teleport keeps working, because tp-node1's tunnel goes **out** to the front door, and the firewall only blocks traffic coming **in**.

📍 **WHERE YOU SHOULD BE:** In the Teleport web terminal tab, at `labadmin@tp-node1:~$`.

**[Teleport web terminal → tp-node1]** paste the whole box:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw --force enable
sudo ufw status verbose
```

It asks for your labadmin password once.

✅ **CHECK:** the output includes:
- `Status: active`
- `Default: deny (incoming), allow (outgoing)`

✅ **CHECK:** you're **still connected** in this browser tab. That proves Teleport doesn't need any door left open on tp-node1.

> ## 📸📸📸 SCREENSHOT TIME: W2-04 📸📸📸
> Screenshot this tab showing `Status: active` and `deny (incoming)`. Save it as `W2-04.png` in `captures\week2\`.

⚠️ **HEADS UP:** If you ever lock yourself out completely, tp-node1's **VirtualBox window** still works. It's like a keyboard plugged straight into the server, so the firewall can't block it. Log in there and run `sudo ufw disable` to undo the firewall.

Leave this browser tab open.

### 3.3 Prove direct SSH fails

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`.

**[PowerShell window → Windows]** try to SSH straight into tp-node1, the old way. The `ConnectTimeout=10` part makes it give up after 10 seconds instead of waiting:
```powershell
ssh -o ConnectTimeout=10 labadmin@192.168.56.11
```

✅ **CHECK:** after about 10 seconds, it prints `Connection timed out`, and you're back at `PS C:\Users\keena>`.

> ## 📸📸📸 SCREENSHOT TIME: W2-05 📸📸📸
> Screenshot the PowerShell window showing `Connection timed out`. If you can, arrange it next to the browser tab that's still connected to tp-node1 through Teleport, so both show in one shot. Save it as `W2-05.png` in `captures\week2\`.

**That's this week's proof.** The old door is locked, and the only way in is through Teleport.

---

## Phase 4: Lock down tp-node2

Same as Phase 3, without the screenshots.

### 4.1 Open a terminal on tp-node2 through Teleport

📍 **WHERE YOU SHOULD BE:** In the browser. Click back to the tab with the Teleport **Resources** page.

1. Find **tp-node2** and click **Connect**.
2. Choose **labadmin**.

✅ **CHECK:** a new tab opens, and the prompt reads `labadmin@tp-node2:~$`.

### 4.2 Turn on the firewall

**[Teleport web terminal → tp-node2]** paste the whole box:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw --force enable
sudo ufw status verbose
```

✅ **CHECK:** `Status: active` and `Default: deny (incoming), allow (outgoing)`, and you're still connected.

### 4.3 Confirm direct SSH fails

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`.

**[PowerShell window → Windows]**
```powershell
ssh -o ConnectTimeout=10 labadmin@192.168.56.12
```

✅ **CHECK:** `Connection timed out`, and you're back at `PS C:\Users\keena>`.

---

## Phase 5: Save your place

### 5.1 Turn off tp-node1 and tp-node2 through Teleport

📍 **WHERE YOU SHOULD BE:** In the browser.

1. Click the **tp-node1** terminal tab.

**[Teleport web terminal → tp-node1]**
```bash
sudo shutdown -h now
```

✅ **CHECK:** the tab shows the session ended. You can close the tab.

2. Click the **tp-node2** terminal tab.

**[Teleport web terminal → tp-node2]**
```bash
sudo shutdown -h now
```

✅ **CHECK:** the tab shows the session ended. You can close the tab.

### 5.2 Turn off tp-proxy

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`.

**[PowerShell window → Windows]**
```powershell
ssh -t labadmin@192.168.56.10 "sudo shutdown -h now"
```

Type your password when asked, possibly twice.

✅ **CHECK:** in VirtualBox, all four VMs show **Powered Off**.

### 5.3 Snapshot all four VMs

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

Do this four times, once each for **tp-proxy**, **tp-node1**, **tp-node2**, and **tp-db**:

1. Select the VM.
2. Click the menu icon next to its name and choose **Snapshots**.
3. Click **Take**.
4. Name it `week2-done` and click **OK**.

✅ **CHECK:** all four VMs have a `week2-done` snapshot.

✅ **CHECK:** `D:\teleport-lab\teleport-access-lab\captures\week2\` holds `W2-01.png` through `W2-05.png`.

🎉 **Week 2 is done.**

---

## Troubleshooting

Each fix below is complete on its own.

### tp-node1 or tp-node2 doesn't show up on the Resources page

1. Wait 30 seconds and refresh with **F5**.
2. If it's still missing, open that VM's **VirtualBox window**, log in as `labadmin`, and run:

**[VirtualBox window → tp-node1 or tp-node2]**
```bash
sudo journalctl -u teleport --no-pager -n 30
```

| If the output mentions | Then |
|---|---|
| `token` and `expired` or `not found` | The join token expired or was mistyped. Redo the "Get a join token" and "Join" steps for that VM |
| `certificate` or `x509` | That VM doesn't trust your badge office yet. Redo the "Teach the front door's name, and trust the badge office" step for that VM |
| `no such host` or `teleport.lab.internal` | The hosts file line is missing. Redo the same step |

### The token step says `sudo: a terminal is required`

The `-t` is missing from the command. It must start with `ssh -t`:
```powershell
ssh -t labadmin@192.168.56.10 "sudo tctl tokens add --type=node --format=text --ttl=1h"
```

### I locked myself out of tp-node1 or tp-node2

1. Open that VM's **VirtualBox window**. The firewall can't block it.
2. Log in as `labadmin`.
3. Run `sudo ufw disable`. That turns the firewall off, so you can fix things and turn it back on later.

---

## Lab-only shortcuts (for the Week 4 write-up)

| Shortcut in this lab | What a real company would do instead |
|---|---|
| mkcert private badge office on the host PC | A managed internal certificate authority, or a public certificate on a real domain |
| Hosts file entries instead of DNS | Real DNS records for the cluster name |
| Auth and Proxy on one VM | Separate, redundant Auth and Proxy servers |
| Join tokens copied by hand | Automatic joining, like cloud identity join methods, so no token ever gets copied |
| tp-proxy still accepts direct SSH, for setup | The Teleport servers themselves would also be locked down, and reached through Teleport or a break-glass process |
| Local Linux password still exists on each VM | Accounts created on demand by Teleport, with no standing passwords |

---

## Linux Essentials tie-ins this week

- **Copying files between machines:** `scp`
- **Variables:** `TOKEN=...` saves a value, and `echo $TOKEN` reads it back
- **Firewalls:** `ufw default`, `ufw enable`, `ufw status verbose`
- **Services:** `systemctl enable`, `start`, `status`, and `journalctl`
- **Name lookup:** `/etc/hosts` and `getent`
- **Trust stores:** `update-ca-certificates` and `/usr/local/share/ca-certificates/`
- **Network testing:** `curl` with `-w "%{http_code}"` to check whether a web service answers

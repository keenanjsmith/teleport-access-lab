# Teleport Access Lab: Week 3 RUNBOOK

**Week 3 of 4: Roles, a denied attempt, session recording, and the database**
Teleport version: **18.11.0** (Community Edition), the same version as Weeks 1 and 2

---

## 🗺️ THE BIG PICTURE (read once, then start at Part A)

Right now, the only Teleport user is you, **keenan-admin**, and you can reach everything. Real companies don't work that way. A developer should reach the dev server, and nothing else.

This week has two parts.

### Part A: Roles, a denied attempt, and session recording

1. You write a **developer** role. It's a short rule that says "only servers tagged `env=dev`, and only as the Linux account `dev`."
2. You create a second Teleport user, **dev-user**, with only that role.
3. You log in as dev-user and see that **tp-node2 doesn't even show up**.
4. As dev-user, you try to read a protected file on tp-node1 and get **Permission denied**.
5. You switch back to keenan-admin and **play back a recording** of everything dev-user just did.

### Part B: The database

1. You install **PostgreSQL** on tp-db and put a small table in it.
2. You give PostgreSQL its own certificate from Teleport, so it only accepts connections that Teleport vouches for.
3. You connect tp-db to Teleport, then **query the database straight from your browser**.
4. You lock tp-db down with the same firewall as Week 2, and show that Teleport logged your database query.

> ⚠️ **HEADS UP: Part B is the hardest part of the whole lab.** If you're short on time, you can stop after Part A and still have a complete story. Part B can come after your Linux Essentials exam.

**This week's VMs:**

| VM | Address | Role this week |
|---|---|---|
| tp-proxy | `192.168.56.10` | The front door |
| tp-node1 | `192.168.56.11` | Dev server, `env=dev`. dev-user is allowed here |
| tp-node2 | `192.168.56.12` | Prod server, `env=prod`. dev-user is **not** allowed here |
| tp-db | `192.168.56.13` | PostgreSQL database, Part B only |

**The two Teleport users:**

| User | Roles | Can reach |
|---|---|---|
| `keenan-admin` | editor, access, auditor | Everything |
| `dev-user` | developer | tp-node1 only, as the Linux account `dev` |

**📸 Week 3 captures, all saved to `D:\teleport-lab\teleport-access-lab\captures\week3\`:**

| ID | What it shows |
|---|---|
| W3-01 | The developer role, written as code |
| W3-02 | dev-user's Resources page, showing only tp-node1 |
| W3-03 | dev-user on tp-node1, getting `Permission denied` |
| W3-04 | keenan-admin playing back dev-user's recorded session |
| W3-05 | The audit log showing dev-user's session |
| W3-06 | A database query running in the browser through Teleport |
| W3-07 | The audit log showing that database query |

> ✅ **You never have to scroll back up.** Every step tells you which window to use and repeats every value it needs.

---

## 🚦 MARKERS AND LABELS

| Marker | Meaning |
|---|---|
| 📍 **WHERE YOU SHOULD BE** | Which window should be open before you start the step |
| 🛑 **STOP** | Do not move on until this is right |
| ✏️ **REPLACE** | Swap in your own value. Anything not marked REPLACE is typed exactly |
| ✅ **CHECK** | What you should see if it worked |
| 📸 **SCREENSHOT TIME** | Take a screenshot now |
| ⚠️ **HEADS UP** | Something surprising that is normal |

| Label | Which window | The prompt looks like |
|---|---|---|
| **[PowerShell window → Windows]** | Windows PowerShell, **not** connected to any VM | `PS C:\Users\yourname>` |
| **[Admin PowerShell window → Windows]** | PowerShell opened with **Run as administrator** | `PS C:\WINDOWS\system32>` |
| **[PowerShell window → connected to tp-xxxx]** | The same PowerShell window, after connecting with `ssh` | `labadmin@tp-xxxx:~$` |
| **[Teleport web terminal → tp-xxxx]** | A browser tab opened from the Teleport web page. Paste with **Ctrl + V** | `labadmin@tp-xxxx:~$` or `dev@tp-xxxx:~$` |
| **[VirtualBox window → tp-xxxx]** | That VM's VirtualBox window. Paste doesn't work here | `labadmin@...:~$` |

> ⚠️ **HEADS UP: Paste one line at a time when a password prompt is coming.** If you paste two lines at once and the first one asks for a password, the second line can get swallowed by the password prompt and never run. Boxes that contain only Linux commands on a VM are fine to paste whole.

> 🛑 **STOP: Only run `ssh` when the prompt starts with `PS C:\`.** If it starts with `labadmin@`, you're already inside a VM. If you lose track, close the PowerShell window with the **X** and open a fresh one.

> ⚠️ **HEADS UP: Two browser windows this week.** keenan-admin stays in your **normal** browser window. dev-user goes in a **private** window, called **InPrivate** in Edge or **Incognito** in Chrome. A private window keeps the two logins from bumping each other off.

---

# PART A: Roles, a denied attempt, and session recording

## Phase A0: Get ready

### A0.1 Make the Week 3 screenshot folder

📍 **WHERE YOU SHOULD BE:** On your Windows desktop.

1. Press the **Windows key**, type `PowerShell`, and press **Enter**.

**[PowerShell window → Windows]**
```powershell
New-Item -ItemType Directory -Path D:\teleport-lab\teleport-access-lab\captures\week3 -Force
```

✅ **CHECK:** it lists a folder named `week3`.

Leave this PowerShell window open.

### A0.2 Start tp-proxy, tp-node1, and tp-node2

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

1. Select **tp-proxy** and click **Start**. Wait for its `login:` prompt. You don't need to log in.
2. Select **tp-node1** and click **Start**. Wait for its `login:` prompt.
3. Select **tp-node2** and click **Start**. Wait for its `login:` prompt.

✅ **CHECK:** tp-proxy, tp-node1, and tp-node2 show **Running**. tp-db stays **Powered Off** until Part B.

You can minimize the VirtualBox windows.

### A0.3 Sign in as keenan-admin

📍 **WHERE YOU SHOULD BE:** In your **normal** Chrome or Edge window.

1. Go to `https://teleport.lab.internal`
2. Sign in:

| Box | What goes in it |
|---|---|
| **Username** | `keenan-admin` |
| **Password** | Your keenan-admin password |
| **Multi-factor Type** | **Authenticator App** |
| **Authenticator Code** | The 6 digits under **keenan-admin** in Google Authenticator |

✅ **CHECK:** the **Resources** page shows **tp-node1**, **tp-node2**, and **tp-proxy**.

⚠️ **HEADS UP:** If tp-node1 or tp-node2 is missing, wait a minute and press **F5**. They reconnect on their own after booting.

---

## Phase A1: Make a limited Linux account on tp-node1

dev-user won't log in as `labadmin`, because `labadmin` can run `sudo` and do anything. Instead, they get a plain account named `dev`, with no admin powers.

### A1.1 Open a terminal on tp-node1 as keenan-admin

📍 **WHERE YOU SHOULD BE:** In your normal browser window, on the **Resources** page.

1. On the **tp-node1** card, click **Connect**, then choose **labadmin**.

✅ **CHECK:** a new tab opens at `labadmin@tp-node1:~$`.

### A1.2 Create the `dev` account

**[Teleport web terminal → tp-node1]** paste with **Ctrl + V**:
```bash
sudo adduser --disabled-password --gecos "" dev
id dev
```

It asks for your labadmin password once.

✅ **CHECK:** the last line starts with `uid=` and mentions `dev`, with **no** `sudo` group in it.

⚠️ **HEADS UP:** `--disabled-password` means `dev` has no Linux password at all. The only way in is through Teleport. That's on purpose.

**[Teleport web terminal → tp-node1]** close the session:
```bash
exit
```

Close this tab. You're back on the **Resources** tab.

---

## Phase A2: Write the developer role

A role is a short rulebook written in a format called YAML. Think of it like a keycard program: it lists which doors open, and for which badge.

### A2.1 Connect to tp-proxy

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\Users\keena>`.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.10
```

Type your labadmin password.

✅ **CHECK:** the prompt reads `labadmin@tp-proxy:~$`.

⚠️ **HEADS UP:** If it says `Connection timed out`, open PowerShell with **Run as administrator**, run `arp -d *`, then run the `ssh` line again in that Admin window.

### A2.2 Write the role file

**[PowerShell window → connected to tp-proxy]** paste the whole box:
```bash
cat > developer.yaml <<'EOF'
kind: role
version: v7
metadata:
  name: developer
  description: Developers can reach dev servers only, as the dev account
spec:
  allow:
    logins: ['dev']
    node_labels:
      'env': 'dev'
  deny: {}
EOF
cat developer.yaml
```

✅ **CHECK:** the file prints back with the same lines you pasted.

What each part means:

| Line | In plain English |
|---|---|
| `logins: ['dev']` | You can only log in as the Linux account `dev` |
| `node_labels: 'env': 'dev'` | You can only see and reach servers tagged `env=dev` |
| `deny: {}` | No extra "never" rules. Anything not allowed is already blocked |

### A2.3 Load the role into Teleport

**[PowerShell window → connected to tp-proxy]**
```bash
sudo tctl create -f developer.yaml
sudo tctl get roles/developer
```

✅ **CHECK:** the first command prints `role "developer" has been created`, and the second prints the role back.

⚠️ **HEADS UP:** Teleport prints the role in its own tidier style, so it won't look exactly like your file. Look for these lines in the middle:
```
  allow:
    logins:
    - dev
    node_labels:
      env: dev
```
That's the same rule you wrote. The long `options:` section underneath is just Teleport filling in its default settings. One of them, `record_session`, is what records dev-user's session for Phase A5.

> ## 📸📸📸 SCREENSHOT TIME: W3-01 📸📸📸
> Screenshot the PowerShell window showing `role 'developer' has been created` and the role printed back. Save it as `W3-01.png` in `captures\week3\`.

### A2.4 Create dev-user

**[PowerShell window → connected to tp-proxy]**
```bash
sudo tctl users add dev-user --roles=developer
```

✅ **CHECK:** it prints a link starting with `https://teleport.lab.internal:443/web/invite/`.

🛑 **STOP: The link expires in 1 hour.** Highlight the whole link and press **Ctrl + C**. Then go straight to Phase A3.

Leave this PowerShell window connected. You'll use it again in Part B.

---

## Phase A3: Set up dev-user and see what they can reach

### A3.1 Open a private browser window

📍 **WHERE YOU SHOULD BE:** In your browser.

1. Open a private window:
   - **Edge:** press **Ctrl + Shift + N** for an **InPrivate** window
   - **Chrome:** press **Ctrl + Shift + N** for an **Incognito** window

✅ **CHECK:** the new window says **InPrivate** or **Incognito** near the top.

### A3.2 Finish dev-user's setup

📍 **WHERE YOU SHOULD BE:** In the **private** window.

1. Paste the invite link into the address bar and press **Enter**.
2. Set a password for dev-user and save it in your password manager.
3. If it asks for a second factor type, choose **Authenticator App**.
4. On your phone, open **Google Authenticator**, tap **+**, choose **Scan a QR code**, and scan it.
5. Type the 6 digits from the **new** entry, the one that says **dev-user**.

⚠️ **HEADS UP:** Your phone now has two Teleport entries. Each shows its username underneath, `keenan-admin` or `dev-user`. Always use the one that matches who you're logging in as.

✅ **CHECK:** you land on the **Resources** page, logged in as **dev-user** in the top right.

### A3.3 See what dev-user can reach

📍 **WHERE YOU SHOULD BE:** In the private window, on dev-user's **Resources** page.

✅ **CHECK:** **only tp-node1** is listed. tp-node2 and tp-proxy don't appear at all.

That's the role working. dev-user isn't told "no" to tp-node2. As far as dev-user can tell, tp-node2 doesn't exist.

> ## 📸📸📸 SCREENSHOT TIME: W3-02 📸📸📸
> Screenshot dev-user's Resources page showing only tp-node1, with **dev-user** visible in the top right. Save it as `W3-02.png` in `captures\week3\`.

---

## Phase A4: The denied attempt

### A4.1 Connect to tp-node1 as dev-user

📍 **WHERE YOU SHOULD BE:** In the private window, on dev-user's **Resources** page.

1. On the **tp-node1** card, click **Connect**.
2. Choose **dev**. It's the only login offered.

✅ **CHECK:** a new tab opens at `dev@tp-node1:~$`.

### A4.2 Try to read a protected file

`/etc/shadow` is where Linux keeps password information. Only admins can read it.

**[Teleport web terminal → tp-node1 as dev]** paste with **Ctrl + V**:
```bash
whoami
hostname
cat /etc/shadow
ls /root
```

✅ **CHECK:**
- `whoami` prints `dev`
- `hostname` prints `tp-node1`
- `cat /etc/shadow` prints `Permission denied`
- `ls /root` prints `Permission denied`

> ## 📸📸📸 SCREENSHOT TIME: W3-03 📸📸📸
> Screenshot this tab showing `dev`, `tp-node1`, and both `Permission denied` lines. Save it as `W3-03.png` in `captures\week3\`.

**[Teleport web terminal → tp-node1 as dev]** end the session:
```bash
exit
```

⚠️ **HEADS UP: Always end with `exit`.** The recording is kept on tp-node1 while the session runs, and only gets handed to the front door when the session ends cleanly. If the connection drops instead, for example if tp-proxy freezes, the recording may not show up in Phase A5. If that happens, just redo Phase A4 from the top and end with `exit`.

Close this tab, then close the whole private window.

---

## Phase A5: Play back the recording

Teleport recorded everything dev-user just typed and saw, without dev-user doing anything special. Now you'll watch it as the admin.

### A5.1 Find the recording

📍 **WHERE YOU SHOULD BE:** In your **normal** browser window, signed in as **keenan-admin**.

1. In the left menu, click **Audit**.
2. Click **Session Recordings**.

✅ **CHECK:** the newest recording at the top shows user **dev-user**, on **tp-node1**, as login **dev**.

⚠️ **HEADS UP:** If it's not listed yet, wait 30 seconds and refresh. Recordings upload when the session ends. If it still isn't there, the session probably didn't end cleanly. Redo Phase A4, ending with `exit`.

### A5.2 Play it

1. Click **Play** on dev-user's recording.
2. Let it play through the `cat /etc/shadow` line.

✅ **CHECK:** you watch dev-user's commands appear, including both `Permission denied` lines.

> ## 📸📸📸 SCREENSHOT TIME: W3-04 📸📸📸
> Pause the playback where `Permission denied` shows, and screenshot it. Save it as `W3-04.png` in `captures\week3\`.

### A5.3 Check the audit log

1. In the left menu, click **Audit**, then **Audit Log**.

✅ **CHECK:** near the top, you see events for **dev-user**, such as a session starting and ending on tp-node1, plus dev-user's login.

> ## 📸📸📸 SCREENSHOT TIME: W3-05 📸📸📸
> Screenshot the audit log showing dev-user's events. Save it as `W3-05.png` in `captures\week3\`.

### A5.4 Add a backup admin

Teleport recommends a second admin account, so a lost phone can't lock you out of your own cluster.

📍 **WHERE YOU SHOULD BE:** In the PowerShell window still connected to tp-proxy, at `labadmin@tp-proxy:~$`. If you got disconnected, run `ssh labadmin@192.168.56.10` from a `PS C:\` prompt.

**[PowerShell window → connected to tp-proxy]**
```bash
sudo tctl users add keenan-backup --roles=editor,access,auditor --logins=labadmin
```

✅ **CHECK:** it prints an invite link.

1. Copy the link.
2. Open a new private window with **Ctrl + Shift + N**.
3. Paste the link, set a password, and scan the QR code with Google Authenticator.
4. Close the private window.

✅ **CHECK:** Google Authenticator now has three Teleport entries: `keenan-admin`, `dev-user`, and `keenan-backup`.

🎉 **Part A is done.** If you're stopping here, skip to **Part C** to shut down and save your place.

---

# PART B: The database

## Phase B0: Start tp-db and connect to it

### B0.1 Start tp-db

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

1. Select **tp-db** and click **Start**.

🛑 **CHECK the title bar** of the window that opens. It must say **tp-db [Running] - Oracle VM VirtualBox**.

2. Wait for its `login:` prompt. You don't need to log in.

### B0.2 Copy the badge office's certificate to tp-db, then connect

tp-db needs your badge office's certificate, `rootCA.pem`. That file lives on your Windows PC, so you copy it over **from Windows first**, then connect.

📍 **WHERE YOU SHOULD BE:** In PowerShell at a `PS C:\` prompt. If your window still shows `labadmin@tp-proxy:~$`, type `exit` first.

**[PowerShell window → Windows]** copy the file. Paste this line by itself, because it asks for a password:
```powershell
scp D:\teleport-lab\certs\rootCA.pem labadmin@192.168.56.13:~/
```

✅ **CHECK:** one line ending in `100%`, then you're back at `PS C:\`.

**[PowerShell window → Windows]** then connect:
```powershell
ssh labadmin@192.168.56.13
```

✅ **CHECK:** the prompt reads `labadmin@tp-db:~$`.

⚠️ **HEADS UP: If it says `Connection timed out`:** tp-node2 briefly used `.13` back in Week 1, so your PC may still point that address at tp-node2's network card.
1. Open PowerShell with **Run as administrator**.
2. **[Admin PowerShell window → Windows]**
```powershell
arp -d *
scp D:\teleport-lab\certs\rootCA.pem labadmin@192.168.56.13:~/
ssh labadmin@192.168.56.13
```

⚠️ **HEADS UP:** If it says `REMOTE HOST IDENTIFICATION HAS CHANGED`, run this, then the two lines above again:
```powershell
ssh-keygen -R 192.168.56.13
```

---

## Phase B1: Prepare tp-db

### B1.1 Name the front door, and trust the badge office

This is the same prep you did for tp-node1 and tp-node2 in Week 2.

📍 **WHERE YOU SHOULD BE:** In PowerShell at `labadmin@tp-db:~$`.

**[PowerShell window → connected to tp-db]** paste the whole box:
```bash
echo "192.168.56.10  teleport.lab.internal" | sudo tee -a /etc/hosts
sudo cp ~/rootCA.pem /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates
rm ~/rootCA.pem
curl -s -o /dev/null -w "%{http_code}\n" https://teleport.lab.internal/webapi/ping
```

✅ **CHECK:** `1 added, 0 removed`, and the last line prints `200`.

### B1.2 Install PostgreSQL

**[PowerShell window → connected to tp-db]**
```bash
sudo apt update
sudo apt install -y postgresql
ls /etc/postgresql
```

⚠️ **HEADS UP:** This takes a few minutes.

✅ **CHECK:** the last line prints `16`. That's the PostgreSQL version, and every path below uses it.

🛑 If it prints a different number, stop and get help before continuing. The paths below would need that number instead.

### B1.3 Create a small database to query

This makes a database named `labdb`, with one table listing your lab servers, and a read-only database user named `labreader`.

**[PowerShell window → connected to tp-db]** paste the whole box:
```bash
sudo -u postgres psql <<'EOF'
CREATE DATABASE labdb;
\c labdb
CREATE TABLE servers (name text, address text, environment text);
INSERT INTO servers VALUES
  ('tp-proxy', '192.168.56.10', 'core'),
  ('tp-node1', '192.168.56.11', 'dev'),
  ('tp-node2', '192.168.56.12', 'prod'),
  ('tp-db',    '192.168.56.13', 'data');
CREATE USER labreader;
GRANT CONNECT ON DATABASE labdb TO labreader;
GRANT SELECT ON servers TO labreader;
EOF
```

✅ **CHECK:** the output includes `CREATE DATABASE`, `INSERT 0 4`, `CREATE ROLE`, and two `GRANT` lines.

⚠️ **HEADS UP:** `labreader` has no password. Like the `dev` Linux account, the only way in is through Teleport.

---

## Phase B2: Give PostgreSQL a Teleport certificate

PostgreSQL will only accept a connection if Teleport has signed for it, like a bouncer who only takes IDs from one specific office. For that, PostgreSQL needs its own certificate from Teleport.

### B2.1 Make the certificate on tp-proxy

📍 **WHERE YOU SHOULD BE:** In PowerShell at `labadmin@tp-db:~$`.

**[PowerShell window → connected to tp-db]** step out to Windows:
```bash
exit
```

✅ **CHECK:** the prompt starts with `PS C:\`.

**[PowerShell window → Windows]** ask tp-proxy to make the certificate. This connects, makes three files, and drops you back at `PS C:\` on its own:
```powershell
ssh -t labadmin@192.168.56.10 "sudo tctl auth sign --format=db --host=localhost,tp-db --out=/home/labadmin/server --ttl=2190h && sudo chown labadmin:labadmin /home/labadmin/server.*"
```

Type your password when asked, possibly twice.

✅ **CHECK:** it mentions writing `server.crt`, `server.key`, and `server.cas`, then you're back at `PS C:\`.

### B2.2 Carry the three files from tp-proxy to tp-db

**[PowerShell window → Windows]** first, pull them onto your PC, into the private `certs` folder:
```powershell
scp labadmin@192.168.56.10:/home/labadmin/server.crt labadmin@192.168.56.10:/home/labadmin/server.key labadmin@192.168.56.10:/home/labadmin/server.cas D:\teleport-lab\certs\
```

✅ **CHECK:** three lines, each ending in `100%`.

🛑 **STOP:** `server.key` is a private key. It stays in `D:\teleport-lab\certs\`, which never goes to GitHub.

**[PowerShell window → Windows]** then push them to tp-db:
```powershell
scp D:\teleport-lab\certs\server.crt D:\teleport-lab\certs\server.key D:\teleport-lab\certs\server.cas labadmin@192.168.56.13:~/
```

✅ **CHECK:** three lines, each ending in `100%`.

**[PowerShell window → Windows]** remove the copies left on tp-proxy:
```powershell
ssh labadmin@192.168.56.10 "rm /home/labadmin/server.crt /home/labadmin/server.key /home/labadmin/server.cas"
```

### B2.3 Put the certificate in place on tp-db

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.13
```

✅ **CHECK:** `labadmin@tp-db:~$`

**[PowerShell window → connected to tp-db]** move the files where PostgreSQL can read them, and lock the key down. Paste the whole box:
```bash
sudo mkdir -p /var/lib/postgresql/teleport
sudo mv ~/server.crt ~/server.key ~/server.cas /var/lib/postgresql/teleport/
sudo chown -R postgres:postgres /var/lib/postgresql/teleport
sudo chmod 600 /var/lib/postgresql/teleport/server.key
sudo ls -l /var/lib/postgresql/teleport
```

✅ **CHECK:** three files are listed, all owned by `postgres`, and `server.key` shows `-rw-------`.

### B2.4 Tell PostgreSQL to use it

**[PowerShell window → connected to tp-db]** turn on encryption with the new certificate. Paste the whole box:
```bash
sudo tee /etc/postgresql/16/main/conf.d/teleport.conf > /dev/null <<'EOF'
ssl = on
ssl_cert_file = '/var/lib/postgresql/teleport/server.crt'
ssl_key_file = '/var/lib/postgresql/teleport/server.key'
ssl_ca_file = '/var/lib/postgresql/teleport/server.cas'
EOF
```

**[PowerShell window → connected to tp-db]** require a Teleport certificate for every network connection. These two lines go at the **very top** of the access file, because PostgreSQL uses the first rule that matches:
```bash
sudo sed -i '1i hostssl all all 127.0.0.1/32 cert' /etc/postgresql/16/main/pg_hba.conf
sudo sed -i '1i hostssl all all ::1/128 cert' /etc/postgresql/16/main/pg_hba.conf
sudo head -2 /etc/postgresql/16/main/pg_hba.conf
```

✅ **CHECK:** the two printed lines are:
```
hostssl all all ::1/128 cert
hostssl all all 127.0.0.1/32 cert
```

**[PowerShell window → connected to tp-db]** restart PostgreSQL and confirm it picked up the certificate:
```bash
sudo systemctl restart postgresql
sudo -u postgres psql -c "SHOW ssl_cert_file;"
```

✅ **CHECK:** it prints `/var/lib/postgresql/teleport/server.crt`.

🛑 If the restart fails, run this and get help with the output:
```bash
sudo journalctl -u postgresql@16-main --no-pager -n 30
```

---

## Phase B3: Connect tp-db to Teleport

### B3.1 Install Teleport on tp-db

📍 **WHERE YOU SHOULD BE:** In PowerShell at `labadmin@tp-db:~$`.

**[PowerShell window → connected to tp-db]**
```bash
curl https://cdn.teleport.dev/install.sh | bash -s 18.11.0
teleport version
```

✅ **CHECK:** `Teleport v18.11.0`.

### B3.2 Get a database join token

This token is a different type from Week 2's. It says "this machine is joining as a **database** connector," not a server.

**[PowerShell window → connected to tp-db]**
```bash
exit
```

✅ **CHECK:** the prompt starts with `PS C:\`.

**[PowerShell window → Windows]**
```powershell
ssh -t labadmin@192.168.56.10 "sudo tctl tokens add --type=db --format=text --ttl=1h"
```

✅ **CHECK:** one long line of letters and numbers, then you're back at `PS C:\`.

🛑 **STOP:** Highlight just the token and press **Ctrl + C**. Don't copy anything else until the token is saved in the next step.

### B3.3 Join tp-db

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.13
```

✅ **CHECK:** `labadmin@tp-db:~$`

**[PowerShell window → connected to tp-db]** save the token:
1. **Type** `TOKEN=` yourself.
2. **Right-click** to paste the token right after the `=`.
3. Press **Enter**.

> **Example:** the finished line looks like `TOKEN=f37975988c7c133cfa99bdb5f81ef10f`, with your own token after the `=`.

**[PowerShell window → connected to tp-db]**
```bash
echo $TOKEN
```

✅ **CHECK:** it prints your token. 🛑 If it's blank, type `TOKEN=` and paste again.

**[PowerShell window → connected to tp-db]** write tp-db's Teleport settings. Paste the whole box:
```bash
sudo teleport db configure create \
    -o file \
    --token="$TOKEN" \
    --proxy=teleport.lab.internal:443 \
    --name=lab-postgres \
    --protocol=postgres \
    --uri=localhost:5432 \
    --labels=env=data
```

✅ **CHECK:** it says a configuration file was written to `/etc/teleport.yaml`.

⚠️ **HEADS UP:** Ignore any suggestion to run `sudo teleport start`. Use the next box instead.

**[PowerShell window → connected to tp-db]**
```bash
sudo systemctl enable teleport
sudo systemctl start teleport
sudo systemctl status teleport --no-pager
```

✅ **CHECK:** `active (running)` in green.

**[PowerShell window → connected to tp-db]**
```bash
exit
```

✅ **CHECK:** you're back at `PS C:\`.

---

## Phase B4: Query the database from your browser

### B4.1 Allow keenan-admin to use the database

Right now, even keenan-admin isn't allowed to use the database, because nobody has said which database user and database name they may use.

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\`.

**[PowerShell window → Windows]**
```powershell
ssh -t labadmin@192.168.56.10 "sudo tctl users update keenan-admin --set-db-users=labreader --set-db-names=labdb"
```

✅ **CHECK:** it says user `keenan-admin` has been updated.

### B4.2 Sign out and back in

The new permission only kicks in on a fresh login.

📍 **WHERE YOU SHOULD BE:** In your normal browser window, signed in as keenan-admin.

1. Click **keenan-admin** in the top right, then **Log Out**.
2. Sign back in as `keenan-admin`, with the code from the **keenan-admin** entry in Google Authenticator.

✅ **CHECK:** the **Resources** page now shows **lab-postgres** as a database, alongside the three servers.

⚠️ **HEADS UP:** If lab-postgres is missing, wait 30 seconds and press **F5**.

### B4.3 Connect and run a query

1. On the **lab-postgres** card, click **Connect**.
2. If it asks for a database user, choose `labreader`. For the database name, choose `labdb`.
3. Click **Connect** in that box.

✅ **CHECK:** a new browser tab opens that says `Teleport PostgreSQL interactive shell` and `Connected to "lab-postgres" instance as "labreader" user.`, with a prompt reading `labdb=>`.

⚠️ **HEADS UP:** This browser shell arrived in Teleport 17.1. If yours opens a box showing `tsh` commands instead, you're on an older version.

**[Teleport database tab → lab-postgres]** type this and press **Enter**:
```sql
SELECT * FROM servers;
```

✅ **CHECK:** a table with four rows: tp-proxy, tp-node1, tp-node2, and tp-db.

> ## 📸📸📸 SCREENSHOT TIME: W3-06 📸📸📸
> Screenshot the browser tab showing the query and its four rows. Save it as `W3-06.png` in `captures\week3\`.

### B4.4 See the query in the audit log

1. Go back to the main Teleport tab.
2. In the left menu, click **Audit**, then **Audit Log**.

✅ **CHECK:** near the top are two events:
- **Database Session Started:** `User [keenan-admin] has connected to database [labdb] as [labreader] on [lab-postgres]`
- **Database Query:** `User [keenan-admin] has executed query [SELECT * FROM servers;] in database [labdb] on [lab-postgres]`

> ## 📸📸📸 SCREENSHOT TIME: W3-07 📸📸📸
> Screenshot the audit log entry showing the database query. If clicking the event shows its details, include those too. Save it as `W3-07.png` in `captures\week3\`.

### B4.5 Confirm dev-user still can't see the database

1. Open a private window with **Ctrl + Shift + N**.
2. Go to `https://teleport.lab.internal` and sign in as **dev-user**, with the **dev-user** code.

✅ **CHECK:** the Resources page still shows **only tp-node1**. No lab-postgres, because the developer role never mentions databases.

Close the private window.

---

## Phase B5: Lock down tp-db

Same firewall as Week 2. Teleport keeps working, because tp-db's connector calls **out** to the front door, and PostgreSQL only listens on tp-db itself.

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\`.

**[PowerShell window → Windows]**
```powershell
ssh labadmin@192.168.56.13
```

✅ **CHECK:** `labadmin@tp-db:~$`

**[PowerShell window → connected to tp-db]** paste the whole box. The last line turns the firewall on, which cuts off this very connection:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw --force enable
```

⚠️ **HEADS UP: You'll probably stay connected.** The firewall blocks **new** incoming connections, but lets a connection that was already open finish. If it does freeze instead, press **Enter**, then type `~.`.

**[PowerShell window → connected to tp-db]** leave tp-db, so you can test from the outside:
```bash
exit
```

✅ **CHECK:** the prompt starts with `PS C:\`.

🛑 **STOP: Run the next test from Windows, not from inside tp-db.** If the prompt still says `labadmin@tp-db`, you'd be testing tp-db against itself, which proves nothing.

**[PowerShell window → Windows]** prove direct SSH is now blocked:
```powershell
ssh -o ConnectTimeout=10 labadmin@192.168.56.13
```

✅ **CHECK:** `Connection timed out`.

**Prove the database still works through Teleport:** in the browser, go back to the **lab-postgres** tab, or connect again from Resources, and run `SELECT * FROM servers;` once more.

✅ **CHECK:** the four rows come back.

⚠️ **HEADS UP:** If you ever need to undo tp-db's firewall, its VirtualBox window still works. Log in there and run `sudo ufw disable`.

🎉 **Part B is done.**

---

# PART C: Save your place

### C1 Turn off tp-node1 and tp-node2 through Teleport

📍 **WHERE YOU SHOULD BE:** In your normal browser window, signed in as keenan-admin, on the **Resources** page.

1. On **tp-node1**, click **Connect**, then **labadmin**.

**[Teleport web terminal → tp-node1]**
```bash
sudo shutdown -h now
```

2. Back on Resources, on **tp-node2**, click **Connect**, then **labadmin**.

**[Teleport web terminal → tp-node2]**
```bash
sudo shutdown -h now
```

✅ **CHECK:** both tabs show the session ended. Close them.

### C2 Turn off tp-db, if you did Part B

tp-db's firewall blocks SSH, and it has no Teleport terminal, so you'll use its VirtualBox window.

📍 **WHERE YOU SHOULD BE:** In the **tp-db** VirtualBox window.

🛑 **CHECK the title bar:** it says **tp-db [Running]**.

1. Log in as `labadmin`.

**[VirtualBox window → tp-db]**
```bash
sudo shutdown -h now
```

### C3 Turn off tp-proxy

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS C:\`.

**[PowerShell window → Windows]**
```powershell
ssh -t labadmin@192.168.56.10 "sudo shutdown -h now"
```

Type your password when asked, possibly twice.

✅ **CHECK:** in VirtualBox, all four VMs show **Powered Off**.

### C4 Snapshot all four

📍 **WHERE YOU SHOULD BE:** In VirtualBox.

For each of **tp-proxy**, **tp-node1**, **tp-node2**, and **tp-db**:
1. Select it, click the menu icon next to its name, and choose **Snapshots**.
2. Click **Take**, name it `week3-done`, and click **OK**.

✅ **CHECK:** all four have a `week3-done` snapshot.

✅ **CHECK:** `captures\week3\` holds W3-01 through W3-05, plus W3-06 and W3-07 if you did Part B.

🎉 **Week 3 is done.**

---

## Troubleshooting

Each fix below is complete on its own.

### dev-user sees no servers at all

The role's label must exactly match tp-node1's label. In PowerShell, connect with `ssh labadmin@192.168.56.10`, then run:

**[PowerShell window → connected to tp-proxy]**
```bash
sudo tctl get roles/developer
```

Under `node_labels`, it must say `env: dev`. If it doesn't, redo the role file from Phase A2.2, then run `sudo tctl create -f developer.yaml --force` to replace it.

### dev-user can connect, but gets `Failed to launch` or a login error

The Linux account `dev` doesn't exist on tp-node1. As keenan-admin, connect to tp-node1 as **labadmin** and run:
```bash
sudo adduser --disabled-password --gecos "" dev
```

### lab-postgres doesn't show up

In tp-db's **VirtualBox window**, log in as `labadmin` and run:
```bash
sudo journalctl -u teleport --no-pager -n 30
```

| If the output mentions | Then |
|---|---|
| `token` | The token expired or was mistyped. Redo Phase B3.2 and B3.3 |
| `x509` or `certificate` | tp-db doesn't trust the badge office. Redo Phase B1.1 |

### The database connection fails with a certificate or authentication error

1. In tp-db's **VirtualBox window**, run `sudo head -2 /etc/postgresql/16/main/pg_hba.conf`. Both lines must start with `hostssl` and end with `cert`.
2. Run `sudo -u postgres psql -c "SHOW ssl;"`. It must print `on`.
3. If keenan-admin isn't offered `labreader` or `labdb`, redo Phase B4.1 and sign out and back in.

---

## Lab-only shortcuts (for the Week 4 write-up)

| Shortcut in this lab | What a real company would do instead |
|---|---|
| mkcert private badge office on the host PC | A managed internal certificate authority, or a public certificate on a real domain |
| Hosts file entries instead of DNS | Real DNS records |
| Auth and Proxy on one VM | Separate, redundant Auth and Proxy servers |
| Join tokens copied by hand | Automatic joining, like cloud identity join methods |
| The `dev` Linux account made by hand | Teleport creates accounts on demand when someone logs in, and removes them after |
| Database user `labreader` made by hand | Teleport creates database users on demand, with only the permissions the role allows |
| Database certificate valid for about 3 months, renewed by hand | Automatic renewal before it expires |
| Users created locally in Teleport | Users come from the company's single sign-on, like Okta or Entra ID. That's a paid Teleport feature |
| tp-proxy still accepts direct SSH, for setup | The Teleport servers are also locked down and reached through a break-glass process |

---

## Linux Essentials tie-ins this week

- **Users and permissions:** `adduser`, `id`, `whoami`, and why `cat /etc/shadow` is denied
- **File ownership and modes:** `chown -R`, `chmod 600`, and reading `ls -l` output
- **Package management:** `apt install postgresql`
- **Services:** `systemctl restart postgresql` and `journalctl -u`
- **Editing files from the command line:** `tee`, heredocs with `<<'EOF'`, and `sed -i '1i ...'` to insert a line at the top of a file
- **Viewing files:** `cat` and `head`

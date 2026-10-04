# Teleport Access Lab: Week 4 RUNBOOK

**Week 4 of 4: Put the finished lab on GitHub**

---

## 🗺️ THE BIG PICTURE (read once, then start at Phase 0)

The build is done. This week you:

1. **Phase 0:** Drop the final files into the repo folder
2. **Phase 1:** Do a safety check, so no private keys or secrets go public
3. **Phase 2:** Create an empty repo on GitHub
4. **Phase 3:** Upload the folder with `git`
5. **Phase 4:** Check that it looks right on GitHub

> ✅ **You never have to scroll back up.** Every step tells you which window to use and repeats every value it needs.

| Marker | Meaning |
|---|---|
| 📍 **WHERE YOU SHOULD BE** | Which window should be open before you start the step |
| 🛑 **STOP** | Do not move on until this is right |
| ✏️ **REPLACE** | Swap in your own value |
| ✅ **CHECK** | What you should see if it worked |
| ⚠️ **HEADS UP** | Something surprising that is normal |

---

## Phase 0: Drop in the final files

📍 **WHERE YOU SHOULD BE:** In File Explorer.

Save each file to the exact spot below. Replace any old copy with the same name.

| File | Save it to |
|---|---|
| `README.md` | `D:\teleport-lab\teleport-access-lab\` |
| `.gitignore` | `D:\teleport-lab\teleport-access-lab\` |
| `architecture.svg` | `D:\teleport-lab\teleport-access-lab\diagrams\` |
| `architecture.png` | `D:\teleport-lab\teleport-access-lab\diagrams\` |
| `week1-front-door.md` | `D:\teleport-lab\teleport-access-lab\runbooks\` |
| `week4-write-up-and-publish.md` | `D:\teleport-lab\teleport-access-lab\runbooks\` |

⚠️ **HEADS UP:** Windows sometimes hides files that start with a dot, like `.gitignore`. If you can't see it after saving, that's fine. Phase 1 checks that it's there.

✅ **CHECK:** your folder looks like this:
```
D:\teleport-lab\teleport-access-lab\
├── .gitignore
├── BUILD-LOG.md
├── README.md
├── captures\
│   ├── week1\   W1-01.png, W1-02.png
│   ├── week2\   W2-01.png to W2-05.png
│   └── week3\   W3-01.png to W3-07.png
├── diagrams\    architecture.svg, architecture.png
└── runbooks\    week1 to week4 .md files
```

---

## Phase 1: Safety check

Once something is on GitHub, assume it's public forever. This phase makes sure nothing secret goes up.

### 1.1 Check for key and certificate files

📍 **WHERE YOU SHOULD BE:** On your Windows desktop.

1. Press the **Windows key**, type `PowerShell`, and press **Enter**.

**[PowerShell window → Windows]**
```powershell
cd D:\teleport-lab\teleport-access-lab
Get-ChildItem -Recurse -Force -Include *.pem,*.key,*.crt,*.cas
```

✅ **CHECK:** it prints **nothing at all**.

🛑 **STOP:** If it lists any file, move that file into `D:\teleport-lab\certs\` before going on. Your private keys belong there, and that folder never goes to GitHub.

### 1.2 Check that `.gitignore` is in place

**[PowerShell window → Windows]**
```powershell
Get-Content .gitignore
```

✅ **CHECK:** it prints a list that includes `*.pem`, `*.key`, `*.crt`, and `*.cas`.

### 1.3 Look over your screenshots

Open each screenshot in `captures\` and make sure none of them shows:
- An **invite link**, the long `https://teleport.lab.internal:443/web/invite/...` address
- A **join token**, the long line of letters and numbers
- A **password**

✅ **CHECK:** none do. Your passwords only ever showed as dots, and your invite links and tokens were never in a capture.

---

## Phase 2: Create the empty repo on GitHub

📍 **WHERE YOU SHOULD BE:** In your browser, signed in to GitHub.

1. Go to `https://github.com/new`
2. Fill in:

| Field | What goes in it |
|---|---|
| **Repository name** | `teleport-access-lab` |
| **Description** | `Teleport home lab: MFA, short-lived certificates, least-privilege roles, session recording, and audited database access` |
| **Public or Private** | **Public** |
| **Add a README file** | 🛑 Leave **unchecked** |
| **Add .gitignore** | 🛑 Leave as **None** |
| **Choose a license** | Leave as **None** |

🛑 **STOP:** Leave all three "add" options off. You already have a README and a `.gitignore`, and GitHub's versions would clash with yours.

3. Click **Create repository**.

✅ **CHECK:** GitHub shows a mostly empty page titled **Quick setup**. Leave it open.

---

## Phase 3: Upload the folder

📍 **WHERE YOU SHOULD BE:** In PowerShell at `PS D:\teleport-lab\teleport-access-lab>`. If you're somewhere else, run `cd D:\teleport-lab\teleport-access-lab` first.

### 3.1 Turn the folder into a git repo, and stage everything

**[PowerShell window → Windows]**
```powershell
git init
git add .
git status
```

✅ **CHECK:** `git status` lists files in green under **Changes to be committed**, including `README.md`, `BUILD-LOG.md`, `.gitignore`, the `captures`, `diagrams`, and `runbooks` folders, and **no** `.pem` or `.key` files.

### 3.2 Save a snapshot of everything

**[PowerShell window → Windows]**
```powershell
git commit -m "Teleport access lab: three-week build, runbooks, build log, and write-up"
```

✅ **CHECK:** it prints a summary like `XX files changed`.

⚠️ **HEADS UP:** If it says `Please tell me who you are`, run these two lines with your own name and GitHub email, then run the `git commit` line again:
```powershell
git config --global user.name "Keenan Smith"
git config --global user.email "your-github-email@example.com"
```
✏️ **REPLACE** `your-github-email@example.com` with the email on your GitHub account.

### 3.3 Send it to GitHub

**[PowerShell window → Windows]** paste these **one line at a time**, because the last one may open a sign-in window:
```powershell
git branch -M main
```
```powershell
git remote add origin https://github.com/keenanjsmith/teleport-access-lab.git
```
```powershell
git push -u origin main
```

⚠️ **HEADS UP:** If a GitHub sign-in window pops up, sign in and approve it.

✅ **CHECK:** the last lines include `main -> main` and `branch 'main' set up to track 'origin/main'`.

---

## Phase 4: Check it on GitHub

📍 **WHERE YOU SHOULD BE:** In your browser.

1. Go to `https://github.com/keenanjsmith/teleport-access-lab` and press **F5**.

✅ **CHECK:**
- The README shows on the front page, with the **architecture diagram** near the top
- In the **What This Lab Proves** table, clicking **W1-01** opens your screenshot
- The **Rebuild It Yourself** links open each week's runbook
- There are **no** `.pem` or `.key` files anywhere in the file list

⚠️ **HEADS UP:** If a screenshot link shows a 404 page, the image file name doesn't match exactly. GitHub cares about capital letters, so `W1-01.png` and `w1-01.png` count as different names. Rename the file to match the README, then run these from `PS D:\teleport-lab\teleport-access-lab>`:
```powershell
git add .
git commit -m "Fix screenshot file name"
git push
```

🎉 **The Teleport Access Lab is live.**

---

## Making changes later

Any time you edit a file in `D:\teleport-lab\teleport-access-lab\`, upload the change from PowerShell:

**[PowerShell window → Windows]**
```powershell
cd D:\teleport-lab\teleport-access-lab
git add .
git commit -m "Describe what you changed"
git push
```

✏️ **REPLACE** `Describe what you changed` with a short note, like `Add Week 3 screenshot markup`.

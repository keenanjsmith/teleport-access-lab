# Build Log

Real problems hit during the build, and what actually fixed them. Each fix has also been folded back into the runbook.

---

## Entry 1

**STEP:** Week 1, Phase 2.3, giving each clone its own identity

**WHAT I WAS DOING:** Typing the rename, machine ID, and SSH key commands into the tp-proxy VirtualBox window.

**WHAT HAPPENED:** Paste did not work in the VirtualBox window, even with bidirectional clipboard turned on. While retyping by hand, I put the tp-proxy IP where `machine-id` belonged, so the machine ID commands failed with "command not found."

**WHAT I TRIED:** Retyping the commands and trying variations of the machine ID command.

**WHAT ACTUALLY FIXED IT:** Ubuntu Server has no desktop, so the VirtualBox clipboard can never reach it. I used the VirtualBox window only for short commands, then connected over SSH from Windows PowerShell, where right-click paste works.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 2

**STEP:** Week 1, Phase 2.3B, tp-node1 identity

**WHAT I WAS DOING:** Connecting from PowerShell to tp-node1's temporary address, 192.168.56.101.

**WHAT HAPPENED:** `ssh` returned "Connection timed out," even though `ip -br a` in the VM showed 192.168.56.101.

**WHAT I TRIED:** Confirmed the address was right. Tried pinging my PC from the VM, which also failed.

**WHAT ACTUALLY FIXED IT:** tp-proxy had used .101 earlier, so Windows still matched that address to tp-proxy's old network card. Running `arp -d *` in an Admin PowerShell cleared it. The VM-to-PC ping failing was a red herring, since Windows Firewall blocks pings by default.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 3

**STEP:** Week 1, Phase 2.3C and 2.3D, connecting to the remaining clones

**WHAT I WAS DOING:** Following the runbook's temporary address method for tp-node2.

**WHAT HAPPENED:** Every clone kept getting the same temporary address, .101, because they all started with the same machine ID. Guessing other addresses like .102 timed out, and the runbook's example numbers added to the confusion.

**WHAT I TRIED:** Clearing the Windows address book, trying different addresses, and removing saved host keys.

**WHAT ACTUALLY FIXED IT:** Changed the method. In the VirtualBox window, give the VM its final address first with `sudo ip addr add 192.168.56.12/24 dev enp0s8`, then SSH straight to that address. No temporary address needed. The runbook now uses this method for every VM.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 4

**STEP:** Week 1, Phase 2.3C, tp-node2 identity

**WHAT I WAS DOING:** Connecting to tp-node2 from PowerShell to paste the identity steps.

**WHAT HAPPENED:** The prompt kept showing `labadmin@tp-base` or `labadmin@tp-node2` even after typing `exit`, and "Last login" said it came from 192.168.56.101 instead of my PC.

**WHAT I TRIED:** Typing `exit` repeatedly, and running `ssh-keygen` and `ssh` again from the same window.

**WHAT ACTUALLY FIXED IT:** I had run `ssh` from inside an existing SSH session several times, stacking sessions inside each other. Each `exit` only removes one layer. Closing the PowerShell window and opening a fresh one cleared them all. Rule going forward: only run `ssh` when the prompt starts with `PS C:\`.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 5

**STEP:** Week 1, Phase 2.3D, tp-db identity

**WHAT I WAS DOING:** Giving tp-db its address, 192.168.56.13, in the VirtualBox window.

**WHAT HAPPENED:** Connecting to .13 landed on tp-node2 instead. SSH also warned that the key for .13 matched the one saved for .12.

**WHAT I TRIED:** Caught it from the prompt name before running any tp-db steps.

**WHAT ACTUALLY FIXED IT:** With several VM windows open, I had typed the address command into tp-node2's window. Shut tp-node2 down, which also cleared the extra address, ran `ssh-keygen -R 192.168.56.13` to forget the wrong key, then redid the step in the window whose title bar said tp-db. The runbook now has a title bar check after starting each VM.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 6

**STEP:** Week 2, Phase 1.6, joining tp-node1

**WHAT I WAS DOING:** Saving the join token into a variable on tp-node1 with the runbook's `read -p "Paste the token, then press Enter: " TOKEN` line.

**WHAT HAPPENED:** I put the token where the prompt message goes, so it showed the token back to me as a question and waited. The variable never got the token.

**WHAT I TRIED:** Ran the `read` line again with the token in the same spot.

**WHAT ACTUALLY FIXED IT:** Skipped `read` entirely. Typed `TOKEN=` and pasted the token right after it, then confirmed with `echo $TOKEN`. The runbook now uses this simpler method for both servers.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 7

**STEP:** Week 3, Phases A4 and A5, dev-user's session recording

**WHAT I WAS DOING:** Running dev-user's commands on tp-node1 through the Teleport web terminal, then looking for the recording as keenan-admin.

**WHAT HAPPENED:** tp-proxy froze mid-session and the browser pages failed. After restarting tp-proxy and logging back in, dev-user's recording was missing from Session Recordings.

**WHAT I TRIED:** Restarted tp-proxy, logged back in, and checked Audit, then Session Recordings.

**WHAT ACTUALLY FIXED IT:** Teleport keeps the recording on the server during the session, and only hands it to the front door when the session ends cleanly. The freeze cut the session off, so it was never handed over. Redoing the session and ending it with `exit` produced the recording, and the audit log showed a "Session Uploaded" event confirming the handoff.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 8

**STEP:** Week 3, Parts A and B, tp-proxy stability

**WHAT I WAS DOING:** Running Week 3 with all four VMs on: tp-proxy at 2 GB, tp-node1 and tp-node2 at 1 GB each, and tp-db at 2 GB.

**WHAT HAPPENED:** tp-proxy froze three times in one evening. Later, tp-db got "Destination Host Unreachable" when pinging tp-proxy, and its connection test returned `000`.

**WHAT I TRIED:** Restarted tp-proxy each time, which only worked until the next freeze. Ran `getent`, `ping`, and `curl -sv` from tp-db, which showed the name and certificate were fine and tp-proxy itself wasn't answering.

**WHAT ACTUALLY FIXED IT:** tp-proxy runs the Auth Service, the Proxy Service, the web interface, and recording uploads all on one VM, and 2 GB wasn't enough headroom. Powered it off, raised Base Memory to 4096 MB in VirtualBox, and started it again.

**TIME LOST:**

**SCREENSHOT:**

---

## Entry 9

**STEP:** Week 3, Phase B5, proving tp-db's firewall works

**WHAT I WAS DOING:** Turning on tp-db's firewall over SSH, then testing that direct SSH was blocked.

**WHAT HAPPENED:** My session didn't freeze when the firewall turned on, and running the SSH test gave a "REMOTE HOST IDENTIFICATION HAS CHANGED" warning instead of a timeout.

**WHAT I TRIED:** Looked at where the test was running from.

**WHAT ACTUALLY FIXED IT:** The firewall only blocks new incoming connections, so my already-open session kept working. I had also run the test from inside tp-db, so it was testing tp-db against itself. After `exit` back to Windows, the same test timed out as expected, and the database still answered through Teleport.

**TIME LOST:**

**SCREENSHOT:**

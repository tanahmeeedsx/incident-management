# IncidentOps — MERN Incident Management & On-Call Tracker

**A mini PagerDuty-style incident management & on-call tracking system**, deployed end-to-end on AWS EC2 as a hands-on DevOps project — from provisioning the server, to breaking things, to fixing them properly.

> 🔧 The application codebase (React/Express/MongoDB) started from a DevOps training template. **Everything from "The Deployment Journey" onward is my own independent work** — infrastructure setup, configuration, and real production-style debugging on AWS.

---

## 📑 Table of Contents

- [Why This Project Matters (for DevOps)](#-why-this-project-matters-for-devops)
- [What the App Actually Does — In an Incident](#-what-the-app-actually-does--in-an-incident)
- [Live Demo](#-live-demo)
- [Tech Stack & Tools](#-tech-stack--tools)
- [Architecture](#-architecture)
- [The Deployment Journey — Problems I Hit & How I Fixed Them](#-the-deployment-journey--problems-i-hit--how-i-fixed-them)
- [Command Reference](#-command-reference)
- [Features](#-features)
- [Project Structure](#-project-structure)
- [Running It Yourself](#-running-it-yourself)
- [Roadmap](#-roadmap)
- [Author](#-author)

---

## 💡 Why This Project Matters (for DevOps)

Every real engineering team eventually deals with the same question: **something broke — now what?**

- Who gets paged?
- How fast is it acknowledged?
- Is anyone actually working on it, or is it sitting untouched?
- What did we learn once it's fixed?

This app models that entire lifecycle — **report → assign → investigate → resolve → postmortem** — the same loop tools like PagerDuty, Opsgenie, and Incident.io are built around. Building and deploying it myself meant I had to *actually* think about:

- **On-call ownership** — incidents aren't just tickets, they need a named, accountable engineer
- **Escalation & reminders** — an unresolved critical incident sitting quietly for an hour is a failure mode in itself
- **MTTR (Mean Time To Resolve)** — the core metric any reliability team is judged on
- **Postmortems** — the difference between fixing a fire and preventing the next one

As a DevOps engineer, this isn't just a CRUD app — it's a simplified version of the exact tooling that sits at the center of an SRE/DevOps team's daily workflow.

---

## 🔥 What the App Actually Does — In an Incident

Walking through a real scenario end-to-end:

1. **An incident is reported** — someone (engineer, monitoring alert, on-call lead) creates an incident and sets its **severity** (`low` → `critical`).
2. **An engineer is assigned** — the incident gets an owner from the On-Call Roster. Nothing sits unowned.
3. **Status moves through a Kanban board** — `open` → `investigating` → `resolved`, drag-and-drop, so anyone glancing at the dashboard instantly sees what's actively being worked vs. what's stuck.
4. **Timeline comments track the investigation live** — engineers post updates, and can `@mention` any teammate (searchable by name/email/team) to pull them in — e.g. `@ava@auto-reliability.com can you check the payment API logs?`
5. **The mentioned person gets an in-app notification immediately**, opens it, and jumps straight to the incident — no context lost hunting through Slack threads.
6. **A background reminder job watches for stale high-severity incidents** — if a `high`/`critical` incident has sat too long without movement, it flags it, so nothing critical silently falls through the cracks. This is the same principle behind PagerDuty's escalation policies, just simplified.
7. **Once resolved**, the team fills in **root cause** and **resolution notes**, adds **postmortem action items**, and can **export the whole thing as a PDF** for sharing with stakeholders or leadership.
8. **Dashboard metrics** (including **MTTR**) give a running picture of how the team is actually performing over time — not just how many incidents happened, but how fast they're being closed.
9. **Admins manage the roster** — removing a user automatically unassigns their open incidents and cleans up their notifications, so the system never points to a "ghost" owner.

This is exactly the kind of tool a DevOps/SRE team leans on daily to keep accountability and response time visible — not just to fix problems, but to *prove* how reliably they're being fixed.

---

## 🌐 Live Demo

- **Frontend:** `http://<EC2-PUBLIC-IP>:5173`
- **Backend health check:** `http://<EC2-PUBLIC-IP>:5001/api/health`
- **Demo login:** `admin@auto-reliability.com` / `hello123`

> ⚠️ The EC2 instance isn't kept running 24/7 to avoid unnecessary AWS billing. The walkthrough in this README covers the full experience — happy to spin it back up on request.

---

## 🧰 Tech Stack & Tools

**Application:**

| Layer | Tool |
|---|---|
| Frontend | React.js (Vite) |
| Backend | Node.js, Express.js |
| Database | MongoDB Community Server 8.0 |
| Auth | JWT |
| PDF export | jsPDF |

**Infrastructure & DevOps tools I used to deploy it:**

| Tool | Purpose |
|---|---|
| **AWS EC2** | Virtual server hosting the app (Ubuntu Server 24.04 LTS, t3.micro) |
| **AWS Security Groups** | Firewall-level inbound/outbound traffic control |
| **AWS EC2 Instance Connect** | Browser-based SSH fallback, used to isolate network-level issues |
| **SSH (key-pair auth)** | Secure remote access to the server |
| **Bash scripting** | Wrote `installer.sh` to automate the entire server setup |
| **systemd** | Managing the `mongod` service (start/enable/status) |
| **apt** | Ubuntu package management (updates, upgrades, installs) |
| **Git & GitHub** | Version control and source hosting |
| **npm** | Dependency management & task running for both frontend and backend |
| **nodemon** | Auto-restarting backend during development |
| **Vite dev server** | Frontend dev server, bound to `0.0.0.0` for external access |

---

## 🏗️ Architecture

```
 Developer Machine
      │  SSH (key-pair auth)
      ▼
┌─────────────────────────────────────────────┐
│  AWS EC2 — Ubuntu 24.04 LTS (t3.micro)       │
│                                               │
│   installer.sh                               │
│     ├─ apt update && apt upgrade             │
│     ├─ Node.js 20.x  (NodeSource)            │
│     ├─ MongoDB 8.0   (systemd service)       │
│     └─ npm install   (backend + frontend)    │
│                                               │
│   ┌───────────────┐       ┌────────────────┐ │
│   │ Express API   │──────▶│ MongoDB         │ │
│   │ :5001         │       │ 127.0.0.1:27017 │ │
│   └───────────────┘       └────────────────┘ │
│           ▲                                   │
│   ┌───────────────┐                           │
│   │ Vite Frontend │                           │
│   │ :5173 (0.0.0.0)│                          │
│   └───────────────┘                           │
└─────────────────────────────────────────────┘
      ▲
      │  Inbound: 22 (SSH), 80, 443, 5001, 5173
 AWS Security Group
```

---

## 🐛 The Deployment Journey — Problems I Hit & How I Fixed Them

This is the part of the project that actually taught me the most. Nothing went perfectly the first time — here's exactly what broke and how I diagnosed and fixed each one, the way I'd do it on a real team.

### 1. App "wasn't deploying" — process wasn't persistent
**Symptom:** The app worked while connected over SSH, but the moment I pressed `Ctrl+C` or disconnected, everything went down.
**Root cause:** `npm run dev` was running directly in the foreground of an SSH session — no process manager, so it died with the session (`SIGINT`/`SIGHUP`).
**Fix:** Identified this as a classic "works on my terminal" trap. The proper fix is running the app under a process manager (**pm2**) so it survives disconnects and can auto-restart on crash — this is now the top item on my roadmap below.

### 2. SSH connection timing out
**Symptom:** `ssh: connect to host ... port 22: Connection timed out` — even though it had worked minutes earlier.
**Root cause:** My local ISP assigns a **dynamic public IP**, and the EC2 Security Group's SSH rule only allowed one specific IP. Once my IP changed, I was locked out at the network level.
**Fix:** Diagnosed this systematically rather than guessing:
   - Confirmed the Security Group rule and current IP didn't match
   - Temporarily widened the source and re-tested to isolate SG vs. other causes
   - Used **AWS EC2 Instance Connect (browser-based)** as a control — since it doesn't depend on my local network, if *that* worked but my terminal SSH didn't, it confirmed the issue was client-side/network, not the instance itself

### 3. Pending kernel upgrade after `apt upgrade`
**Symptom:** After a routine `sudo apt upgrade`, the system flagged: *"Pending kernel upgrade! Running kernel version is not the expected kernel version."*
**Root cause:** A new kernel was installed but the running system hadn't rebooted into it yet.
**Fix:** Rather than ignoring the warning, I reasoned through the risk (Nitro-based EC2 instances can occasionally hang on boot after a kernel change) and rebooted deliberately, then verified recovery via:
   - EC2 **Status Checks** (3/3 passed)
   - **EC2 Instance Connect** as a second confirmation the instance was actually reachable, not just "running" per AWS's dashboard

### 4. Nested/duplicated project folders during a repo restructure
**Symptom:** After cloning into a differently-named folder, ended up with a nested `incident-management/incidentops/` structure — not the clean, professional layout I wanted for the final repo.
**Root cause:** `git clone <url> <folder-name>` still creates the repo contents *inside* that folder — an easy, common mistake.
**Fix:** Used `rsync -a incidentops/ ./` to flatten the structure cleanly (safer than a raw `mv *`, which breaks on hidden files and shell-specific glob syntax — zsh in particular chokes on `!` in glob patterns).

### 5. `git push` authentication failing
**Symptom:** `remote: No anonymous write access. fatal: Authentication failed`
**Root cause:** GitHub deprecated password authentication over HTTPS years ago — a plain password is no longer accepted, and I hadn't set up either a Personal Access Token or SSH key on this machine yet.
**Fix:** Set up **SSH key-based authentication** with GitHub (`ssh-keygen`, added the public key to GitHub's SSH settings) and switched the remote from HTTPS to SSH with `git remote set-url`, avoiding the need to manage token expiry going forward.

**The common thread:** almost none of these were application bugs — they were **infrastructure, networking, and tooling** issues. That's the actual day-to-day of DevOps work: the code is often fine, but getting it running reliably in a real environment is its own skill.

---

## ⌨️ Command Reference

Every command I actually used, grouped by context.

### 🖥️ Local Setup

```bash
# Copy environment templates
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env

# Install all dependencies (root, backend, frontend)
npm install
npm run install:all

# Seed the database with sample data
npm run seed

# Run both frontend + backend together
npm run dev

# Run individually
npm run dev --prefix backend
npm run dev --prefix frontend
```

### ☁️ AWS EC2 — Server Provisioning & Access

```bash
# Secure the downloaded key pair
chmod 400 "your-key.pem"

# SSH into the instance
ssh -i "your-key.pem" ubuntu@<EC2-PUBLIC-DNS>
```

### 🔧 Server Setup (on the EC2 instance)

```bash
# Update package lists and upgrade installed packages
sudo apt update
sudo apt upgrade

# Run the automated installer (Node.js, MongoDB, dependencies, seed)
chmod +x installer.sh
./installer.sh

# Verify installed versions / service status
node -v
npm -v
sudo systemctl status mongod
sudo systemctl start mongod
sudo systemctl enable mongod
```

### 📝 Editing environment config on the server

```bash
nano frontend/.env
# VITE_API_URL=http://<EC2-PUBLIC-IP>:5001/api

nano backend/.env
# CLIENT_URL=http://<EC2-PUBLIC-IP>:5173
```

### ▶️ Running the app on the server

```bash
npm run dev
```

### 🔍 Diagnostics used while troubleshooting

```bash
# Raw TCP connectivity check (bypasses SSH client entirely)
nc -vz <EC2-PUBLIC-IP> 22

# Check current public IP (to compare against Security Group rules)
curl ifconfig.me

# Check firewall status inside the instance
sudo ufw status
```

### 🌱 Git / GitHub

```bash
git clone <repo-url>
git remote -v
git remote set-url origin git@github.com:<username>/<repo>.git

git add .
git commit -m "message"
git branch -M main
git push -u origin main

# SSH key setup for GitHub auth
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub   # → add this to GitHub → Settings → SSH and GPG keys
```

---

## 📦 Features

- JWT authentication (login, protected routes)
- Incident creation with severity levels (`low`, `medium`, `high`, `critical`)
- Engineer assignment & status tracking (`open`, `investigating`, `resolved`)
- Drag-and-drop Kanban board for status changes
- Timeline comments with searchable `@mention` support across all users
- In-app notifications when mentioned, linking directly to the incident
- Root cause analysis & resolution notes
- Postmortem action items + one-click PDF export
- Dashboard metrics including **MTTR**
- Admin-only incident deletion & roster management, with automatic unassignment/cleanup
- Background reminder job for stale high-severity incidents

---

## 📂 Project Structure

```
incident-management/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   ├── controllers/
│   │   ├── middleware/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── utils/
│   │   └── server.js
│   └── scripts/seed.js
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── context/
│   │   ├── pages/
│   │   ├── services/
│   │   └── styles.css
│   └── public/
├── installer.sh
├── package.json
└── README.md
```

---

## ▶️ Running It Yourself

### Locally
```bash
git clone https://github.com/tanahmeeedsx/incident-management.git
cd incident-management
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
npm install
npm run install:all
npm run seed
npm run dev
```
Frontend: `http://localhost:5173` · Backend health: `http://localhost:5001/api/health`

### On an AWS EC2 Ubuntu VM
```bash
git clone https://github.com/tanahmeeedsx/incident-management.git
cd incident-management
chmod +x installer.sh
./installer.sh
```
Then update `frontend/.env` (`VITE_API_URL`) and `backend/.env` (`CLIENT_URL`) with your instance's public IP, then:
```bash
npm run dev
```

**Default login:** `admin@auto-reliability.com` / `hello123`

---

## 🔮 Roadmap

- [ ] Move process management from `npm run dev` to **pm2** for persistence across SSH sessions and auto-restart on crash
- [ ] Attach an **Elastic IP** so the public address doesn't change on instance restart
- [ ] Put **Nginx** in front of the app as a reverse proxy + HTTPS (Let's Encrypt), instead of exposing dev ports directly
- [ ] Basic **CloudWatch** monitoring/alarms for the instance
- [ ] **CI/CD pipeline** (GitHub Actions) for automated deploy on push
- [ ] Real escalation policies (e.g. auto-reassign if a critical incident isn't acknowledged within X minutes)

---

## 👤 Author

**Tanjim Ahmed** — DevOps Engineering
- GitHub: [@tanahmeeedsx](https://github.com/tanahmeeedsx)
- LinkedIn: [linkedin.com/in/tanahmedd](https://linkedin.com/in/tanahmedd)

# IncidentOps — MERN Incident Management & On-Call Tracker

A mini PagerDuty-style incident management system (MERN stack), deployed end-to-end on AWS EC2 by me as a hands-on DevOps exercise — from provisioning the server to debugging real production-style issues.

> **Note on origin:** The application codebase (React/Express/MongoDB app) is based on a DevOps training project template. **Everything in the "What I Did" section below — provisioning, deployment, configuration, and troubleshooting — is my own work**, done independently on AWS.

---

## 🚀 Live Demo

- **Frontend:** `http://<EC2-PUBLIC-IP>:5173` *(may not be live 24/7 — see note below)*
- **Backend health check:** `http://<EC2-PUBLIC-IP>:5001/api/health`
- **Demo login:** `admin@auto-reliability.com` / `hello123`

> ⚠️ The EC2 instance is not kept running continuously to avoid unnecessary AWS billing. If the link above isn't live, see the screenshots/description below, or reach out and I can spin it back up.

---

## 🛠️ What I Did (DevOps Work)

This is the part that matters most — the actual deployment engineering:

### 1. AWS Infrastructure
- Provisioned an **EC2 instance** (Ubuntu Server 24.04 LTS, t3.micro) from scratch
- Created and managed a **key pair** for SSH access, applied correct permissions (`chmod 400`)
- Configured a **Security Group** with inbound rules for SSH (22), HTTP (80), HTTPS (443), and custom app ports (5001 backend, 5173 frontend)
- Diagnosed and fixed **SSH connectivity issues** caused by dynamic/CGNAT client IPs — identified the difference between Security Group source restrictions vs. actual network reachability by testing with EC2 Instance Connect (browser-based) as a control

### 2. Server Setup & Automation
- Wrote and ran a Bash **installer script** to automate:
  - System package updates
  - Node.js 20.x installation (via NodeSource)
  - MongoDB Community Server 8.0 installation and service startup
  - Project dependency installation (`npm install` for both backend & frontend)
  - Database seeding
- Verified services post-install with `systemctl status mongod`, `node -v`, `npm -v`

### 3. Application Configuration
- Configured environment variables (`backend/.env`, `frontend/.env`) to point the frontend at the correct public API URL and the backend at the correct CORS client origin
- Ran the full stack concurrently (`nodemon` for backend, Vite dev server bound to `0.0.0.0` for external access)

### 4. Debugging & Incident Response (the real DevOps part)
- **SSH timeout troubleshooting:** isolated whether the failure was Security Group, Network ACL, or client-side network related, using systematic elimination (raw TCP checks, browser-based Instance Connect as a control, source-IP rule review)
- **Kernel upgrade handling:** identified a pending kernel upgrade (`apt upgrade` flagged a kernel mismatch) and safely rebooted without breaking the running services
- **Process persistence:** recognized that running the app directly via `npm run dev` in an SSH session is fragile (dies on `Ctrl+C` / disconnect) — moving toward `pm2` for production-style process management and auto-restart
- Rebuilt the environment cleanly on a **fresh EC2 instance** after infrastructure changes, re-verifying the full install → configure → run pipeline end-to-end

---

## 📦 Application Features (base app)

- JWT authentication (login, protected routes)
- Incident creation with severity levels (`low`, `medium`, `high`, `critical`)
- Engineer assignment & status tracking (`open`, `investigating`, `resolved`)
- Drag-and-drop Kanban board for status changes
- Timeline comments with searchable `@mention` support
- Root cause analysis & resolution notes
- Postmortem action items + PDF export (jsPDF)
- Dashboard metrics including MTTR
- Admin-only incident deletion, with cascading notification/assignment cleanup
- In-app notification system
- Local reminder job for stale high-severity incidents

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React.js (Vite) |
| Backend | Node.js, Express.js |
| Database | MongoDB (Community Server 8.0) |
| Auth | JWT |
| Infra | AWS EC2 (Ubuntu 24.04 LTS), Security Groups |
| Process/Deploy | Bash automation script, npm workspaces, (moving to) pm2 |

---

## 🏗️ Architecture / Deployment Flow

```
Local Dev Machine
      │  (SSH, key-pair auth)
      ▼
AWS EC2 (Ubuntu 24.04, t3.micro)
      │
      ├── installer.sh
      │     ├── apt update/upgrade
      │     ├── Node.js 20.x (NodeSource)
      │     ├── MongoDB 8.0 (systemd service)
      │     └── npm install (backend + frontend)
      │
      ├── backend/  → Express API on :5001 → MongoDB (127.0.0.1:27017)
      └── frontend/ → Vite dev server on :5173 (host 0.0.0.0)

Security Group: 22 (SSH), 80, 443, 5001, 5173
```

---

## 📂 Project Structure

```
incidentops/
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

### Local
```bash
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
git clone <this-repo-url>
cd incidentops
chmod +x installer.sh
./installer.sh
```
Then update `frontend/.env` (`VITE_API_URL`) and `backend/.env` (`CLIENT_URL`) with your instance's public IP, and:
```bash
npm run dev
```

**Default login:** `admin@auto-reliability.com` / `hello123`

---

## 🔮 Next Steps

- [ ] Move process management from `npm run dev` to **pm2** for persistence across SSH sessions
- [ ] Attach an **Elastic IP** so the public address doesn't change on instance restart
- [ ] Put Nginx in front of the app as a reverse proxy + HTTPS (Let's Encrypt), instead of exposing dev ports directly
- [ ] Set up basic CloudWatch monitoring / alarms
- [ ] CI/CD pipeline (GitHub Actions) for automated deploy on push

---

## 👤 Author

**Tanjim Ahmed** — DevOps Engineering
- GitHub: [@tanahmeeedsx](https://github.com/tanahmeeedsx)
- LinkedIn: [linkedin.com/in/tanahmedd](https://linkedin.com/in/tanahmedd)

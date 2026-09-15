# IncidentOps — MERN Incident Management & On-Call Tracker

A mini PagerDuty-style **Incident Management & On-Call Tracker** built with the MERN stack and deployed end-to-end on **AWS EC2** as a hands-on DevOps project.

> The application codebase started from a DevOps training template. The deployment, infrastructure setup, configuration, troubleshooting, and AWS-related work documented below were completed as part of my DevOps learning journey.

---

## Features

* 🔐 JWT authentication
* 🚨 Incident reporting with severity levels: Low, Medium, High, Critical
* 👨‍💻 Engineer assignment from the On-Call Roster
* 📋 Kanban workflow: Open → Investigating → Resolved
* 💬 Timeline comments with searchable `@mentions`
* 🔔 In-app notifications
* ⏰ Background reminders for stale high-severity incidents
* 📝 Root cause and resolution notes
* 📄 Postmortem action items with PDF export
* 📊 Dashboard metrics including MTTR
* 👥 Admin roster management

---

## Tech Stack

### Application

![React](https://img.shields.io/badge/React-20232A?style=for-the-badge\&logo=react\&logoColor=61DAFB)
![Vite](https://img.shields.io/badge/Vite-646CFF?style=for-the-badge\&logo=vite\&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge\&logo=node.js\&logoColor=white)
![Express.js](https://img.shields.io/badge/Express.js-000000?style=for-the-badge\&logo=express\&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge\&logo=mongodb\&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?style=for-the-badge\&logo=jsonwebtokens\&logoColor=white)

### DevOps & Infrastructure

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge\&logo=amazonaws\&logoColor=white)
![EC2](https://img.shields.io/badge/Amazon_EC2-FF9900?style=for-the-badge\&logo=amazonec2\&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge\&logo=ubuntu\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge\&logo=linux\&logoColor=black)
![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge\&logo=git\&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge\&logo=github\&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-121011?style=for-the-badge\&logo=gnubash\&logoColor=white)

* AWS EC2
* Ubuntu Server 24.04 LTS
* AWS Security Groups
* AWS EC2 Instance Connect
* SSH key-pair authentication
* Bash scripting
* systemd
* apt
* npm

---

## Architecture

```text
Developer Machine
       │
       │ SSH
       ▼
┌─────────────────────────────┐
│       AWS EC2               │
│     Ubuntu 24.04            │
│                             │
│  ┌───────────────────────┐  │
│  │ React / Vite :5173    │  │
│  └───────────┬───────────┘  │
│              │              │
│  ┌───────────▼───────────┐  │
│  │ Express API :5001     │  │
│  └───────────┬───────────┘  │
│              │              │
│  ┌───────────▼───────────┐  │
│  │ MongoDB :27017        │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

---

## Deployment on AWS EC2

The application was deployed on an **AWS EC2 t3.micro instance running Ubuntu 24.04 LTS**.

### 1. Connect to EC2

```bash
chmod 400 "your-key.pem"

ssh -i "your-key.pem" ubuntu@<EC2-PUBLIC-DNS>
```

### 2. Install Required Packages

```bash
sudo apt update
sudo apt upgrade

chmod +x installer.sh
./installer.sh
```

The `installer.sh` script handles:

* System package updates
* Required Linux packages
* Node.js 20.x
* MongoDB 8.0
* MongoDB service configuration
* Environment file setup
* Backend dependencies
* Frontend dependencies
* Database seeding

### 3. Verify Services

```bash
node -v
npm -v

sudo systemctl status mongod
```

If MongoDB is not running:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

### 4. Configure Environment Variables

Frontend:

```bash
nano frontend/.env
```

```env
VITE_API_URL=http://<EC2-PUBLIC-IP>:5001/api
```

Backend:

```bash
nano backend/.env
```

```env
CLIENT_URL=http://<EC2-PUBLIC-IP>:5173
```

### 5. Configure AWS Security Group

Required inbound ports:

| Port | Purpose       |
| ---- | ------------- |
| 22   | SSH           |
| 80   | HTTP          |
| 443  | HTTPS         |
| 5001 | Express API   |
| 5173 | Vite Frontend |

### 6. Start the Application

```bash
npm run dev
```

Frontend:

```text
http://<EC2-PUBLIC-IP>:5173
```

Backend health check:

```text
http://<EC2-PUBLIC-IP>:5001/api/health
```

---

## Deployment Challenges & Fixes

### SSH Connection Timeout

The EC2 Security Group was configured to allow SSH from a specific public IP. When the ISP IP changed, SSH connections started timing out.

**Troubleshooting:**

```bash
curl ifconfig.me

nc -vz <EC2-PUBLIC-IP> 22
```

**Fix:**

* Checked the current public IP
* Verified the Security Group SSH rule
* Tested port 22 connectivity
* Used EC2 Instance Connect to isolate the issue

---

### Pending Kernel Upgrade

After running:

```bash
sudo apt upgrade
```

the system reported that a newer kernel was installed but was not yet running.

**Fix:**

```bash
sudo reboot
```

After rebooting, I verified the EC2 status checks and confirmed access through EC2 Instance Connect.

---

### Application Process Stopped

Initially, the application was started using:

```bash
npm run dev
```

Because it was running directly inside the SSH session, the process stopped when the session ended.

**Finding:**

A process manager such as **PM2** is more suitable for persistent application processes. This is kept as a future improvement.

---

### Nested Repository Directory

After cloning the repository, the application was placed inside a nested directory:

```text
incident-management/
└── incidentops/
```

**Fix:**

```bash
rsync -a incidentops/ ./
```

The nested directory was then removed.

---

### GitHub Authentication

HTTPS Git authentication failed because GitHub no longer supports password authentication for Git operations.

**Fix:**

Created an Ed25519 SSH key:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"

cat ~/.ssh/id_ed25519.pub
```

Then changed the repository remote:

```bash
git remote set-url origin git@github.com:<username>/<repo>.git
```

---

## Project Structure

```text
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
│   └── scripts/
│       └── seed.js
│
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── context/
│   │   ├── pages/
│   │   ├── services/
│   │   └── styles.css
│   └── public/
│
├── installer.sh
├── package.json
└── README.md
```

---

## Run Locally

### Prerequisites

* Node.js 20+
* npm
* MongoDB

### Setup

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

Frontend:

```text
http://localhost:5173
```

Backend health check:

```text
http://localhost:5001/api/health
```

---

## Useful Commands

### Backend

```bash
npm run dev --prefix backend
```

### Frontend

```bash
npm run dev --prefix frontend
```

### Database Seed

```bash
npm run seed
```

### Git

```bash
git status
git add .
git commit -m "Update project"
git push
```

### MongoDB

```bash
sudo systemctl status mongod
sudo systemctl start mongod
sudo systemctl enable mongod
```

---

## DevOps Learning

This project provided hands-on experience with:

* AWS EC2 deployment
* Linux server administration
* SSH authentication
* AWS Security Groups
* MongoDB service management
* Environment configuration
* Bash automation
* Git & GitHub
* Network troubleshooting
* Application deployment and debugging

---

## Roadmap

* [ ] PM2 process management
* [ ] Elastic IP
* [ ] Nginx reverse proxy
* [ ] HTTPS with Let's Encrypt
* [ ] CloudWatch monitoring and alarms
* [ ] GitHub Actions CI/CD
* [ ] Advanced escalation policies

---

## Author

**Tanjim Ahmed**

DevOps Engineer | Linux | Docker | CI/CD | AWS | Kubernetes

* GitHub: [tanahmeeedsx](https://github.com/tanahmeeedsx)
* LinkedIn: [tanahmedd](https://linkedin.com/in/tanahmedd)

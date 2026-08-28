# 🏙️ CivicWatch (CityCare24)

A full-stack civic issue reporting platform that allows citizens to report, track, and resolve community problems like road damage, garbage, broken streetlights, and more — with real-time updates and an interactive map view.

---

## 🚀 Live Demo

| Service  | URL |
|----------|-----|
| Frontend | http://13.233.33.151:3000 |
| Backend API | http://13.233.33.151:5000 |

---

## 🧰 Tech Stack

### Frontend
| Technology | Purpose |
|-----------|---------|
| React (Vite) | UI framework |
| React Router | Client-side routing |
| Socket.IO Client | Real-time updates |
| CSS Modules | Component styling |

### Backend
| Technology | Purpose |
|-----------|---------|
| Node.js + Express | REST API server |
| MongoDB Atlas | Cloud database |
| Mongoose | MongoDB ODM |
| JWT | Authentication tokens |
| bcryptjs | Password hashing |
| Socket.IO | Real-time WebSocket events |

### DevOps / Deployment
| Technology | Purpose |
|-----------|---------|
| Docker | Containerization |
| Docker Compose | Multi-container orchestration |
| Terraform | AWS infrastructure provisioning |
| Ansible | Automated server configuration & deployment |
| AWS EC2 | Cloud virtual machine (Ubuntu 24.04, t3.micro) |
| AWS Security Groups | Firewall / port management |

---

## ✨ Features

- 🔐 **User Authentication** — Register / Login with JWT-based auth
- 📋 **Report Issues** — Submit civic issues with title, category, location, image URL
- 🗺️ **Map View** — See all reported issues on an interactive map with lat/lng pins
- 📄 **List View** — Browse all issues in a card layout with status badges
- 👍 **Upvoting** — Community upvote issues to prioritize them
- 📡 **Real-time Updates** — Socket.IO pushes live updates when issues are created/updated
- 🏷️ **Issue Status Tracking** — `Reported` → `In Progress` → `Resolved` → `Re-opened`
- 👤 **Tag Representatives** — Link issues to specific representatives
- ✅ **Resolution Proof** — Representatives can close issues with a message + image

---

## 📁 Project Structure

```
CityCare24/
│
├── docker-compose.yml          # Runs backend + frontend containers together
│
├── backend/                    # Node.js REST API
│   ├── Dockerfile
│   ├── server.js               # Entry point — Express + Socket.IO setup
│   ├── routes/
│   │   ├── authRoutes.js       # /api/auth — login, register
│   │   └── issueRoutes.js      # /api/issues — CRUD + upvote
│   ├── models/
│   │   ├── User.js             # User schema
│   │   └── Issue.js            # Issue schema
│   ├── middleware/
│   │   └── auth.js             # JWT verification middleware
│   └── api/
│       └── index.js            # Vercel serverless entry (alternate)
│
├── frontend/                   # React app (Vite)
│   ├── Dockerfile
│   ├── src/
│   │   ├── App.jsx             # Routes + Auth protection
│   │   ├── pages/
│   │   │   ├── Auth.jsx        # Login / Register page
│   │   │   ├── ListView.jsx    # Issue list page
│   │   │   ├── MapView.jsx     # Map page
│   │   │   └── ReportIssue.jsx # Report new issue page
│   │   ├── components/
│   │   │   ├── Navbar.jsx
│   │   │   ├── IssueCard.jsx
│   │   │   └── ThemeSwitcher.jsx
│   │   └── utils/
│   │       ├── api.js          # Axios API calls
│   │       └── AuthContext.jsx # Global auth state
│   └── vite.config.js
│
├── terraform/                  # AWS infrastructure as code
│   ├── main.tf                 # EC2 instance + Security Group
│   ├── variables.tf
│   └── terraform.tfvars        # key_name = "citycare24-key"
│
└── ansible/                    # Automated deployment
    ├── deploy.yml              # Full deployment playbook
    └── inventory               # EC2 server IP
```

---

## 🏗️ Deployment Architecture

```
Your Machine
    │
    ├── Terraform ──────────► AWS EC2 (Ubuntu 24.04, t3.micro)
    │                              │
    └── Ansible ────────────►      ├── Docker installed
                                   ├── Code cloned from GitHub
                                   ├── .env created (MONGO_URI injected)
                                   └── docker compose up --build
                                              │
                                   ┌──────────┴──────────┐
                                   │                     │
                          citycare-backend      citycare-frontend
                          (Node.js:5000)        (Nginx:3000→80)
                                   │
                                   └──► MongoDB Atlas (Cloud)
```

---

## ⚙️ API Endpoints

### Auth — `/api/auth`
| Method | Route | Description |
|--------|-------|-------------|
| POST | `/register` | Register new user |
| POST | `/login` | Login, returns JWT token |

### Issues — `/api/issues`
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/` | Get all issues |
| POST | `/` | Create new issue (auth required) |
| PUT | `/:id` | Update issue status (auth required) |
| PUT | `/:id/upvote` | Upvote an issue (auth required) |

---

## 🐳 Docker Setup

### backend/Dockerfile
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev        # Install production deps only
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

### frontend/Dockerfile (Multi-stage)
```dockerfile
# Stage 1: Build React app
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL=http://localhost:5000/api
RUN npm run build             # Outputs to /app/dist

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 🌍 Infrastructure (Terraform)

`terraform/main.tf` provisions:
- **EC2 Instance** — `t3.micro`, Ubuntu 24.04, 20GB gp3 disk, `ap-south-1` (Mumbai)
- **Security Group** — Opens ports: `22` (SSH), `80` (HTTP), `3000` (Frontend), `5000` (Backend)
- **Auto-discovers** latest Ubuntu 24.04 AMI — no hardcoded AMI ID

```bash
cd terraform/
terraform init
terraform apply -auto-approve

# Outputs:
# instance_public_ip  = "x.x.x.x"
# instance_public_dns = "ec2-x-x-x-x.ap-south-1.compute.amazonaws.com"
```

---

## 📦 Deployment (Ansible)

`ansible/deploy.yml` runs these steps on the EC2 server automatically:

| Step | Task |
|------|------|
| 1 | Update Ubuntu apt cache |
| 2 | Install git, curl, ca-certificates |
| 3 | Install docker.io + docker-compose-v2 |
| 4 | Start & enable Docker service |
| 5 | Add ubuntu user to docker group |
| 6 | Clone repo from GitHub |
| 7 | Copy docker-compose.yml + Dockerfiles |
| 8 | Create backend `.env` with MONGO_URI |
| 9 | Create 2GB swap file (memory safety for t3.micro) |
| 10 | `docker compose up -d --build` |
| 11 | Print running container status |

```bash
cd ansible/
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory deploy.yml --extra-vars "@secrets.yml" -v
```

---

## 🔧 Local Development Setup

### Prerequisites
- Node.js >= 18
- MongoDB (local or Atlas URI)

### Backend
```bash
cd backend/
npm install
# Create .env file:
echo "MONGO_URI=your_mongodb_uri" > .env
npm start
# Runs on http://localhost:5000
```

### Frontend
```bash
cd frontend/
npm install
npm run dev
# Runs on http://localhost:5173
```

---

## 🔒 Environment Variables

Create `backend/.env`:
```
MONGO_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/civicwatch
```

> In production, this file is created automatically by Ansible using the value from `ansible/secrets.yml` (which is **not** committed to git).

---

## 🛡️ Files Not in This Repository

These files exist locally but are excluded from git for security:

| File | Reason |
|------|--------|
| `*.pem` | SSH private key |
| `ansible/secrets.yml` | Contains MongoDB password |
| `ansible/ansible.cfg` | Contains local file paths |
| `terraform/terraform.tfstate` | Contains AWS account/resource IDs |

---

## 👨‍💻 Author

**AshVortex404** — [github.com/AshVortex404](https://github.com/AshVortex404)

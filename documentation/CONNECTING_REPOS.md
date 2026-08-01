# 🔗 Connecting FaithFlows Frontend & Backend

Guide for running both repositories together for local development.

---

## 📦 Repository Structure

You have **two separate GitHub repositories**:

1. **Frontend**: `faithflows-frontend` (faithflow-studio)
2. **Backend**: `faithflows-backend` (churchcms-be)

---

## 🚀 Quick Setup (Both Repos)

### Option 1: Using Docker (Recommended)

#### Step 1: Setup Directory Structure

```bash
# Create a workspace folder
mkdir faithflows-workspace
cd faithflows-workspace

# Clone both repositories
git clone <frontend-repo-url> frontend
git clone <backend-repo-url> backend
```

Your structure:

```
faithflows-workspace/
├── frontend/          # Frontend repo
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── ...
│
└── backend/           # Backend repo
    ├── docker-compose.yml
    ├── Dockerfile
    └── ...
```

#### Step 2: Start Backend First

```bash
cd backend

# Setup environment
cp env.example .env
# Edit .env if needed (defaults work fine)

# Start backend + database + redis
docker-compose up -d

# Run migrations
docker-compose exec backend npx prisma migrate deploy

# Seed database
docker-compose exec backend npm run seed

# Verify it's running
curl http://localhost:3000/health
```

#### Step 3: Start Frontend

```bash
cd ../frontend

# Setup environment
cp env.example .env

# Make sure VITE_API_URL points to backend
# .env should have:
# VITE_API_URL=http://localhost:3000/api

# Start frontend
docker-compose up -d

# Access the app
open http://localhost:8080
```

#### ✅ Done! Both services running:

- Frontend: http://localhost:8080
- Backend: http://localhost:3000
- Database: localhost:5432
- Redis: localhost:6379

---

### Option 2: Manual Setup (No Docker)

#### Step 1: Clone Both Repos

```bash
mkdir faithflows-workspace
cd faithflows-workspace

git clone <frontend-repo-url> frontend
git clone <backend-repo-url> backend
```

#### Step 2: Setup Backend

```bash
cd backend

# Install dependencies
npm install

# Setup environment
cp env.example .env
# Make sure PostgreSQL is running and update DATABASE_URL in .env

# Generate Prisma client
npx prisma generate

# Run migrations
npx prisma migrate dev

# Seed database
npm run seed

# Start backend
npm run start:dev
```

Backend runs on http://localhost:3000

#### Step 3: Setup Frontend

```bash
cd ../frontend

# Install dependencies
npm install

# Setup environment
cp env.example .env
# Make sure VITE_API_URL=http://localhost:3000/api

# Start frontend
npm run dev

# (Optional) Start JSON server for mock data
json-server --watch db.json --port 3001
```

Frontend runs on http://localhost:8080

---

## 🔌 Connection Configuration

### Frontend → Backend Connection

The frontend connects to backend via environment variable:

**Frontend `.env`:**

```env
VITE_API_URL=http://localhost:3000/api
```

### Backend → Frontend CORS

The backend needs to allow frontend origin:

**Backend `.env`:**

```env
CORS_ORIGIN=http://localhost:8080
```

### Multi-Subdomain Support

For testing multi-tenancy locally:

**Add to hosts file:**

**Windows:** `C:\Windows\System32\drivers\etc\hosts`

```
127.0.0.1 olamchurch.localhost
127.0.0.1 gracechurch.localhost
```

**Mac/Linux:** `/etc/hosts`

```
127.0.0.1 olamchurch.localhost
127.0.0.1 gracechurch.localhost
```

Then access:

- http://olamchurch.localhost:8080
- http://gracechurch.localhost:8080

---

## 🛠️ Development Workflow

### Starting Your Day

```bash
# Terminal 1: Start backend
cd backend
docker-compose -f docker-compose.dev.yml up

# Terminal 2: Start frontend
cd frontend
docker-compose -f docker-compose.dev.yml up
```

### Making Changes

**Frontend Changes:**

- Edit files in `frontend/src`
- Hot reload updates browser instantly

**Backend Changes:**

- Edit files in `backend/src`
- NestJS auto-recompiles and restarts

**Database Changes:**

- Edit `backend/prisma/schema.prisma`
- Run: `docker-compose exec backend npx prisma migrate dev --name change_description`

### Ending Your Day

```bash
# Stop everything
cd backend && docker-compose down
cd ../frontend && docker-compose down
```

---

## 🧪 Testing Both Together

### Test Login Flow

```bash
# 1. Make sure both are running
# Backend: http://localhost:3000
# Frontend: http://localhost:8080

# 2. Open frontend
open http://olamchurch.localhost:8080/login

# 3. Login as admin
Email: nathanielguggisberg@gmail.com
Password: olam@church

# 4. Check API calls in browser DevTools
# Network tab should show calls to http://localhost:3000/api
```

### Test Member Portal

```bash
# Login as member
Email: anasfiat@yahoo.com
Password: OLAMParish@2025

# Should redirect to /change-password
# Change password, then access member dashboard
```

---

## 📊 Port Summary

| Service           | Port | URL                            |
| ----------------- | ---- | ------------------------------ |
| **Frontend**      | 8080 | http://localhost:8080          |
| **Backend**       | 3000 | http://localhost:3000          |
| **Database**      | 5432 | localhost:5432                 |
| **Redis**         | 6379 | localhost:6379                 |
| **Swagger**       | 3000 | http://localhost:3000/api/docs |
| **Prisma Studio** | 5555 | http://localhost:5555          |

---

## 🔄 Keeping Repositories in Sync

### Pull Latest Changes

```bash
# Backend
cd backend
git pull
docker-compose down
docker-compose up --build

# Frontend
cd ../frontend
git pull
docker-compose down
docker-compose up --build
```

### Push Changes

```bash
# Backend
cd backend
git add .
git commit -m "Your commit message"
git push

# Frontend
cd ../frontend
git add .
git commit -m "Your commit message"
git push
```

---

## 🐛 Troubleshooting

### Frontend Can't Connect to Backend

1. **Check backend is running:**

   ```bash
   curl http://localhost:3000/health
   ```

2. **Check CORS settings:**
   Backend `.env` should have:

   ```env
   CORS_ORIGIN=http://localhost:8080
   ```

3. **Check frontend API URL:**
   Frontend `.env` should have:
   ```env
   VITE_API_URL=http://localhost:3000/api
   ```

### Database Connection Issues

```bash
# Check database is running
docker-compose ps

# View database logs
docker-compose logs database

# Test connection
docker-compose exec database psql -U faithflows -d faithflows_db
```

### Port Conflicts

```bash
# Check what's using ports
netstat -ano | findstr :3000  # Windows
lsof -i :3000                 # Mac/Linux

# Change ports in .env files
# Backend: PORT=3001
# Frontend: PORT=8081
```

---

## 🎯 Deployment Strategy

### Development

- Run both locally with Docker Compose
- Each developer clones both repos

### Staging/Production

- Deploy backend to server/cloud (with PostgreSQL)
- Deploy frontend to CDN/static hosting
- Configure CORS and API URLs

### Recommended Hosting

**Backend:**

- AWS EC2 + RDS
- DigitalOcean Droplet + Managed Database
- Heroku + Heroku Postgres
- Railway.app

**Frontend:**

- Vercel
- Netlify
- AWS S3 + CloudFront
- GitHub Pages

---

## 📝 Environment Variables Summary

### Backend `.env`

```env
PORT=3000
DATABASE_URL=postgresql://...
JWT_SECRET=secret-key
CORS_ORIGIN=http://localhost:8080
```

### Frontend `.env`

```env
PORT=8080
VITE_API_URL=http://localhost:3000/api
```

---

## ✅ Quick Checklist

Before starting development:

- [ ] Both repos cloned
- [ ] Docker Desktop running
- [ ] Backend `.env` configured
- [ ] Frontend `.env` configured
- [ ] Backend started (`docker-compose up -d`)
- [ ] Migrations run (`docker-compose exec backend npx prisma migrate deploy`)
- [ ] Database seeded (`docker-compose exec backend npm run seed`)
- [ ] Frontend started (`docker-compose up -d`)
- [ ] Can access http://localhost:8080
- [ ] API calls work (check DevTools Network tab)

---

**Happy Coding! 🚀**








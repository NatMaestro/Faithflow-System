# 🚀 Quick Start: Running FaithFlows with Docker

## ✅ What's Included

Your Docker setup includes:
- ✅ **Nginx** - Web server for frontend (production mode)
- ✅ **Redis** - Caching and session storage
- ✅ **PostgreSQL** - Database
- ✅ **Django Backend** - API server
- ✅ **React Frontend** - User interface

## 📋 Prerequisites

1. **Docker Desktop** installed and running
   - Download: https://www.docker.com/products/docker-desktop
   - Verify: `docker --version` and `docker-compose --version`

2. **Environment file** (optional, uses defaults if not present)
   ```bash
   cp env.example .env
   ```

## 🎯 Step-by-Step Instructions

### Step 1: Navigate to Project Root

```bash
cd "C:\Users\DELL XPS\Documents\Projects\Church Management System\faithflows"
```

### Step 2: Choose Your Mode

#### Option A: Development Mode (Recommended for local development)
```bash
docker-compose -f docker-compose.dev.yml up --build
```

**Features:**
- ✅ Hot reload for frontend and backend
- ✅ See logs in real-time
- ✅ Easy debugging

#### Option B: Production Mode
```bash
docker-compose up --build -d
```

**Features:**
- ✅ Runs in background
- ✅ Optimized builds
- ✅ Nginx serving frontend

### Step 3: Wait for Services to Start

You'll see output like:
```
✅ faithflows-db is up
✅ faithflows-redis is up
✅ faithflows-backend is up
✅ faithflows-frontend is up
```

### Step 4: Initialize Database (First Time Only)

Open a **new terminal** and run:

```bash
# Navigate to project
cd "C:\Users\DELL XPS\Documents\Projects\Church Management System\faithflows"

# Run migrations
docker-compose exec backend python manage.py migrate_schemas --shared
docker-compose exec backend python manage.py migrate_schemas

# Create admin user (optional)
docker-compose exec backend python manage.py createsuperuser
```

### Step 5: Access the Application

- **Frontend**: http://localhost:8080
- **Backend API**: http://localhost:8000
- **API Docs**: http://localhost:8000/api/docs

## 🛑 Stopping Docker

### Development Mode:
```bash
# Press Ctrl+C in the terminal
# Or in a new terminal:
docker-compose -f docker-compose.dev.yml down
```

### Production Mode:
```bash
docker-compose down
```

## 📊 Check Service Status

```bash
# See all running containers
docker-compose ps

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f database
docker-compose logs -f redis
```

## 🔧 Common Commands

### View All Logs
```bash
docker-compose logs -f
```

### Restart a Service
```bash
docker-compose restart backend
docker-compose restart frontend
```

### Access Container Shell
```bash
# Backend shell
docker-compose exec backend bash

# Database shell
docker-compose exec database psql -U faithflows -d faithflows_db

# Redis shell
docker-compose exec redis redis-cli
```

### Rebuild After Code Changes
```bash
# Rebuild specific service
docker-compose up --build backend

# Rebuild all
docker-compose up --build
```

## 🐛 Troubleshooting

### Port Already in Use
```bash
# Check what's using the port
netstat -ano | findstr :8080
netstat -ano | findstr :8000

# Edit .env file to change ports
BACKEND_PORT=8001
FRONTEND_PORT=8081
```

### Services Won't Start
```bash
# Check logs
docker-compose logs backend
docker-compose logs frontend

# Restart everything
docker-compose down
docker-compose up --build
```

### Database Connection Issues
```bash
# Wait for database to be ready
docker-compose ps

# Check database logs
docker-compose logs database

# Restart database
docker-compose restart database
```

### Clear Everything and Start Fresh
```bash
# ⚠️ WARNING: This deletes all data!
docker-compose down -v
docker-compose up --build
```

## 📦 Services Overview

| Service | Port | Description |
|---------|------|-------------|
| **Frontend** | 8080 | React app (Nginx in production) |
| **Backend** | 8000 | Django API server |
| **Database** | 5432 | PostgreSQL |
| **Redis** | 6379 | Cache & sessions |

## ✅ Verification Checklist

After starting, verify:

1. ✅ All containers are running: `docker-compose ps`
2. ✅ Frontend accessible: http://localhost:8080
3. ✅ Backend accessible: http://localhost:8000/api/v1/
4. ✅ Database connected: Check backend logs
5. ✅ Redis connected: `docker-compose exec redis redis-cli ping` (should return `PONG`)

## 🎉 You're Ready!

Your FaithFlows application is now running with:
- ✅ Nginx serving the frontend (production)
- ✅ Redis for caching
- ✅ PostgreSQL database
- ✅ Django backend API
- ✅ All services networked together

**Happy coding! 🚀**



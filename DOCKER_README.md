# 🐳 FaithFlows Docker Setup (Django Backend)

Complete Docker containerization for the FaithFlows Church Management System with Django backend.

## 📋 Prerequisites

- **Docker** (20.10 or higher)
- **Docker Compose** (2.0 or higher)
- **Git** (for cloning the repository)

### Check Your Docker Installation

```bash
docker --version
docker-compose --version
```

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone <repository-url>
cd faithflows
```

### 2. Setup Environment Variables

Copy the example environment file:

```bash
cp env.example .env
```

Edit `.env` and update with your configurations (or use defaults for development).

### 3. Run with Docker Compose

#### **Development Mode** (with hot reload):

```bash
docker-compose -f docker-compose.dev.yml up --build
```

#### **Production Mode**:

```bash
docker-compose up --build -d
```

### 4. Access the Application

- **Frontend**: http://localhost:8080
- **Backend API**: http://localhost:8000
- **API Documentation**: http://localhost:8000/api/docs (if Swagger is enabled)
- **Database**: localhost:5432

---

## 📦 Services Overview

### Architecture

```
┌─────────────────┐
│   Frontend      │ ← http://localhost:8080
│  (React + Vite) │
└────────┬────────┘
         │ API Calls
         ↓
┌─────────────────┐
│   Backend       │ ← http://localhost:8000
│  (Django)       │
└────────┬────────┘
         │
         ↓
┌─────────────────┐     ┌─────────────────┐
│   PostgreSQL    │     │     Redis       │
│   Database      │     │   (Cache)       │
└─────────────────┘     └─────────────────┘
```

### Services

1. **Frontend (faithflow-studio)**
   - Port: 8080
   - Technology: React + TypeScript + Vite
   - Hot reload in dev mode

2. **Backend (faithflow-backend)**
   - Port: 8000
   - Technology: Django + PostgreSQL + Redis
   - Hot reload in dev mode

3. **Database (PostgreSQL)**
   - Port: 5432
   - Version: PostgreSQL 16
   - Persistent volume for data

4. **Redis (Cache)**
   - Port: 6379
   - Used for sessions and caching

---

## 🛠️ Docker Commands

### Start Services

```bash
# Development (with logs)
docker-compose -f docker-compose.dev.yml up

# Development (background)
docker-compose -f docker-compose.dev.yml up -d

# Production (background)
docker-compose up -d
```

### Stop Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (⚠️ deletes database data)
docker-compose down -v
```

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f frontend
docker-compose logs -f backend
docker-compose logs -f database
```

### Rebuild Services

```bash
# Rebuild all
docker-compose up --build

# Rebuild specific service
docker-compose up --build frontend
```

### Execute Commands in Containers

```bash
# Backend shell
docker-compose exec backend bash

# Frontend shell
docker-compose exec frontend sh

# Database shell
docker-compose exec database psql -U faithflows -d faithflows_db
```

---

## 🔧 Development Workflow

### Initial Setup

```bash
# 1. Start services
docker-compose -f docker-compose.dev.yml up -d

# 2. Run database migrations
docker-compose exec backend python manage.py migrate_schemas --shared
docker-compose exec backend python manage.py migrate_schemas

# 3. Create superuser (optional)
docker-compose exec backend python manage.py createsuperuser

# 4. Access the app
open http://localhost:8080
```

### Making Changes

**Frontend Changes:**
- Edit files in `faithflow-studio/src`
- Hot reload automatically updates the browser

**Backend Changes:**
- Edit files in `faithflow-backend/apps`
- Django automatically restarts server

**Database Changes:**
- Create migrations in backend:
  ```bash
  docker-compose exec backend python manage.py makemigrations
  docker-compose exec backend python manage.py migrate_schemas
  ```

---

## 📊 Database Management

### Run Migrations

```bash
# Create migrations
docker-compose exec backend python manage.py makemigrations

# Run migrations (public schema)
docker-compose exec backend python manage.py migrate_schemas --shared

# Run migrations (all tenant schemas)
docker-compose exec backend python manage.py migrate_schemas
```

### Backup Database

```bash
# Create backup
docker-compose exec database pg_dump -U faithflows faithflows_db > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore from backup
docker-compose exec -T database psql -U faithflows faithflows_db < backup_file.sql
```

---

## 🐛 Troubleshooting

### Port Already in Use

```bash
# Check what's using the port
netstat -ano | findstr :8080  # Windows
lsof -i :8080                 # Mac/Linux

# Change ports in .env file
FRONTEND_PORT=3001
BACKEND_PORT=8001
```

### Database Connection Issues

```bash
# Check database health
docker-compose ps

# View database logs
docker-compose logs database

# Restart database
docker-compose restart database
```

### Clear Everything and Start Fresh

```bash
# Stop all services
docker-compose down

# Remove all volumes (⚠️ deletes all data)
docker-compose down -v

# Remove all images
docker-compose down --rmi all

# Rebuild from scratch
docker-compose up --build
```

### Container Won't Start

```bash
# Check container status
docker-compose ps

# View container logs
docker-compose logs <service-name>

# Restart specific service
docker-compose restart <service-name>
```

---

## 🔒 Production Deployment

### Security Checklist

Before deploying to production:

1. ✅ Change all default passwords in `.env`
2. ✅ Set strong `SECRET_KEY`
3. ✅ Configure proper `CORS_ORIGINS`
4. ✅ Set `DEBUG=False`
5. ✅ Enable HTTPS/SSL
6. ✅ Set up database backups
7. ✅ Configure email service
8. ✅ Set up monitoring/logging

### Deploy to Production

```bash
# 1. Set environment to production in .env
DEBUG=False

# 2. Build production images
docker-compose build

# 3. Start services
docker-compose up -d

# 4. Run migrations
docker-compose exec backend python manage.py migrate_schemas --shared
docker-compose exec backend python manage.py migrate_schemas

# 5. Check health
docker-compose ps
```

---

## 🌐 Subdomain Development

For testing multi-tenancy with subdomains locally:

### Modify Hosts File

**Windows** (`C:\Windows\System32\drivers\etc\hosts`):
```
127.0.0.1 olamchurch.localhost
127.0.0.1 gracechurch.localhost
```

**Mac/Linux** (`/etc/hosts`):
```
127.0.0.1 olamchurch.localhost
127.0.0.1 gracechurch.localhost
```

---

## 💾 Data Persistence

### Volumes

Docker volumes ensure data persists across container restarts:

- `postgres_data` - Database data
- `redis_data` - Redis cache
- `static_volume` - Static files
- `media_volume` - Uploaded media files

---

## 🆘 Getting Help

### Common Issues

1. **Port conflicts** - Change ports in `.env`
2. **Database connection failed** - Wait for health check, check logs
3. **Module not found** - Rebuild containers: `docker-compose up --build`
4. **Permission denied** - Run with sudo (Linux/Mac) or check Docker Desktop (Windows)

### Debug Mode

```bash
# Run with verbose logging
docker-compose --verbose up

# Check container details
docker inspect faithflows-backend

# Check network
docker network inspect faithflows-network
```

---

## 🎉 Summary

With Docker, your entire development environment is:

- ✅ **Consistent** - Same environment for all developers
- ✅ **Isolated** - No conflicts with local installations
- ✅ **Portable** - Works on Windows, Mac, Linux
- ✅ **Simple** - One command to start everything
- ✅ **Production-ready** - Same config for dev and prod

**Happy Coding! 🚀**




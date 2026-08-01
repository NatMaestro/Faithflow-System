# 🐳 FaithFlows Docker Setup Guide

Complete Docker containerization for the FaithFlows Church Management System.

## 📋 Prerequisites

- **Docker** (20.10 or higher)
- **Docker Compose** (2.0 or higher)
- **Git** (for cloning the repository)

### Check Your Docker Installation

```bash
docker --version
docker-compose --version
```

---

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
- **Backend API**: http://localhost:3000
- **API Documentation**: http://localhost:3000/api/docs (if Swagger is enabled)
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
│   Backend       │ ← http://localhost:3000
│  (NestJS)       │
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

2. **Backend (churchcms-be)**

   - Port: 3000
   - Technology: NestJS + TypeScript
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
docker-compose exec backend sh

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

# 2. Run database migrations (backend)
docker-compose exec backend npx prisma migrate dev

# 3. Seed database (backend)
docker-compose exec backend npm run seed

# 4. Access the app
open http://localhost:8080
```

### Making Changes

**Frontend Changes:**

- Edit files in `faithflow-studio/src`
- Hot reload automatically updates the browser

**Backend Changes:**

- Edit files in `churchcms-be/src`
- NestJS automatically recompiles and restarts

**Database Changes:**

- Edit `churchcms-be/prisma/schema.prisma`
- Run migrations:
  ```bash
  docker-compose exec backend npx prisma migrate dev --name your_migration_name
  ```

---

## 📊 Database Management

### Run Migrations

```bash
# Create and run migration
docker-compose exec backend npx prisma migrate dev --name migration_name

# Run existing migrations
docker-compose exec backend npx prisma migrate deploy

# Reset database (⚠️ deletes all data)
docker-compose exec backend npx prisma migrate reset
```

### Seed Database

```bash
# Run seeder
docker-compose exec backend npm run seed

# Or use Prisma Studio
docker-compose exec backend npx prisma studio
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
BACKEND_PORT=3002
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
2. ✅ Set strong `JWT_SECRET`
3. ✅ Configure proper `CORS_ORIGIN`
4. ✅ Enable HTTPS/SSL
5. ✅ Set up database backups
6. ✅ Configure email service
7. ✅ Set up monitoring/logging
8. ✅ Use secrets management (Docker Secrets, AWS Secrets Manager, etc.)

### Deploy to Production

```bash
# 1. Set environment to production
export NODE_ENV=production

# 2. Build production images
docker-compose build

# 3. Start services
docker-compose up -d

# 4. Run migrations
docker-compose exec backend npx prisma migrate deploy

# 5. Check health
docker-compose ps
```

---

## 📈 Performance Optimization

### Production Optimizations

1. **Enable Redis Caching**

   - Uncomment Redis configuration in backend
   - Configure cache TTL values

2. **Database Connection Pooling**

   - Configure Prisma connection pool in `schema.prisma`

3. **Nginx Caching**

   - Configure static asset caching in `nginx.conf`

4. **Resource Limits**
   ```yaml
   # Add to docker-compose.yml
   services:
     backend:
       deploy:
         resources:
           limits:
             cpus: "1"
             memory: 1G
           reservations:
             cpus: "0.5"
             memory: 512M
   ```

---

## 🧪 Testing

### Run Tests in Docker

```bash
# Backend tests
docker-compose exec backend npm test

# Frontend tests
docker-compose exec frontend npm test

# E2E tests
docker-compose exec backend npm run test:e2e
```

---

## 📁 Project Structure

```
faithflows/
├── docker-compose.yml          # Production Docker Compose
├── docker-compose.dev.yml      # Development Docker Compose
├── env.example                 # Environment variables template
│
├── faithflow-studio/           # Frontend
│   ├── Dockerfile              # Production Dockerfile
│   ├── Dockerfile.dev          # Development Dockerfile
│   ├── .dockerignore
│   ├── nginx.conf              # Nginx configuration
│   └── src/                    # Source code
│
└── churchcms-be/               # Backend
    ├── Dockerfile              # Production Dockerfile
    ├── Dockerfile.dev          # Development Dockerfile
    ├── .dockerignore
    ├── prisma/                 # Database schema
    └── src/                    # Source code
```

---

## 🎯 Common Use Cases

### Scenario 1: New Developer Onboarding

```bash
# 1. Clone repository
git clone <repo-url>
cd faithflows

# 2. Setup environment
cp env.example .env

# 3. Start everything
docker-compose -f docker-compose.dev.yml up

# Done! App is running
```

### Scenario 2: Pull Latest Changes

```bash
# 1. Pull code
git pull

# 2. Rebuild (if dependencies changed)
docker-compose -f docker-compose.dev.yml up --build

# 3. Run migrations (if schema changed)
docker-compose exec backend npx prisma migrate dev
```

### Scenario 3: Clean Slate

```bash
# Remove everything
docker-compose down -v --rmi all

# Start fresh
docker-compose up --build
```

---

## 🌐 Subdomain Development

For testing multi-tenancy with subdomains locally:

### Option 1: Modify Hosts File

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

### Option 2: Use DNS Service

Configure nginx to handle wildcard subdomains:

```nginx
server_name *.localhost;
```

---

## 💾 Data Persistence

### Volumes

Docker volumes ensure data persists across container restarts:

- `postgres_data` - Database data
- `redis_data` - Redis cache

### Backup Volumes

```bash
# Backup
docker run --rm -v faithflows_postgres_data:/data -v $(pwd):/backup alpine tar czf /backup/db_backup.tar.gz /data

# Restore
docker run --rm -v faithflows_postgres_data:/data -v $(pwd):/backup alpine tar xzf /backup/db_backup.tar.gz -C /
```

---

## 🔍 Monitoring & Logs

### Health Checks

All services have health checks configured:

```bash
# Check service health
docker-compose ps

# Expected output:
# NAME                  STATUS
# faithflows-backend    Up (healthy)
# faithflows-frontend   Up (healthy)
# faithflows-db         Up (healthy)
# faithflows-redis      Up (healthy)
```

### View Logs

```bash
# Real-time logs (all services)
docker-compose logs -f

# Last 100 lines
docker-compose logs --tail=100

# Specific service with timestamps
docker-compose logs -f --timestamps backend
```

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








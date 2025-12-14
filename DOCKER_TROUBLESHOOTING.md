# 🐛 Docker Troubleshooting Guide

## Network Issues During Build

### Problem: `npm install` fails with ECONNRESET

**Error:**
```
npm error code ECONNRESET
npm error network aborted
```

### Solutions:

#### Solution 1: Retry the Build (Recommended)
The Dockerfile now has automatic retry logic. Simply run again:
```bash
docker-compose -f docker-compose.dev.yml up --build
```

#### Solution 2: Build Frontend Separately
If the issue persists, build the frontend container separately:
```bash
# Build just the frontend
docker-compose -f docker-compose.dev.yml build frontend

# Then start all services
docker-compose -f docker-compose.dev.yml up
```

#### Solution 3: Use Local node_modules (Fastest for Development)
If you already have `node_modules` locally, you can skip the npm install in Docker:

1. Install dependencies locally first:
```bash
cd faithflow-studio
npm install
cd ..
```

2. Modify `faithflow-studio/Dockerfile.dev` temporarily:
```dockerfile
# Comment out npm install if node_modules exists
# RUN npm install --legacy-peer-deps || ...
```

3. Or use a volume mount (already configured in docker-compose.dev.yml)

#### Solution 4: Use Different npm Registry
If npm registry is slow, use a mirror:

Edit `faithflow-studio/Dockerfile.dev`:
```dockerfile
RUN npm config set registry https://registry.npmmirror.com
```

#### Solution 5: Increase Docker Build Timeout
If builds timeout:
```bash
# Set environment variable
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1

# Or in PowerShell
$env:DOCKER_BUILDKIT=1
$env:COMPOSE_DOCKER_CLI_BUILD=1
```

## Common Issues

### Issue: "version is obsolete" Warning
✅ **Fixed!** Removed `version: "3.8"` from docker-compose files.

### Issue: Port Already in Use
```bash
# Check what's using the port
netstat -ano | findstr :8080
netstat -ano | findstr :8000

# Change ports in .env
BACKEND_PORT=8001
FRONTEND_PORT=8081
```

### Issue: Database Connection Failed
```bash
# Wait for database to be ready
docker-compose ps

# Check database logs
docker-compose logs database

# Restart database
docker-compose restart database
```

### Issue: Permission Denied
**Windows:**
- Make sure Docker Desktop is running
- Run PowerShell/CMD as Administrator if needed

**Linux/Mac:**
```bash
sudo docker-compose up --build
```

### Issue: Out of Disk Space
```bash
# Clean up Docker
docker system prune -a

# Remove unused volumes
docker volume prune
```

### Issue: Build Cache Problems
```bash
# Build without cache
docker-compose -f docker-compose.dev.yml build --no-cache

# Or rebuild specific service
docker-compose -f docker-compose.dev.yml build --no-cache frontend
```

## Network-Specific Solutions

### If Behind Corporate Proxy
Create `.docker/config.json`:
```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://proxy.company.com:8080",
      "httpsProxy": "http://proxy.company.com:8080",
      "noProxy": "localhost,127.0.0.1"
    }
  }
}
```

### If Using VPN
1. Disconnect VPN temporarily during build
2. Or configure Docker to use VPN network

### Slow Internet Connection
1. Build services one at a time
2. Use `--parallel=false` flag
3. Consider using a faster npm registry mirror

## Quick Fixes

### Restart Everything
```bash
docker-compose down
docker-compose -f docker-compose.dev.yml up --build
```

### Clean Build
```bash
docker-compose down -v
docker system prune -f
docker-compose -f docker-compose.dev.yml up --build
```

### Check Service Health
```bash
docker-compose ps
docker-compose logs backend
docker-compose logs frontend
```

## Still Having Issues?

1. **Check Docker Desktop** is running and updated
2. **Check Internet Connection** - npm needs to download packages
3. **Check Disk Space** - Docker needs space for images
4. **Check Logs** - `docker-compose logs` shows detailed errors
5. **Try Building Locally First** - Install dependencies outside Docker to verify they work

## Alternative: Run Without Docker

If Docker continues to have issues, you can run locally:

```bash
# Backend
cd faithflow-backend
python manage.py runserver

# Frontend (new terminal)
cd faithflow-studio
npm install
npm run dev
```



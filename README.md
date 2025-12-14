# ⛪ FaithFlows - Church Management System

A comprehensive, multi-tenant church management platform built with modern web technologies.

## 🌟 Features

### For Church Members

- 📊 Personal Dashboard
- 👤 Profile Management
- 💵 Online Giving & Payment History
- 📅 Event RSVP & Calendar
- 📢 Announcements & Updates
- 👥 Ministry & Small Group Management
- 🤝 Volunteer Opportunities
- 🙏 Prayer Requests & Altar Calls
- 📝 Service Requests (Baptism, Marriage, etc.)

### For Church Administrators

- 📊 Comprehensive Dashboard & Analytics
- 👥 Member Management
- 📅 Event Planning & Management
- 💰 Financial Tracking & Reporting
- 📢 Announcement System
- 👨‍💼 Role & Permission Management
- 👥 Ministry & Volunteer Management
- 🧾 Tax Receipt Generation
- ⚙️ Church Settings & Customization
- 🎨 Theme Customization

### For Platform Administrators

- 🏢 Multi-Tenant Management
- 🔧 Denomination Configuration
- 📊 System-Wide Analytics
- ⚡ Feature Flags Management

---

## 🚀 Quick Start with Docker (Recommended)

The easiest way to get started is using Docker:

```bash
# 1. Clone the repository
git clone <repository-url>
cd faithflows

# 2. Setup environment
cp env.example .env

# 3. Start everything with one command
docker-compose -f docker-compose.dev.yml up

# 4. Access the application
# Frontend: http://localhost:8080
# Backend: http://localhost:3000
```

**📖 See [DOCKER_SETUP.md](DOCKER_SETUP.md) for complete Docker documentation.**

---

## 💻 Manual Setup (Without Docker)

If you prefer to run without Docker:

### Prerequisites

- Node.js 20+
- PostgreSQL 16+
- npm or yarn

### Backend Setup

```bash
cd churchcms-be

# Install dependencies
npm install

# Setup environment
cp .env.example .env
# Edit .env with your database credentials

# Generate Prisma Client
npx prisma generate

# Run migrations
npx prisma migrate dev

# Seed database
npm run seed

# Start development server
npm run start:dev
```

### Frontend Setup

```bash
cd faithflow-studio

# Install dependencies
npm install

# Start development server
npm run dev
```

---

## 🏗️ Tech Stack

### Frontend

- **Framework**: React 18 + TypeScript
- **Build Tool**: Vite
- **State Management**: Redux Toolkit
- **UI Components**: Shadcn/ui + Tailwind CSS
- **Routing**: React Router v6
- **Forms**: React Hook Form
- **HTTP Client**: Axios
- **Animations**: Framer Motion

### Backend

- **Framework**: NestJS + TypeScript
- **Database**: PostgreSQL + Prisma ORM
- **Authentication**: JWT
- **Validation**: class-validator
- **Documentation**: Swagger/OpenAPI
- **Caching**: Redis (optional)

### DevOps

- **Containerization**: Docker + Docker Compose
- **Reverse Proxy**: Nginx
- **Database**: PostgreSQL 16
- **Cache**: Redis 7

---

## 📁 Project Structure

```
faithflows/
├── docker-compose.yml              # Production Docker setup
├── docker-compose.dev.yml          # Development Docker setup
├── DOCKER_SETUP.md                 # Docker documentation
├── FRONTEND_BACKEND_REQUIREMENTS.md # API documentation
│
├── faithflow-studio/               # Frontend application
│   ├── src/
│   │   ├── features/              # Feature modules
│   │   ├── components/            # Reusable components
│   │   ├── store/                 # Redux store
│   │   ├── api/                   # API services
│   │   └── utils/                 # Utilities
│   ├── Dockerfile                 # Production Dockerfile
│   ├── Dockerfile.dev             # Development Dockerfile
│   └── nginx.conf                 # Nginx configuration
│
└── churchcms-be/                  # Backend application
    ├── src/
    │   ├── modules/               # Feature modules
    │   ├── shared/                # Shared utilities
    │   └── database/              # Database services
    ├── prisma/
    │   └── schema.prisma          # Database schema
    ├── Dockerfile                 # Production Dockerfile
    └── Dockerfile.dev             # Development Dockerfile
```

---

## 🔑 Default Credentials

### Superadmin

- Email: `superadmin@faithflows.com`
- Password: `super123`

### Olam Church Admin

- Email: `nathanielguggisberg@gmail.com`
- Password: `olam@church`

### Olam Church Members (Demo)

All members use temporary password: `OLAMParish@2025`

- `anasfiat@yahoo.com`
- `mild.aikins@gmail.com`
- `ghampsongloria@gmail.com`

**Note:** First-time login requires password change.

---

## 🌐 Multi-Tenancy

FaithFlows supports multiple churches with subdomain-based routing:

- `olamchurch.localhost:8080` - Olam Church
- `gracechurch.localhost:8080` - Grace Church
- etc.

For local development, add entries to your hosts file:

```
127.0.0.1 olamchurch.localhost
127.0.0.1 gracechurch.localhost
```

---

## 📚 Documentation

- [Docker Setup Guide](DOCKER_SETUP.md) - Complete Docker documentation
- [Frontend-Backend Requirements](FRONTEND_BACKEND_REQUIREMENTS.md) - API specifications
- [Backend Documentation](churchcms-be/documentation/) - Backend implementation details
- [Frontend README](faithflow-studio/README.md) - Frontend-specific documentation

---

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📝 License

[Add your license here]

---

## 🆘 Support

For issues and questions:

- Check [DOCKER_SETUP.md](DOCKER_SETUP.md) for Docker troubleshooting
- Review [FRONTEND_BACKEND_REQUIREMENTS.md](FRONTEND_BACKEND_REQUIREMENTS.md) for API documentation
- Open an issue on GitHub

---

## 🎯 Roadmap

- ✅ Multi-tenant architecture
- ✅ Role-based access control
- ✅ Member & admin portals
- ✅ Event management
- ✅ Financial tracking
- ✅ Ministry management
- ✅ Volunteer tracking
- ✅ Docker containerization
- 🚧 Mobile app (React Native)
- 🚧 Email integration
- 🚧 SMS notifications
- 🚧 Payment gateway integration
- 🚧 Advanced analytics

---

**Built with ❤️ for churches worldwide**








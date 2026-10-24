# ObraFlow - Deployment Guide
# ===========================================
# Complete guide for deploying ObraFlow to production
# Supports: Render (Backend) + Vercel (Frontend) OR Render (Both) OR Docker Compose (VPS)

# ===========================================
# ARCHITECTURE OVERVIEW
# ===========================================
# ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
# │   Frontend      │────▶│    Backend      │────▶│   Database      │
# │   (Next.js)     │     │   (Spring Boot) │     │   (PostgreSQL)  │
# │   Port 3000     │     │   Port 8080     │     │   Port 5432     │
# └─────────────────┘     └─────────────────┘     └─────────────────┘
#        │                       │                       │
#        │                  JWT (HS256)                  │
#        │                  REST API                     │
#        └───────────────────────────────────────────────┘

# ===========================================
# OPTION 1: Render (Backend) + Vercel (Frontend) - RECOMMENDED
# ===========================================
# Best for: Zero-ops, auto-scaling, free tier available
# Cost: ~$7-25/month (Backend: $7, DB: $7, Frontend: Free on Vercel)

## Step 1: Prepare Repository
```bash
# Ensure main branch is up to date
git push origin main
```

## Step 2: Create PostgreSQL Database on Render
1. Go to Render Dashboard → New → PostgreSQL
2. Name: `obraflow-db`
3. Database: `obraflow`, User: `obraflow`
4. Region: Oregon (US West) or Frankfurt (EU)
5. Plan: Starter ($7/month)
6. Save the **Internal Database URL** for backend config

## Step 3: Deploy Backend to Render
1. New → Web Service → Connect GitHub repo
2. Name: `obraflow-backend`
3. Runtime: **Docker** (uses `backend/Dockerfile`)
3. Build Command: `./mvnw -B -DskipTests package`
4. Start Command: `java -jar target/obraflow-backend-0.0.1-SNAPSHOT.jar`
5. Environment Variables:
   ```
   SPRING_PROFILES_ACTIVE=prod
   SERVER_PORT=8080
   DB_HOST=<from database internal hostname>
   DB_PORT=5432
   DB_NAME=obraflow
   DB_USER=obraflow
   DB_PASS=<from database password>
   OBRAFLOW_JWT_SECRET=<generate: openssl rand -base64 32>
   SPRING_PROFILES_ACTIVE=prod
   ```
6. Health Check Path: `/actuator/health`
7. Auto-Deploy: Yes

## Step 4: Deploy Frontend to Vercel
1. Install Vercel CLI: `npm i -g vercel`
2. Login: `vercel login`
3. In frontend folder: `vercel`
4. Link to GitHub repo, select framework: Next.js
5. Environment Variables (Project Settings → Environment Variables):
   ```
   BACKEND_URL=https://obraflow-backend.onrender.com
   NEXTAUTH_URL=https://your-app.vercel.app
   NEXTAUTH_SECRET=<generate: openssl rand -base64 32>
   NEXT_PUBLIC_APP_URL=https://your-app.vercel.app
   ```
6. Deploy: `vercel --prod`

## Step 5: Update Backend CORS (if needed)
Add to `application.yml` or `application-prod.yml`:
```yaml
spring:
  web:
    cors:
      allowed-origins: "https://your-app.vercel.app"
```

# ===========================================
# OPTION 2: Render (Both Backend + Frontend)
# ===========================================
# Best for: Single platform, simpler DNS management
# Cost: ~$14-25/month (Backend $7 + Frontend $7 + DB $7)

## Render Blueprint (render.yaml)
Create `render.yaml` in repo root:

```yaml
services:
  - type: pserv
    name: obraflow-db
    env: postgres
    plan: starter
    databaseName: obraflow
    user: obraflow

  - type: web
    name: obraflow-backend
    env: docker
    dockerfilePath: ./backend/Dockerfile
    plan: starter
    healthCheckPath: /actuator/health
    envVars:
      - key: SPRING_PROFILES_ACTIVE
        value: prod
      - fromDatabase:
          name: obraflow-db
          property: host
        key: DB_HOST
      - fromDatabase:
          name: obraflow-db
          property: port
        key: DB_PORT
      - fromDatabase:
          name: obraflow-db
          property: databaseName
        key: DB_NAME
      - fromDatabase:
          name: obraflow-db
          property: user
        key: DB_USER
      - fromDatabase:
          name: obraflow-db
          property: password
        key: DB_PASS
      - key: OBRAFLOW_JWT_SECRET
        generateValue: true
      - key: SPRING_PROFILES_ACTIVE
        value: prod

  - type: web
    name: obraflow-frontend
    env: node
    plan: starter
    buildCommand: npm run build
    startCommand: npm start
    envVars:
      - key: BACKEND_URL
        fromService:
          name: obraflow-backend
          type: web
          property: host
      - key: NEXTAUTH_URL
        value: https://obraflow-frontend.onrender.com
      - key: NEXTAUTH_SECRET
        generateValue: true
      - key: NEXT_PUBLIC_APP_URL
        value: https://obraflow-frontend.onrender.com
```

Deploy: Push `render.yaml` to repo → Render Dashboard → New → Blueprint → Select repo

# ===========================================
# OPTION 3: Docker Compose (VPS/EC2/DigitalOcean)
# ===========================================
# Best for: Full control, existing infrastructure, cost optimization
# Cost: $5-20/month (depending on VPS size)

## Prerequisites
- Ubuntu 22.04+ VPS (2GB+ RAM)
- Docker & Docker Compose installed
- Domain with DNS pointing to VPS IP
- SSL certificates (Let's Encrypt via nginx/Caddy)

## Deployment Steps
```bash
# 1. Clone repository
git clone https://github.com/your-org/obraflow-case.git
cd obraflow-case

# 2. Create production environment file
cp .env.production.example .env.production
# Edit .env.production with your values

# 3. Build and start
docker-compose -f docker-compose.prod.yml up -d --build

# 4. Verify
docker-compose -f docker-compose.prod.yml ps
curl -f http://localhost:3000
curl -f http://localhost:8080/actuator/health
```

## Reverse Proxy (nginx/Caddy)
```nginx
# /etc/nginx/sites-available/obraflow
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Backend API
    location /api/ {
        proxy_pass http://localhost:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/obraflow /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

# ===========================================
# PRODUCTION CHECKLIST
# ===========================================

## Security
- [ ] Strong JWT secret (32+ chars, base64 encoded)
- [ ] Strong NEXTAUTH_SECRET (32+ chars)
- [ ] Strong database password
- [ ] HTTPS enforced (HSTS headers configured)
- [ ] CORS restricted to frontend domain only
- [ ] Rate limiting on auth endpoints
- [ ] Security headers (CSP, HSTS, X-Frame-Options)

## Database
- [ ] Automated backups enabled (Render: daily, VPS: pg_dump cron)
- [ ] Connection pooling configured (HikariCP: 10 connections)
- [ ] SSL/TLS for database connections
- [ ] Read replica for read-heavy workloads (optional)

## Monitoring
- [ ] Health checks: `/actuator/health` (backend), `/` (frontend)
- [ ] Logging: Structured JSON logs
- [ ] Metrics: Prometheus + Grafana (optional)
- [ ] Alerting: Uptime monitoring (UptimeRobot, Pingdom)

## CI/CD
- [ ] GitHub Actions: Lint → Test → Build → Deploy
- [ ] Automated security scanning (Dependabot, Trivy)
- [ ] Staging environment for PR previews
- [ ] Rollback strategy (Render: manual, Vercel: instant rollback)

## Performance
- [ ] CDN for static assets (Vercel Edge Network / Cloudflare)
- [ ] Compression enabled (gzip/brotli)
- [ ] Caching headers for static assets
- [ ] Database indexes on frequently queried columns

# ===========================================
# SWAGGER/OPENAPI DOCUMENTATION
# ===========================================

## Backend (Spring Boot + SpringDoc)
- **Development**: http://localhost:8080/swagger-ui.html
- **Production**: https://your-backend-domain.com/swagger-ui.html
- **OpenAPI JSON**: https://your-backend-domain.com/v3/api-docs

### Configuration (already in place)
- `OpenApiConfig.java` - JWT Bearer auth, API info
- `application.yml` - `springdoc.swagger-ui.path=/swagger-ui.html`
- `application-prod.yml` - Enable in production

### Customization (optional)
```java
// OpenApiConfig.java - Add custom tags, examples, etc.
@Bean
public OpenAPI customOpenAPI() {
    return new OpenAPI()
        .info(new Info()
            .title("ObraFlow API")
            .version("1.0.0")
            .description("API documentation..."))
        .servers(List.of(
            new Server().url("https://api.yourdomain.com").description("Production"),
            new Server().url("https://staging-api.yourdomain.com").description("Staging")
        ))
        .components(new Components()
            .addSecuritySchemes("bearerAuth", new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT")));
}
```

## Frontend API Documentation
- Frontend doesn't expose OpenAPI (it's a consumer)
- Use backend Swagger for API reference
- Frontend components documented in Storybook (optional)

# ===========================================
# ENVIRONMENT VARIABLES REFERENCE
# ===========================================

## Backend (Required)
| Variable | Description | Example |
|----------|-------------|---------|
| `DB_HOST` | PostgreSQL host | `db.internal` |
| `DB_PORT` | PostgreSQL port | `5432` |
| `DB_NAME` | Database name | `obraflow` |
| `DB_USER` | Database user | `obraflow` |
| `DB_PASS` | Database password | `********` |
| `OBRAFLOW_JWT_SECRET` | JWT signing key (base64, 32+ chars) | `base64...` |
| `SPRING_PROFILES_ACTIVE` | Active profile | `prod` |
| `SERVER_PORT` | HTTP port | `8080` |

## Frontend (Required)
| Variable | Description | Example |
|----------|-------------|---------|
| `BACKEND_URL` | Backend API base URL | `https://api.yourdomain.com` |
| `NEXTAUTH_URL` | Frontend canonical URL | `https://app.yourdomain.com` |
| `NEXTAUTH_SECRET` | NextAuth secret (32+ chars) | `base64...` |
| `NEXT_PUBLIC_APP_URL` | Public frontend URL | `https://app.yourdomain.com` |

# ===========================================
# TROUBLESHOOTING
# ===========================================

## Backend won't start
```bash
# Check logs
docker logs obraflow-backend-prod

# Common issues:
# - DB connection: check DB_HOST, credentials, security groups
# - JWT secret: must be 32+ chars, base64 encoded
# - Port conflict: check port 8080 not in use
```

## Frontend build fails
```bash
# Check build locally
npm run build

# Common issues:
# - TypeScript errors: run `npm run typecheck`
# - ESLint errors: run `npm run lint`
# - Missing env vars: check .env.production
```

## Database connection fails
```bash
# Test connection
docker exec -it obraflow-db-prod pg_isready -U obraflow -d obraflow

# Check network
docker network ls
docker network inspect obraflow_default
```

# ===========================================
# ROLLBACK PROCEDURES
# ===========================================

## Render (Backend)
- Dashboard → Service → Manual Deploy → Select previous commit
- Or: `render deploy --service=obraflow-backend --commit=<sha>`

## Vercel (Frontend)
- Dashboard → Deployments → Click "..." on previous deployment → Promote to Production
- Or: `vercel rollback <deployment-url>`

## Docker Compose (VPS)
```bash
# Quick rollback
docker-compose -f docker-compose.prod.yml down
git checkout <previous-tag>
docker-compose -f docker-compose.prod.yml up -d --build

# Or use specific image tag
docker-compose -f docker-compose.prod.yml up -d --force-recreate
```

# ===========================================
# SUPPORT & CONTACTS
# ===========================================
- **Documentation**: This guide + README.md + inline code comments
- **API Docs**: `/swagger-ui.html` (backend)
- **Monitoring**: Render Dashboard / Vercel Dashboard
- **Logs**: `docker logs <container>` or Render/Vercel logs
- **Issues**: GitHub Issues for bugs/features
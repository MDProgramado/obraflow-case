# Render Deployment Configuration
# ===========================================
# This file documents the Render deployment setup
# For actual deployment, use the Render Dashboard or render.yaml

# ===========================================
# Backend Service (Web Service)
# ===========================================
# Name: obraflow-backend
# Region: Oregon (US West) or Frankfurt (EU Central)
# Branch: main
# Build Command: ./mvnw -B -DskipTests package
# Start Command: java -jar target/obraflow-backend-0.0.1-SNAPSHOT.jar
# Runtime: Docker (uses backend/Dockerfile)
# 
# Environment Variables:
# - DB_HOST (from database internal hostname)
# - DB_PORT=5432
# - DB_NAME=obraflow
# - DB_USER=obraflow
# - DB_PASS (from database password)
# - OBRAFLOW_JWT_SECRET (generate secure 32+ char secret)
# - SPRING_PROFILES_ACTIVE=prod
# - SERVER_PORT=8080
#
# Health Check Path: /actuator/health
# Auto-Deploy: Yes

# ===========================================
# Database (PostgreSQL)
# ===========================================
# Name: obraflow-db
# Database Name: obraflow
# User: obraflow
# Region: Same as backend
# Plan: Starter or higher
# 
# Connection Info (auto-injected):
# - POSTGRES_HOST
# - POSTGRES_PORT
# - POSTGRES_DATABASE
# - POSTGRES_USER
# - POSTGRES_PASSWORD

# ===========================================
# Frontend Service (Static Site or Web Service)
# ===========================================
# Option A: Static Site (Recommended for Next.js with output: 'export')
# Name: obraflow-frontend
# Build Command: npm run build
# Publish Directory: out
# 
# Option B: Web Service (for Next.js with output: 'standalone')
# Name: obraflow-frontend
# Build Command: npm run build
# Start Command: npm start
# Runtime: Node.js
# 
# Environment Variables:
# - BACKEND_URL=https://obraflow-backend.onrender.com
# - NEXTAUTH_URL=https://obraflow-frontend.onrender.com
# - NEXTAUTH_SECRET (generate secure secret)
# - NEXT_PUBLIC_APP_URL=https://obraflow-frontend.onrender.com

# ===========================================
# Render Blueprint (render.yaml) - Optional
# ===========================================
# Save as render.yaml in repo root for IaC deployment

# services:
#   - type: web
#     name: obraflow-backend
#     env: docker
#     dockerfilePath: ./backend/Dockerfile
#     plan: starter
#     healthCheckPath: /actuator/health
#     envVars:
#       - key: SPRING_PROFILES_ACTIVE
#         value: prod
#       - key: SERVER_PORT
#         value: "8080"
#       - fromDatabase:
#           name: obraflow-db
#           property: host
#         key: DB_HOST
#       - fromDatabase:
#           name: obraflow-db
#           property: port
#         key: DB_PORT
#       - fromDatabase:
#           name: obraflow-db
#           property: databaseName
#         key: DB_NAME
#       - fromDatabase:
#           name: obraflow-db
#           property: user
#         key: DB_USER
#       - fromDatabase:
#           name: obraflow-db
#           property: password
#         key: DB_PASS
#       - key: OBRAFLOW_JWT_SECRET
#         generateValue: true
#       - key: SPRING_PROFILES_ACTIVE
#         value: prod
#       - key: SERVER_PORT
#         value: "8080"
#
#   - type: pserv
#     name: obraflow-db
#     env: postgres
#     plan: starter
#     databaseName: obraflow
#     user: obraflow
#
#   - type: web
#     name: obraflow-frontend
#     env: node
#     plan: starter
#     buildCommand: npm run build
#     startCommand: npm start
#     envVars:
#       - key: BACKEND_URL
#         fromService:
#           name: obraflow-backend
#           type: web
#           property: host
#       - key: NEXTAUTH_URL
#         value: https://obraflow-frontend.onrender.com
#       - key: NEXTAUTH_SECRET
#         generateValue: true
#       - key: NEXT_PUBLIC_APP_URL
#         value: https://obraflow-frontend.onrender.com
# Next.js Docker Deployment Guide

[← Back to Docker Setup](docker-setup.md) | [Next: CI/CD Setup →](cicd-docker-github.md)

---

## Next.js Docker Architecture

```mermaid
graph TB
    subgraph "Production Environment"
        subgraph "Docker Container"
            N[Next.js App] --> |serves| S[Static Assets]
            N --> |handles| A[API Routes]
            N --> |renders| P[Pages]
            
            subgraph "Node.js Runtime"
                N --> |uses| R[Runtime]
                R --> |executes| C[Server Components]
                R --> |processes| D[Data Fetching]
            end
        end
        
        L[Load Balancer] --> |routes| N
        N --> |connects| DB[(Database)]
        N --> |caches| Redis[(Redis)]
    end
    
    style N fill:#90EE90,stroke:#333
    style L fill:#87CEEB,stroke:#333
    style DB fill:#FFB6C1,stroke:#333
    style Redis fill:#DDA0DD,stroke:#333
```

## Build Process

```mermaid
flowchart TD
    A[Source Code] --> B[Development Image]
    B --> C{Build Stage}
    C --> |Dependencies| D[Install npm packages]
    C --> |Build| E[Next.js Build]
    C --> |Assets| F[Static Files]
    
    D --> G[Production Image]
    E --> G
    F --> G
    
    G --> |Optimize| H[Final Image]
    H --> |Deploy| I[Production]
    
    style A fill:#f9f,stroke:#333
    style G fill:#90EE90,stroke:#333
    style I fill:#87CEEB,stroke:#333
```

## Table of Contents
1. [Project Structure](#project-structure)
2. [Docker Configuration](#docker-configuration)
3. [Development Setup](#development-setup)
4. [Production Build](#production-build)
5. [Deployment](#deployment)
6. [CI/CD Integration](#cicd-integration)

## Project Structure

### Expected Next.js Project Structure
```plaintext
your-nextjs-app/
├── src/
│   ├── app/
│   ├── components/
│   └── ...
├── public/
├── package.json
├── next.config.js
├── .dockerignore
├── Dockerfile
├── Dockerfile.dev
└── docker-compose.yml
```

### Essential Configuration Files

1. **.dockerignore**
```plaintext
node_modules
.next
.git
.env*.local
```

2. **Dockerfile.dev (Development)**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source
COPY . .

# Expose port
EXPOSE 3000

# Start development server
CMD ["npm", "run", "dev"]
```

3. **Dockerfile (Production)**
```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine AS runner

WORKDIR /app

# Copy necessary files from builder
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/node_modules ./node_modules

# Set environment to production
ENV NODE_ENV=production

# Expose port
EXPOSE 3000

# Start production server
CMD ["npm", "start"]
```

4. **docker-compose.yml**
```yaml
version: '3.8'

services:
  # Development service
  nextjs-dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    
  # Production service
  nextjs-prod:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
```

## Development Setup

### Starting Development Environment
```bash
# Build and start development container
docker-compose up nextjs-dev

# View logs
docker-compose logs -f nextjs-dev

# Stop development environment
docker-compose down
```

### Development Best Practices
1. **Hot Reloading**
```yaml
# Add to docker-compose.yml under nextjs-dev service
environment:
  - WATCHPACK_POLLING=true  # Enable hot reloading in Docker
```

2. **Environment Variables**
```yaml
# Add to docker-compose.yml
services:
  nextjs-dev:
    env_file:
      - .env.local
```

## Production Build

### Building Production Image
```bash
# Build production image
docker build -t your-nextjs-app:latest .

# Run production container
docker run -p 3000:3000 your-nextjs-app:latest
```

### Multi-Stage Build Optimization
```dockerfile
# Add to Dockerfile
# Cache dependencies
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Rebuild source code only
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production image
FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
CMD ["npm", "start"]
```

## Deployment

### Docker Registry Push
```bash
# Tag image
docker tag your-nextjs-app:latest your-registry/your-nextjs-app:latest

# Push to registry
docker push your-registry/your-nextjs-app:latest
```

### Server Deployment
```bash
# Pull latest image
docker pull your-registry/your-nextjs-app:latest

# Run with environment variables
docker run -d \
  --name nextjs-production \
  -p 3000:3000 \
  -e DATABASE_URL=your_db_url \
  -e API_KEY=your_api_key \
  your-registry/your-nextjs-app:latest
```

### Using Docker Compose in Production
```yaml
# production.docker-compose.yml
version: '3.8'

services:
  nextjs:
    image: your-registry/your-nextjs-app:latest
    restart: always
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - API_KEY=${API_KEY}
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## CI/CD Integration

### GitHub Actions Example
```yaml
# .github/workflows/docker-deploy.yml
name: Docker Deploy

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: your-registry/your-nextjs-app:latest
```

### Automated Deployment Script
```bash
#!/bin/bash
# deploy.sh

# Pull latest image
docker pull your-registry/your-nextjs-app:latest

# Stop existing container
docker stop nextjs-production || true
docker rm nextjs-production || true

# Start new container
docker run -d \
  --name nextjs-production \
  -p 3000:3000 \
  --env-file .env.production \
  your-registry/your-nextjs-app:latest
```

## Performance Optimization

### Docker Image Optimization
1. **Use .dockerignore**
```plaintext
# Add to .dockerignore
.git
.next
node_modules
*.log
.env*
```

2. **Optimize Node Modules**
```dockerfile
# In Dockerfile
RUN npm ci --only=production
```

3. **Cache Optimization**
```dockerfile
# Copy package files first
COPY package*.json ./
RUN npm install
# Then copy source
COPY . .
```

### Production Considerations
1. **Health Checks**
```dockerfile
# Add to Dockerfile
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/api/health || exit 1
```

2. **Logging Configuration**
```yaml
# In docker-compose.yml
services:
  nextjs:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Build Optimization Process

```mermaid
graph TB
    subgraph "Build Optimization"
        SRC[Source Code] -->|analyze| AN[Code Analysis]
        
        AN -->|optimize| BP[Bundle Processing]
        AN -->|optimize| IM[Image Processing]
        AN -->|optimize| FO[Font Optimization]
        
        BP -->|minify| MIN[Minification]
        IM -->|compress| COMP[Compression]
        FO -->|subset| FONT[Font Subsetting]
        
        MIN --> TREE[Tree Shaking]
        COMP --> CACHE[Cache Strategy]
        FONT --> LOAD[Load Strategy]
        
        TREE --> BUILD[Production Build]
        CACHE --> BUILD
        LOAD --> BUILD
    end
    
    style SRC fill:#f96,stroke:#333
    style BUILD fill:#9cf,stroke:#333
    style MIN fill:#9f9,stroke:#333
    style CACHE fill:#ff9,stroke:#333
```

## Deployment Architecture

```mermaid
graph TB
    subgraph "Client Side"
        CSR[Client Side Rendering]
        SWR[SWR/React Query]
        CACHE[Browser Cache]
        
        CSR -->|fetch| SWR
        SWR -->|store| CACHE
    end
    
    subgraph "Edge Network"
        CDN[CDN]
        EDGE[Edge Functions]
        
        CDN -->|cache| STATIC[Static Assets]
        EDGE -->|process| API[API Routes]
    end
    
    subgraph "Server Side"
        SSR[Server Side Rendering]
        ISR[Incremental Static Regeneration]
        SSG[Static Site Generation]
        
        SSR -->|render| PAGE[Pages]
        ISR -->|cache| PAGE
        SSG -->|build| PAGE
    end
    
    subgraph "Data Layer"
        DB[(Database)]
        REDIS[(Redis Cache)]
        CMS[Headless CMS]
        
        DB -->|data| SSR
        REDIS -->|cache| SSR
        CMS -->|content| SSG
    end
    
    style CDN fill:#f96,stroke:#333
    style EDGE fill:#9cf,stroke:#333
    style SSR fill:#9f9,stroke:#333
    style DB fill:#ff9,stroke:#333
```

## Container Scaling

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant C1 as Container 1
    participant C2 as Container 2
    participant C3 as Container 3
    participant DB as Database
    
    LB->>C1: Route Request
    C1->>DB: Query Data
    DB->>C1: Return Data
    C1->>LB: Response
    
    Note over LB,C3: High Load Detected
    
    LB->>C2: Scale Out
    LB->>C3: Scale Out
    
    par Parallel Processing
        LB->>C1: Request 1
        LB->>C2: Request 2
        LB->>C3: Request 3
    end
    
    Note over LB,C3: Load Normalized
    
    LB->>C3: Scale In
    Note over C3: Container Removed
```

## Next Steps
- Set up monitoring and logging
- Configure CDN and caching
- Implement container orchestration
- Set up backup strategy

---
*This guide will be updated with more detailed sections as we progress.* 
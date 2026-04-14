# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Klip AI PRO** is a multi-service video clipping SaaS platform that transforms long-form YouTube videos into viral short clips using AI. The system consists of four main applications:

- **klipklap** - Next.js 16 landing page (public marketing site)
- **klipklap-dashboard** - Next.js 16 user dashboard for managing projects
- **klipklap-services** - NestJS backend API with Supabase auth
- **klipklap-worker** - Express.js microservice for yt-dlp video downloads

## Architecture Overview

```
┌─────────────────────┐     ┌──────────────────────────┐
│   klipklap          │     │   klipklap-dashboard     │
│   (Landing Page)    │     │   (User Dashboard)       │
│   Port: 3000        │     │   Port: 3001             │
└──────────┬──────────┘     └─────────────┬────────────┘
           │                              │
           └──────────┬───────────────────┘
                      │
                      ▼
        ┌─────────────────────────┐
        │   klipklap-services      │
        │   (NestJS API)           │
        │   Port: 4000             │
        └──────────┬───────────────┘
                   │
        ┌──────────┴───────────┐
        ▼                      ▼
┌───────────────┐     ┌────────────────┐
│ Supabase      │     │ yt-dlp-worker  │
│ (Auth + DB)   │     │ Port: 3002     │
└───────────────┘     └────────────────┘
```

### Data Flow
1. User submits YouTube URL via dashboard
2. Dashboard calls klipklap-services API
3. Services calls yt-dlp-worker to download video
4. Services uses OpenAI for transcription/analysis
5. FFmpeg processes clips via Redis Bull queue
6. Final clips uploaded to Cloudflare R2 (S3-compatible)
7. Dashboard polls for job status and displays results

## Development Commands

### Root Project
```bash
# Install all dependencies (run from each subdirectory)
cd klipklap && npm install
cd klipklap-dashboard && npm install
cd klipklap-services && npm install
cd klipklap-worker && npm install
```

### klipklap (Landing Page)
```bash
cd klipklap
npm run dev          # Start dev server on :3000
npm run build        # Production build
npm run start        # Start production server
npm run lint         # ESLint
```

### klipklap-dashboard (User Dashboard)
```bash
cd klipklap-dashboard
npm run dev          # Start dev server on :3001
npm run build        # Production build
npm run start        # Start production server
npm run lint         # ESLint
```

### klipklap-services (Backend API)
```bash
cd klipklap-services
npm run start:dev    # Development with hot reload
npm run build        # Compile TypeScript
npm run start:prod   # Production mode
npm run lint         # ESLint with fix
npm run test         # Unit tests
npm run test:cov     # Coverage
npm run test:e2e     # End-to-end tests

# TypeORM migrations
npm run typeorm -- migration:generate -n migration_name
npm run typeorm -- migration:run
npm run typeorm -- migration:revert
```

### klipklap-worker (yt-dlp Service)
```bash
cd klipklap-worker
npm run dev          # Nodemon dev server on :3002
npm start            # Production server
```

### Docker Deployment
```bash
# Services with docker-compose
cd klipklap-services
docker-compose up -d

# Worker separately
cd klipklap-worker
docker build -t yt-dlp-worker .
docker run -p 3002:3000 --env-file .env yt-dlp-worker
```

## Technology Stack

### Frontend (klipklap & klipklap-dashboard)
- **Framework**: Next.js 16.0.10 with App Router
- **Runtime**: React 19.2.x with TypeScript 5
- **Styling**: Tailwind CSS v4.1.x
- **UI**: Radix UI primitives (shadcn/ui "new-york" style)
- **i18n**: next-intl (Indonesian locale as default)
- **Auth**: Supabase Auth with PKCE flow
- **Cloudflare**: @opennextjs/cloudflare for deployment

### Backend (klipklap-services)
- **Framework**: NestJS 11 with TypeScript
- **Database**: PostgreSQL via Supabase with TypeORM
- **Auth**: Supabase JWT validation
- **Queue**: Redis with Bull for FFmpeg jobs
- **AI**: OpenAI (GPT-4o for analysis, Whisper for transcription)
- **Storage**: Cloudflare R2 (S3-compatible)
- **Payments**: Xendit integration
- **API Docs**: Swagger at `/api`

### Worker (klipklap-worker)
- **Framework**: Express.js
- **Video**: yt-dlp with Deno runtime, FFmpeg
- **Auth**: API key via `X-API-KEY` header

## Key Architecture Patterns

### Supabase Authentication Flow
1. Frontend initiates Supabase Auth (magic link or OAuth)
2. User receives callback at `/auth/callback`
3. Frontend stores JWT token
4. Backend validates JWT using `SUPABASE_SECRET_KEY`
5. Protected routes use `@UseGuards(SupabaseAuthGuard)`

**Important**: Never use `SUPABASE_ANON_KEY` on backend. Use `SUPABASE_SECRET_KEY` for server-side operations and `SUPABASE_SERVICE_ROLE_KEY` to bypass RLS for admin tasks.

### Video Processing Pipeline
```
YouTube URL → yt-dlp download → OpenAI Transcribe → AI Analyze → FFmpeg Extract Clips → Upload to R2
```

Job states: `pending` → `downloading` → `transcribing` → `analyzing` → `processing` → `completed`/`failed`

### Module Structure (klipklap-services)
```
src/
├── auth/              # JWT validation, guards, decorators
├── plans/             # Subscription plans (CRUD)
├── admin-users/       # Admin management with role guards
├── projects/          # Video clip projects
├── video-processing/  # Core clipping pipeline
│   └── ffmpeg/        # Bull queue for FFmpeg jobs
├── youtube/           # YouTube integration & OAuth
├── payments/          # Xendit webhooks
├── xendit/            # Payment links
├── general-config/    # App configuration
├── stats/             # Analytics endpoints
├── redis/             # Redis service
├── database/          # TypeORM config
└── migrations/        # DB migrations
```

### Database Tables (Supabase/PostgreSQL)
Key tables managed by TypeORM:
- `plans` - Subscription plans
- `admin_users` - Admin accounts
- `projects` - Video processing projects
- `clips` - Generated clips per project
- `youtube_connections` - OAuth tokens
- `youtube_upload_jobs` - Upload status tracking
- `general_config` - App settings

## Environment Configuration

### Required Supabase Variables
```bash
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SECRET_KEY=xxx        # Backend API operations
SUPABASE_SERVICE_ROLE_KEY=xxx  # Bypass RLS for admin
SUPABASE_JWT_SECRET=xxx        # JWT verification
```

### Required AI/Processing Variables
```bash
OPENAI_API_KEY=sk-xxx
AWS_ACCESS_KEY_ID=xxx          # R2 credentials
AWS_SECRET_ACCESS_KEY=xxx
AWS_S3_BUCKET=klipklap-videos
AWS_S3_ENDPOINT=https://xxx.r2.cloudflarestorage.com
REDIS_URL=redis://localhost:6379
YTDLP_API_URL=http://localhost:3002
YTDLP_API_KEY=secret-key
```

### Required Payment Variables
```bash
XENDIT_SECRET_KEY=xxx
XENDIT_WEBHOOK_TOKEN=xxx
XENDIT_ENVIRONMENT=development
```

## TypeScript Configuration Notes

### Frontend (Next.js)
- Target: ES2017, Module: esnext
- Path alias: `@/*` maps to root
- JSX: react-jsx
- Strict mode enabled

### Backend (NestJS)
- Target: ES2023, Module: NodeNext
- Decorators enabled (emitDecoratorMetadata)
- `noImplicitAny: false` (legacy)
- Output: `dist/`

## Common Development Tasks

### Adding a New API Endpoint
1. Create DTO in `src/{module}/dto/`
2. Add method to controller with `@ApiOperation` decorator
3. Implement in service
4. Add to module if new provider
5. Swagger auto-docs at `/api`

### Adding Database Migration
```bash
cd klipklap-services
npm run typeorm -- migration:generate -n src/migrations/migration_name
npm run typeorm -- migration:run
```

### Running Locally
1. Start Supabase local or use cloud project
2. Start Redis: `redis-server`
3. Start worker: `cd klipklap-worker && npm run dev`
4. Start services: `cd klipklap-services && npm run start:dev`
5. Start dashboard: `cd klipklap-dashboard && npm run dev`
6. Start landing: `cd klipklap && npm run dev`

### Testing Video Processing
```bash
# Check worker health
curl http://localhost:3002/health

# Get YouTube info
curl -X POST http://localhost:3002/info \
  -H "x-api-key: your-key" \
  -d '{"url": "https://youtube.com/watch?v=xxx"}'

# Create clip via API
curl -X POST http://localhost:4000/video-processing/clips \
  -H "Authorization: Bearer YOUR_JWT" \
  -d '{"youtubeUrl": "...", "numberOfClips": 3}'
```

## Deployment Notes

### Cloudflare Pages (Frontend)
- Uses `@opennextjs/cloudflare` adapter
- Build command: `npm run build`
- Output directory handled by adapter

### Docker (Services)
- Image: `ghcr.io/norld/klipklap-services:latest`
- Port mapping: `4000:3000`
- Volume mounts for logs and temp files
- Health check on `/health`

### Environment-Specific
- Development: Use `.env` files (gitignored)
- Production: Use `docker-compose.yml` with `.env.prod`

## Important Gotchas

1. **Auth tokens**: Frontend gets token from Supabase client, passes via `Authorization: Bearer` header
2. **Worker API key**: All worker endpoints except `/health` require `X-API-KEY` header
3. **FFmpeg concurrency**: Configured via `FFMPEG_QUEUE_CONCURRENCY` env var (default: 3)
4. **R2 not S3**: Must set `AWS_S3_ENDPOINT` for Cloudflare R2
5. **i18n**: klipklap uses Indonesian locale by default (`locale = 'id'`)
6. **TypeORM entities**: Located in `src/entities/`, run migrations after schema changes
7. **Job persistence**: Processing jobs cached in Redis with 24-hour TTL

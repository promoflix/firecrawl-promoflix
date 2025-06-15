# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Firecrawl is a web scraping API service that converts websites into clean markdown or structured data. It's deployed on Fly.io as `firecrawl-promoflix`.

## Commands

### Development
```bash
# Install dependencies (use pnpm)
pnpm install

# Start development (requires 3 terminals)
# Terminal 1: Redis
redis-server

# Terminal 2: Workers
pnpm run workers

# Terminal 3: API server
pnpm run start

# Or use Docker Compose (all-in-one)
docker compose up
```

### Testing
```bash
# Run all tests
pnpm run test

# Run tests without authentication (faster for development)
pnpm run test:local-no-auth

# Run specific test file
pnpm run test -- apps/api/src/__tests__/your-test.test.ts

# Run production tests (requires auth setup)
pnpm run test:prod
```

### Build & Production
```bash
# Build TypeScript
pnpm run build

# Start production server
pnpm run start:production

# Start production workers
pnpm run worker:production

# Deploy to Fly.io
flyctl deploy
```

## Architecture

### Service Architecture
- **API Server** (`apps/api/src/`): Express.js server handling HTTP requests
- **Workers** (`apps/api/src/services/queue-worker.ts`): BullMQ workers processing scraping jobs
- **Redis**: Job queue and rate limiting
- **Playwright Service** (`apps/playwright-service-ts/`): Browser automation for complex scraping

### Key Components
1. **Controllers** (`apps/api/src/controllers/`): API endpoints (v0 and v1)
   - `scrape`, `crawl`, `map`, `search`, `extract` controllers
   - Authentication and rate limiting middleware

2. **Scraper** (`apps/api/src/scraper/`):
   - `WebScraperDataProvider`: Main scraping orchestrator
   - Multiple scraping methods: Puppeteer, Playwright, fetch, SCRAPINGBEE, etc.
   - Fallback chain for reliability

3. **Services** (`apps/api/src/services/`):
   - Queue management (BullMQ)
   - Supabase integration (optional)
   - Rate limiting
   - Webhook handling

4. **Shared Libraries** (`apps/api/sharedLibs/`):
   - Go: HTML to Markdown conversion
   - Rust: HTML transformation, PDF parsing

### Data Flow
1. API receives request → Creates job in Redis queue
2. Worker picks up job → Executes scraping logic
3. Scraper tries multiple methods in fallback order
4. Results processed (markdown conversion, extraction, etc.)
5. Response sent via webhook or returned to polling client

### Environment Variables
Required:
- `NUM_WORKERS_PER_QUEUE=10`
- `PORT=3002`
- `HOST=0.0.0.0`
- `REDIS_URL=redis://localhost:6379`
- `REDIS_RATE_LIMIT_URL=redis://localhost:6379`

Optional but important:
- `SUPABASE_ANON_TOKEN`, `SUPABASE_URL`, `SUPABASE_SERVICE_TOKEN` (for auth)
- `OPENAI_API_KEY` (for LLM extraction)
- `PROXY_SERVER`, `PROXY_USERNAME`, `PROXY_PASSWORD` (for proxy support)

## Development Tips

### Running Tests
- Tests use a mock Redis server, no need to start Redis for tests
- Use `test:local-no-auth` during development for faster feedback
- E2E tests validate the entire scraping pipeline

### Debugging Scraping Issues
1. Check `apps/api/src/scraper/WebScraper/single_url.ts` for the scraping logic
2. Enable verbose logging by setting `DEBUG=*` environment variable
3. Test specific scrapers directly in `apps/api/src/scraper/WebScraper/scrapers/`

### Adding New Features
- Follow existing controller patterns in `apps/api/src/controllers/`
- Add queue jobs in `apps/api/src/services/queue-service.ts`
- Update both v0 and v1 APIs when applicable

### Performance Considerations
- Shared libraries (Go/Rust) handle performance-critical operations
- Rate limiting is enforced at multiple levels
- Workers can be scaled horizontally by increasing `NUM_WORKERS_PER_QUEUE`
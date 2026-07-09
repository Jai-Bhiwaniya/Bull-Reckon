# PART A - Writeup

## Project overview

BullReckon is a small monorepo containing a Next.js frontend and several backend microservices for market data, authentication and calculation services. The project focuses on showing real market data nad provides a comprehensive shared infrastructure layer for service orchestration, authentication, queue management, and real-time communication.

This document summarizes the architecture, technologies used, major design choices, challenges encountered and how they were addresssed, current progress, and a rough cost estimate for deploying the microservices.

---

## Approach & system architecture

- Monorepo: pnpm workspace + Turbo for orchestration and type checks
- Services split by responsibility (microservie style):
    - `apps/web` - Next.js frontend (App Router, React 10, TypeScript, Tailwind)
    - `apps/market_server` - market data service (Yahoo Finance client, historical/quote CRUD, normalization and caching)
    - `apps/auth_server` - authentication and user management (JWT / sessions, email)
    - `apps/calc_server` - calculation and trade eecution endpoints (trade execution, risk settings)
    - `apps/api_server` - higher-level API endpoints (portfolio, trades history)

- Sared Infrastructure Layer (`shared/` folder):
    - `BaseApp` - standardized Express.js application foundation with middleware, security, sessions, and WebSocket support
    - `QueueManager` - Redis-based job queue system with BullMQ for background processing (pending orders, notifications, emails)
    - `AuthServiceClient` - internal API client for cross-service authentication validation with fallback mechanisms
    -  `DatabaseManager` - MongoDB connection management with sinsleton pattern and connection pooling
    -  `WebSocketService` - real-time communication service with user authentication and room management
    - `InternalAPIClient` - pre-configured Axious client for secure inter-service commmunication
    - `TokenServis` - JWT token generation and management utilities

Data flow (high level):

User (browser) → Next.js (`apps/web`) → backend services (AUTH / MARKET / CALC / API) → shared infrastructure → market_server fetches Yahoo data

Key design choices:

- Keep market data retrieval seperate (market_server) so we can enforce the constraint: never synthesize market data - only forward Yahoo results or return an error.
- Provide a small client-side symbol search that reads a two-column `public/sp500.csv` (generated from a master CSV) for fast autocompletion without adding network latency to the searc UX.
- Use a small in-memory cache in `market_server` to reduce the duplicate Yahoo API calls during development.
- Implement a shared infrastructure layer to standardize service initialization, middleware, authentication and inter-service communication across all microservices.
- Use Redis queues for background job processing with graceful fallback to direct processing when Redis is unavailable.

---

## Technologies used

- Node.js >= 18, TypeScript 5.9
- pnpm workspace, Turbo (monorepo tasks)
- Next.js (app router) + React 19 for frontend
- TailwindCSS + shadcn UI primitives
- yahoo-finance2 in `market_server` for price/historical data
- Radix UI for toast/dialog primitives
- MongoDB with Mongoose for data persistence
- Redis with BullMQ for job queue management
- Socket.IO for real-time WebSocket communication
- JWT for authentication and inter-service security
- Express.js with comprehensice security middleware (helmet, ratr limiting, CORS)

---

## Shared Infrastructre Components

### BaseApp (`shared/baseApp.ts`)
Standardized Express.js application foundation that provides:
- Security middleware (helmet, rate limiting, CORS, mongo sanitization)
- Session management with MongoDB store
- File upload support with configurable limits
- WebSocket integration with Socket.IO
- Health check endpoints
- Queue manager integration
- Graceful shutdown handling
- Customizable middleware and routing

### AuthServiceClient (`shared/authService.client.ts`)
Internal authentication service client that provides:
- Cross-service token validation
- Fallback authentication using direct JWT verification
- Uder data retieval and caching
- Internal srvice security headers
- Timeout handling and error management

### DatabaseManager (`shared/dbManager.ts`)
MongoDB connection management with:
- Singleton pattern for connection reuse
- Connection pooling and retry logic
- Debug mode for development
- Event handling for connection state
- Graceful connection and disconnection

### WebSocketService (`shared/webSocketserver.ts`)
Real-time communicationn service featuring:
- JWT-base socket authentication
- User room management and tracking
- Personal and broadcast messaging
- Connection state monitoring
- Error handling and reconnection logic

### internalAPIClient (`shared/internalApi.client.ts`)
Pre-configured Axios client for secure inter-service communication with:
- Internal service authentication headers
- Timeout configuration
- standardized request/response andling

### TokenService (`shared/tokens.ts`)
JWT token mangemenet utilities providing:
- Access anf refreh token generation
- User-base token creation
- Configurable expiration times
- Role-based token payloads

---

## Major design and implementation notes

- The frontend `apps/web` expects backend services on the default ports configures in `apps/we/lib/config.ts`. See `setup.md` for exact ports and how to change them.

- The CSV generator `apps/web/scripts/generate_sp500_public_csv.cjs` converts the master S&P CSV to a two-column `public/sp500.csv` used by `SymbolSearch` component.
- The market microservices normalizes Yahoo responses and intentionally throes provider errors wen Yahoo returns no usable istorical data to abide by the "no synthetic data" requirement.
- A small `TradeModal` was added to the market page to place market orders. If an auth token exists it attempts to POST the order to `calc_server`; otherwise orders are persisted locally in localStorage.
- All microservices utilize the shared `BaseApp` class for consistent initialization, middleware configuration and feature enablement.
- The queue system automatically falls back to direct processing when Redis is unavailable, ensuring service resilience.
- Internal API calls between services use secure headers and token validation through the shared authentication client.
- WebSocket connection are authenticated and tracked per user, enabling real-time features like live quotes and trade notifications.

---

## Progress status (what's done)

- Monorepo setup, pnpm + turbo scripts (root `package.json`) - done
- Next.js frontend migrated into `apps/web` with App Router and core pages - done
- Symbol search UI (`SymbolSearch`) + generator script to create `public/sp500.csv` — done
- Market charts with 1M/3M/6M/1Y/YTD/5Y/Max periods and live quote polling — done
- Market microservice (Yahoo integration) hardened so it throws provider errors when real data is missing — done
- Added Navigation sidebar and improved Market page layout — done
- Implemented `TradeModal` to place market orders (backend call with fallback to localStorage) — done
- Fixed toasts (hook and mounting) and verified TypeScript checks — done
- **Shared Infrastructure Layer** — done
  - BaseApp class for standardized Express.js service initialization — done
  - QueueManager for Redis-based background job processing — done
  - AuthServiceClient for internal authentication validation — done
  - DatabaseManager for MongoDB connection management — done
  - WebSocketService for real-time communication — done
  - InternalApiClient for secure inter-service communication — done
  - TokenService for JWT token management — done

## How to verify locally

1. Follow `setup.md` to install dependencies and start backend services on recommended ports.
2. Ensure Redis and MongoDB are running for full functionality (or use fallback modes).
3. Start the web frontend with `pnpm -C apps/web run dev` and open `http://localhost:3000`.
4. Visit `/auth/login` and try login flows (mock users available for demo mode); toasts should appear for success/failure.
5. Test real-time features by opening multiple browser tabs and observing WebSocket connections.
6. Place test orders through the TradeModal to verify queue processing (check logs for queue vs. direct processing).
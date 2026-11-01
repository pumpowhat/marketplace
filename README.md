# BuildUp Marketplace

A multi-seller e-commerce marketplace with three roles: Customer, Seller and Admin.
Built as a learning project covering frontend, backend, database, authentication and authorization.
Monorepo: `apps/api` (NestJS), `apps/web` (Next.js), `docs/` (documentation).

## Getting started

Requires Node.js 20 or newer.

```bash
# API (http://localhost:3000, health check at /health)
cd apps/api
npm install
npm run start:dev

# Web (http://localhost:3001)
cd apps/web
npm install
npx next dev -p 3001
```

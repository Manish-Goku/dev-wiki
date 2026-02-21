# Dev Wiki — Katyayani Organics

Central knowledge base for all backend projects. Documents models, workflows, integrations, and session history.

## Projects

| Project | Stack | DB | Status |
|---------|-------|----|--------|
| [ko-sales-backend](ko-sales-backend/) | NestJS 10, TypeScript, Mongoose 7 | ko-sales-db (MongoDB) | Active |
| [CRM-Backend](CRM-Backend/) | Node.js (JS→TS), Express | CRM-Database (MongoDB) | Active |
| [Inventory-Management-Backend](Inventory-Management-Backend/) | Node.js, Express, BullMQ | CRM-Database (MongoDB) | Active |
| [utilities-go](utilities-go/) | Golang, Gin, GORM | MongoDB + PostgreSQL | Active |
| [utilities](utilities/) | Node.js, Express, RabbitMQ | CRM-Database (MongoDB) | Active |

## Structure

```
dev-wiki/
├── README.md              ← you are here
├── <project>/
│   ├── README.md          ← project overview + module index
│   ├── models/            ← schema docs (fields, indexes, relationships)
│   └── workflows/         ← flow docs (steps, modules involved, tags)
```

## Tags Reference

| Tag | Meaning |
|-----|---------|
| `#active` | Currently in use |
| `#wip` | Work in progress |
| `#completed` | Fully implemented and tested |
| `#needs-discussion` | Requires design decisions |
| `#cross-project` | Involves multiple projects |
| `#event-driven` | Uses event emitter / async |
| `#webhook` | External webhook integration |
| `#cron` | Scheduled job |
| `#auth-required` | JWT protected |
| `#public-endpoint` | No auth needed |

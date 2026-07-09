# ELF Binary Analyzer

A full-stack web platform for static analysis of ELF binaries. Upload any Linux ELF executable and get a structured, shareable report covering security mitigations, entropy analysis, symbol tables, and suspicious string extraction — all without a local toolchain.

---

## What It Does

- **ELF Header parsing** — class, endianness, entry point, machine type, OS/ABI
- **Section & segment table** — full breakdown of all sections and program headers
- **Symbol table** — imported and exported functions with binding and type info
- **Security mitigation detection** — NX, PIE, Full/Partial RELRO, Stack Canary, RPATH
- **Entropy analysis** — per-section Shannon entropy with packed/encrypted section flagging (threshold: 7.0)
- **Suspicious string extraction** — regex-matched IPs, URLs, shell commands, file paths, crypto constants
- **Persistent history** — every analysis run stored per user, searchable and re-openable
- **Shareable reports** — public report links require no login to view
- **REST API** — full programmatic access via API key

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Tailwind CSS, TanStack Query, Recharts |
| Backend | Node.js 20, Express 5, Prisma ORM |
| Queue | Bull + Redis 7 |
| Analysis Worker | Python 3.11, pyelftools |
| Database | PostgreSQL 16 |
| Infrastructure | Docker Compose, Nginx |

---

## Project Structure

```
elf-binary-analyzer/
├── api/                  # Express API server
│   ├── src/
│   │   ├── routes/       # Auth, upload, reports, keys
│   │   ├── middleware/   # requireAuth, sanitize, rateLimiter
│   │   ├── queue/        # Bull producer
│   │   └── jobs/         # Cleanup cron
│   ├── prisma/
│   │   └── schema.prisma
│   └── Dockerfile
├── worker/               # Python analysis worker
│   ├── analyzer/
│   │   ├── elf_header.py
│   │   ├── sections.py
│   │   ├── segments.py
│   │   ├── symbols.py
│   │   ├── mitigations.py
│   │   ├── entropy.py
│   │   └── strings.py
│   ├── worker.py
│   └── Dockerfile
├── frontend/             # React + Vite app
│   ├── src/
│   │   ├── pages/        # Login, Register, Dashboard, Analyze, Report, Settings
│   │   ├── components/
│   │   │   └── report/   # OverviewTab, SectionsTab, SymbolsTab, MitigationsTab, EntropyTab, StringsTab
│   │   └── context/      # AuthContext
│   └── Dockerfile
└── docker/
    ├── docker-compose.yml
    ├── docker-compose.prod.yml
    └── nginx/
        └── nginx.conf
```

---

## Getting Started

### Prerequisites

- Docker and Docker Compose
- Node.js 20+ (for local development without Docker)
- Python 3.11+ (for local worker development)

### Setup

**1. Clone the repo**
```bash
git clone https://github.com/DaivyaPatel/elf-binary-analyzer.git
cd elf-binary-analyzer
```

**2. Configure environment**
```bash
cp .env.example .env
# Edit .env and fill in all required values
```

**3. Start all services**
```bash
docker compose -f docker/docker-compose.yml up --build
```

**4. Run database migrations**
```bash
docker compose exec api npx prisma migrate deploy
```

**5. Open the app**
```
http://localhost:80
```

---

## Environment Variables

```env
DATABASE_URL=postgresql://user:pass@postgres:5432/elfanalyzer
REDIS_URL=redis://redis:6379
JWT_SECRET=<minimum 64 character random hex string>
UPLOAD_DIR=/app/uploads
MAX_FILE_SIZE=52428800
NODE_ENV=development
```

---

## API Reference

All endpoints require `Authorization: Bearer <token>` unless marked public.

| Method | Route | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | None | Create account |
| POST | `/api/auth/login` | None | Returns JWT |
| POST | `/api/upload` | Required | Upload ELF binary, returns jobId |
| GET | `/api/jobs/:id` | Required | Poll job status + result |
| GET | `/api/reports` | Required | List analysis history |
| GET | `/api/reports/:id` | Required | Fetch full report JSON |
| DELETE | `/api/reports/:id` | Required | Delete report + binary |
| PATCH | `/api/reports/:id/share` | Required | Toggle public sharing |
| GET | `/api/public/:id` | None | View public report |
| POST | `/api/keys/generate` | Required | Generate API key |

### Example: Upload and analyze a binary

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"yourpassword"}' \
  | jq -r '.token')

# Upload binary
JOB_ID=$(curl -s -X POST http://localhost:3000/api/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/binary" \
  | jq -r '.jobId')

# Poll for result
curl http://localhost:3000/api/jobs/$JOB_ID \
  -H "Authorization: Bearer $TOKEN"
```

---

## Analysis Report Structure

```json
{
  "id": "uuid",
  "filename": "binary",
  "elfHeader": {
    "elfClass": "ELF64",
    "dataEncoding": "LSB",
    "osABI": "UNIX - System V",
    "entryPoint": "0x401080",
    "machineType": "Advanced Micro Devices X86-64",
    "elfType": "ET_EXEC"
  },
  "sections": [ { "name": ".text", "type": "SHT_PROGBITS", "address": "0x401000", "size": 4096, "flags": "AX" } ],
  "symbols": [ { "name": "printf", "binding": "GLOBAL", "type": "FUNC", "imported": true } ],
  "mitigations": {
    "nx": true,
    "pie": false,
    "relro": "partial",
    "stackCanary": true,
    "rpath": false
  },
  "entropy": [ { "sectionName": ".text", "entropy": 5.82, "packed": false } ],
  "suspiciousStrings": [ { "string": "/bin/sh", "category": "Shell Command", "section": ".rodata" } ]
}
```

---

## Security

- Analysis workers run in isolated Docker containers with no network access
- Binary files are bind-mounted read-only into worker containers
- Containers are destroyed after each job
- Passwords hashed with bcrypt (cost factor 12)
- API keys stored as SHA-256 hashes, never in plaintext
- All routes protected with Helmet.js security headers
- File magic byte validation before any processing
- Upload directory never exposed as a static route

See [Security & Access Document](docs/03_Security_Access_Document.docx) for the full threat model.

---

## Limits

| Resource | Limit |
|---|---|
| Max upload size | 50MB |
| Uploads per hour | 20 per user |
| Analysis timeout | 30 seconds |
| Login attempts | 10 per 15 min per IP |
| Public report requests | 60 per min per IP |

---

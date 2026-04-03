# Commands Reference

All build/test/lint/deploy commands per sub-project.

## MessageForge.Backend (Go)

```bash
# Build & Run
go build ./cmd/messenger
go run ./cmd/messenger

# Test
go test ./... -v            # Verbose
go test -race ./...         # Race detection
go test -cover ./...        # Coverage

# Quality
go vet ./...                # Static analysis
gofmt -w .                  # Format
golangci-lint run ./...     # Lint

# Infrastructure
docker compose up -d        # Start PostgreSQL
docker compose down         # Stop services
bash scripts/setup.sh       # First-time setup
```

## MessageForge.Salesforce (Apex/LWC)

```bash
# Deploy & Retrieve
sf project deploy start --source-dir force-app
sf project retrieve start --source-dir force-app

# Test
sf apex run test --test-level RunLocalTests --code-coverage --wait 30
sf apex run test --class-names ClassName

# Scratch Org
sf org create scratch --definition-file config/project-scratch-def.json --alias dev
sf org login web --alias dev

# Debug
sf apex get log --number 1

# Package (2GP)
sf package version create --package "MessengerIntegration" --release-type Major
sf package version list
```

## MessageForge.PoC (Go + React)

```bash
# Full build & run
make run PORT=8888

# Backend only (watch)
make dev PORT=8888

# Frontend
cd web && npm install && npm run dev    # Dev server
cd web && npm run build                 # Production build
cd web && npm run lint                  # ESLint

# Test
make test                               # Go tests
cd web && npx playwright test           # E2E tests

# Clean
make clean
```

# Alexandrie Setup Guide

This guide explains how to set up, configure, and deploy the Alexandrie stack.

It covers both **local development** and **production deployment**, including how to configure environment variables, connect the different services, and handle HTTPS with a reverse proxy.

---

## Overview of the Stack

The Alexandrie stack consists of four main services, defined in the `docker-compose.yml` file:

| Service      | Description                                                                    |
| ------------ | ------------------------------------------------------------------------------ |
| **MySQL**    | Relational database used to store user data and metadata.                      |
| **RustFS**   | S3-compatible object storage used for file and media storage.                  |
| **Backend**  | The Go-based API server providing authentication, storage, and access control. |
| **Frontend** | A Nuxt 4 web app that communicates with the backend via REST API.              |

All services are connected through a dedicated Docker network (`alexandrie-network`).

---

## Quick Start

For local testing, you can run Alexandrie directly using the provided `docker-compose.yml`:

Download the `docker-compose.yml` file from the repository: [https://github.com/Smaug6739/Alexandrie/blob/main/docker-compose.yml](https://github.com/Smaug6739/Alexandrie/blob/main/docker-compose.yml) and run:

```bash
docker compose up
```

Once the containers are running:

- Frontend → <http://localhost:8200>
- Backend API → <http://localhost:8201>
- Object Storage (RustFS) → <http://localhost:9000>

_Note: No default account is created. You can register a new user directly from the frontend._

## Configuration (Environment Variables)

To customize the setup (ports, domains, security, feature flags, production deployment), you can create a `.env` file next to `docker-compose.yml`.

The `.env` file template can be found here: [https://github.com/Smaug6739/Alexandrie/blob/main/.env.example](https://github.com/Smaug6739/Alexandrie/blob/main/.env.example)

Only the variables defined in `.env.example` are officially supported.

### 1. Public URLs & Global Access

```env
FRONTEND_URL=http://localhost:8200
API_URL=http://localhost:8201
CDN_URL=http://localhost:9005

CDN_ENDPOINT=/alexandrie/
```

These variables define how clients (browsers) reach your services.

- `FRONTEND_URL`: The public URL for the frontend service.
- `API_URL`: The public URL for the backend API service.
- `CDN_URL`: The public URL for the object storage service (RustFS).
- `CDN_ENDPOINT`: The path prefix under which files are exposed. You probably want to keep this as /alexandrie/ to match the bucket name, change it only if you changed the bucket name or if you have configured a reverse proxy path.  
  Final URL are constructed as: `<CDN_URL><CDN_ENDPOINT><object-path>`  
  If you have a problem with files URL just open devtools in your browser and check the actual URL used to load files and then adjust these variables accordingly.

### 2. Application Feature Flags

```env
CONFIG_DISABLE_LANDING=false
CONFIG_DISABLE_SIGNUP=false
CONFIG_DISABLE_NATIVE_LOGIN=false
```

These variables control specific features of the application (useful for private deployments, internal tools etc.).

- `CONFIG_DISABLE_LANDING`: If set to `true`, the landing page is disabled, and users are redirected to the login page. The default is `false`.
- `CONFIG_DISABLE_SIGNUP`: If set to `true`, new user registrations are disabled (in frontend + backend for security). The default is `false`.
- `CONFIG_DISABLE_NATIVE_LOGIN`: If set to `true`, the native username/password login method is disabled. This is useful if you want to enforce OIDC or other external authentication methods. The default is `false`.

### 3. Backend API — Runtime & Security

```env
BACKEND_EXTERNAL_PORT=8201
COOKIE_DOMAIN=localhost:8200
ALLOW_UNSECURE=true
GIN_MODE=debug

# Frontend: frontend/.env
PORT=8200
NUXT_PUBLIC_BASE_API=http://localhost:8201
NUXT_PUBLIC_BASE_CDN=http://localhost:9000/alexandrie
NUXT_PUBLIC_BASE_URL=http://localhost:8200

# 3. Setup database

# Create MySQL database
mysql -u root -p
CREATE DATABASE alexandrie;
CREATE USER 'alexandrie'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON alexandrie.* TO 'alexandrie'@'localhost';
FLUSH PRIVILEGES;

# 4. Start MinIO (object storage)
minio server ./minio-data --console-address :9001 --address "localhost:9000"

# 5. Start backend
cd backend
go run main.go
# Migrations run automatically

# 6. Start frontend (in new terminal)
cd frontend
bun dev
```

### 9.3 Project Commands

#### Backend Commands

```bash
# Run development server
go run main.go

# Build binary
go build -o alexandrie main.go

# Run tests
go test ./tests/... -v

# Run specific test
go test ./tests/auth_test.go -v

# Format code
go fmt ./...

# Vet code (static analysis)
go vet ./...

# Install new dependency
go get github.com/package/name

# Update dependencies
go get -u ./...
go mod tidy
```

#### Frontend Commands

```bash
# Development server (with HMR)
bun dev

# Build for production
bun run build

# Preview production build
bun run preview

# Lint code
bun run lint

# Fix linting issues
bun run lint:fix

# Lint CSS
bun run lint:css

# Fix CSS issues
bun run lint:css:fix

# Install dependency
bun add package-name

# Install dev dependency
bun add -d package-name

# Update dependencies
bun update
```

#### Makefile Commands (Linux)

```bash
# Start all services
make all

# Start backend only
make backend

# Start frontend only
make frontend

# Start MinIO only
make minio

# Stop all services
make stop
```

### 9.4 Database Migrations

#### Creating a Migration

```bash
# Install migrate CLI
go install -tags 'mysql' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Create new migration
cd backend/migrations
migrate create -ext sql -dir . -seq migration_name

# This creates:
# - 000012_migration_name.up.sql
# - 000012_migration_name.down.sql
```

_Note: This tool is optional you can create manually the file but please follow the naming conventions_

#### Writing Migrations

```sql
-- 000012_add_feature.up.sql
ALTER TABLE nodes ADD COLUMN new_field VARCHAR(100);
CREATE INDEX idx_new_field ON nodes(new_field);

-- 000012_add_feature.down.sql
DROP INDEX idx_new_field ON nodes;
ALTER TABLE nodes DROP COLUMN new_field;
```

#### Running Migrations

```bash
# Migrations run automatically on app start
# Or run manually:

# Apply all migrations
migrate -database "mysql://user:pass@tcp(localhost:3306)/alexandrie" -path migrations up

# Rollback last migration
migrate -database "..." -path migrations down 1

# Goto specific version
migrate -database "..." -path migrations goto 10

# Check current version
migrate -database "..." -path migrations version
```

> [!WARNING]
> Migrations down are not fully implemented yet

### 9.5 Code Quality

#### Linting

```bash
# Backend
go vet ./...
golangci-lint run  # If installed

# Frontend
bun run lint        # ESLint
bun run lint:css    # Stylelint
```

#### Formatting

```bash
# Backend
go fmt ./...

# Frontend (automatic with ESLint)
bun run lint:fix        # ESLint fix
bun run lint:css:fix    # Stylelint fix
```

### 9.6 Git Workflow

```bash
# Create feature branch
git checkout -b feature/new-feature

# Make changes and commit
git add .
git commit -m "feat: add new feature"

# Push to remote
git push origin feature/new-feature

# Create pull request on GitHub

# After merge, update main
git checkout main
git pull origin main
```

**Commit Message Convention:**

```
feat: Add new feature
fix: Fix bug in authentication
docs: Update README
style: Format code
refactor: Refactor node service
test: Add tests for permissions
chore: Update dependencies
```

### 9.7 Debugging

#### Backend Debugging

```go
// Add debug logging
import "log"

log.Println("Debug:", variable)
log.Printf("User ID: %s", userId)

// Use Delve debugger
go install github.com/go-delve/delve/cmd/dlv@latest

# Start with debugger
dlv debug main.go

# Set breakpoint
(dlv) break main.main
(dlv) continue
```

#### Frontend Debugging

```typescript
// Console logging
console.log('Debug:', data)
console.table(array)

// Vue DevTools (browser extension)
// Inspect component state, Pinia stores, routing

// Network tab (browser)
// Inspect API calls, responses, timing
```

#### Database Debugging

```bash
# Connect to MySQL
mysql -u alexandrie -p alexandrie

# Check tables
SHOW TABLES;

# Inspect data
SELECT * FROM users LIMIT 10;
SELECT * FROM nodes WHERE user_id = '123';

# Check indexes
SHOW INDEX FROM nodes;

# Explain query performance
EXPLAIN SELECT * FROM nodes WHERE name LIKE '%search%';
```

### 9.9 Performance Monitoring

#### Backend Profiling

```go
import _ "net/http/pprof"

// Enable profiling endpoint
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()

// Access profiles:
// http://localhost:6060/debug/pprof/
// CPU: /debug/pprof/profile?seconds=30
// Memory: /debug/pprof/heap
// Goroutines: /debug/pprof/goroutine
```

#### Frontend Performance

```typescript
// Performance API
const perfData = performance.getEntriesByType('navigation')[0]
console.log('Load time:', perfData.loadEventEnd - perfData.fetchStart)

// Lighthouse (Chrome DevTools)
// Run audit for performance, accessibility, SEO

// Vue DevTools Performance tab
// Track component render time
```

---

## 10. Configuration Reference

### 10.1 Setup application

For details please see [setup guide.md](./setup.md)

### 10.2 Performance Tuning

#### Caching Strategy

```go
// Example: Redis caching (not currently implemented)

import "github.com/go-redis/redis/v8"

// Cache frequently accessed nodes
func GetNodeWithCache(nodeId Snowflake) (*Node, error) {
    // Try cache first
    cached, err := redis.Get(ctx, "node:"+nodeId).Result()
    if err == nil {
        return unmarshalNode(cached), nil
    }
    
    // Cache miss, query database
    node := getNodeFromDB(nodeId)
    
    // Store in cache (expire in 5 minutes)
    redis.Set(ctx, "node:"+nodeId, marshal(node), 5*time.Minute)
    
    return node, nil
}
```

## Other

### A. Glossary

- **Node**: Unified entity representing workspace, category, document or ressource
- **Snowflake**: Distributed unique ID generation algorithm (inspired from <https://discord.com/developers/docs/reference#snowflakes>)
- **Permission Level**: Numeric access level (0-4) determining user capabilities
- **S3**: Simple Storage Service, object storage protocol
- **JWT**: JSON Web Token, stateless authentication mechanism
- **CORS**: Cross-Origin Resource Sharing, browser security feature
- **PWA**: Progressive Web App, installable web application
- **SSR**: Server-Side Rendering
- **SPA**: Single-Page Application
- **ORM**: Object-Relational Mapping

### B. Common Issues and Solutions

#### Backend won't start

```bash
# Check MySQL connection
mysql -u alexandrie -p

# Check port availability
lsof -i :8201

# Check environment variables
go run main.go  # Look for error messages

# Reset database
DROP DATABASE alexandrie;
CREATE DATABASE alexandrie;
# Restart backend (migrations will recreate tables)
```

#### Frontend build errors

```bash
# Clear cache
rm -rf .nuxt node_modules bun.lock
bun install
bun dev

# Check Node version
node --version  # Should be 18+

# Check environment variables
cat .env
```

### C. External Resources

- **Go Documentation**: <https://go.dev/doc/>
- **Gin Framework**: <https://gin-gonic.com/docs/>
- **Nuxt Documentation**: <https://nuxt.com/docs>
- **Vue.js Guide**: <https://vuejs.org/guide/>
- **MySQL Reference**: <https://dev.mysql.com/doc/>
- **MinIO Documentation**: <https://min.io/docs/>
- **JWT.io**: <https://jwt.io/>
- **CodeMirror**: <https://codemirror.net/>

### D. Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for development setup and contribution guidelines.

### E. License

This project is licensed under the MIT License. See [LICENSE](../LICENSE) for details.
JWT_SECRET=your-secure-jwt-secret-key-change-this-in-production

```

These variables configure the backend API service.

- `BACKEND_EXTERNAL_PORT`: The port exposed on your host machine for the Backend API service.
- `COOKIE_DOMAIN` (IMPORTANT): This variable defines where authentication cookies are valid.  
  It should match with the common domain used by both frontend and backend services.  
  Example: If your frontend is at `frontend.yourdomain.com` and your backend API is at `api.yourdomain.com`, then set `COOKIE_DOMAIN=yourdomain.com`.  
  **Rules:**
  - Must match with the highest common domain between the frontend and the backend. See #286 for more details.
  - Must NOT include protocol (http:// or https://)
  - If misconfigured, login will appear to work but users will be instantly logged out
- `ALLOW_UNSECURE`: If set to `true`, HTTP connections are allowed (useful for local and dev). In production, it is recommended to set this to `false` to enforce HTTPS.
- `JWT_SECRET`: Secret key used to sign JWT tokens. You should change this to a random long string correcly generated (with openssl or similar) in production for security. Changing this value will invalidate all existing sessions.

### 4. Database — MySQL

```env
MYSQL_EXTERNAL_PORT=3307
MYSQL_DATABASE=alexandrie
MYSQL_USER=alexandrie
MYSQL_PASSWORD=password
MYSQL_ROOT_PASSWORD=rootpassword
```

These variables configure the MySQL database service.

- `MYSQL_EXTERNAL_PORT`: The port exposed on your host machine for the MySQL service.
- `MYSQL_DATABASE`: The name of the default database created for Alexandrie.
- `MYSQL_USER`: The username for the default Alexandrie database user.
- `MYSQL_PASSWORD`: The password for the default Alexandrie database user.
- `MYSQL_ROOT_PASSWORD`: The password for the MySQL root user.

### 5. Object Storage — S3 (RustFS / MinIO-compatible) (optional, leave empty to disable)

```env
RUSTFS_EXTERNAL_PORT=9005

RUSTFS_ACCESS_KEY=alexandrie-key
RUSTFS_SECRET_KEY=alexandrie-secret

MINIO_BUCKET=alexandrie
```

These variables configure the RustFS object storage service.

- `RUSTFS_EXTERNAL_PORT`: The port exposed on your host machine for the RustFS service.
- `RUSTFS_ACCESS_KEY`: The access key for the RustFS S3-compatible API.
- `RUSTFS_SECRET_KEY`: The secret key for the RustFS S3-compatible API.
- `MINIO_BUCKET`: The name of the default bucket created for Alexandrie. Note that this bucket name should match the `CDN_ENDPOINT` variable in the backend configuration. By default, it is set to `alexandrie` and the backend will create `alexandrie` and `alexandrie-backups` buckets.

### 6. Email — SMTP (optional)

```env
SMTP_HOST=
SMTP_MAIL=
SMTP_PASSWORD=
```

Used only for password reset emails. Leave empty to disable email features (default).

- `SMTP_HOST`: The SMTP server host.
- `SMTP_MAIL`: The email address used to send emails.
- `SMTP_PASSWORD`: The password for the SMTP email account.

### 7. Frontend — Nuxt

```env
FRONTEND_EXTERNAL_PORT=8200
```

The port exposed on the host for the frontend container.

In production, this is usually only accessed by a reverse proxy.

## Production Deployment Notes

In production, you should:

- NOT expose MySQL or RustFS directly
- Use a reverse proxy (Nginx, Traefik, Caddy)
- Terminate HTTPS at the proxy

## Additional Configuration & Advanced Setup

### Custom configuations - config.toml

For advanced configuration options for backend you can create your own `config.toml` file with the same structure as the default one (in `backend/config.toml`) and add the following environment variable to your `.env` file:

```env
CONFIG_PATH=./config.toml # Path to your custom config.toml file
```

This allows you to customize advanced settings not covered by environment variables (such as default values, logging levels, supported files etc).

### OIDC Integration

Open ID Connect (OIDC) is supported since version 8.5.0. You can setup up to 10 providers using environment variables to the backend.

First you have to generate client ID and client secret from your provider (refer to their doc or ask questions in discussions / discord)

Then you have to configure the redirect URI (in the provider interface with this value): `https://your-alexandrie-frontend/login/oidc/callback`

To config a provider you have to fill 4 variables:

```env
OIDC_{i}_CONFIG_URL=https://yourdomain.com/.well-known/openid-configuration
OIDC_{i}_CLIENT_ID=
OIDC_{i}_CLIENT_SECRET=
OIDC_{i}_PROVIDER_NAME= # e.g. Google, Discord, Microsoft...
```

_Replace `{i}` with a number between 1 to 10_

To setup these variables to your configuration you have to edit the `docker-compose.yml` and add variables to the backend config (direct values or a reference to your env file) like this for example:

```yml
OIDC_1_CONFIG_URL: ${OIDC_1_CONFIG_URL}
OIDC_1_CLIENT_ID: ${OIDC_1_CLIENT_ID}
OIDC_1_CLIENT_SECRET: ${OIDC_1_CLIENT_SECRET}
OIDC_1_PROVIDER_NAME: ${OIDC_1_PROVIDER_NAME}
```

### S3 / Minio — TLS, HTTPS and Custom Certificate Authorities

Alexandrie connects to the S3-compatible storage (RustFS / MinIO...) using an internal backend client.
This client supports HTTPS, custom certificate authorities (CA), and explicit TLS configuration.

This section explains how TLS is handled and how to configure it correctly.

#### HTTPS vs HTTP (MINIO_SECURE)

```env
MINIO_SECURE=true|false
```

This variable controls whether HTTPS (TLS) is used at all when the backend connects to the S3 endpoint. It must explicitly match how your S3 server is exposed.

- `true` → HTTPS is used (default)
- `false` → HTTP is used (no TLS)

#### Custom Certificate Authorities (MINIO_CA_PATH)

If your S3 endpoint uses a certificate signed by an internal or private certificate authority, you must explicitly provide this CA to the backend.

```env
MINIO_CA_PATH=/path/to/ca-certificate.pem
```

- The file must contain the CA certificate (root or intermediate), in PEM format
- This CA will be added to the system trust store used by the S3 client
- This is the recommended and secure way to handle internal PKI

_Note: if you are using docker containers, you must mount this file into the backend container using a volume in `docker-compose.yml`._

_TLS Verification Bypass (Not Recommended)_

```env
MINIO_INSECURE_TLS=true|false
```

This option disables TLS certificate verification while still using HTTPS.

- The connection remains encrypted
- Certificate validation (CA + hostname) is skipped

⚠️ Security warning  
This option should only be used for:

- temporary debugging
- local testing environments
- It is not recommended for production use.

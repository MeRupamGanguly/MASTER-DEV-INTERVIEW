# Migrations Handling

```go
type Book struct{
  ISIN string
  Name String
  AuthorID string
}
type Author struct{
  ID string
  Name String
}
```
Assume One Author decided later, he does not want to Disclose its Real Name to Spread. So we can Serve Frontend by Alias instead of Real Name. Without Changing Book Class/Struct, we can add Alias in Author Struct. By that, Existing Authors present in DB will not be affected as Frontend will Change Name only when it Founds that - Alias field is not empty.
```go
type Book struct{
  ISIN string
  Name String
  AuthorID string
}
type Author struct{
  ID string
  Name String
  Alias String
}
```
Asume A project structure:
```bash
project-root/
├── cmd/                # Entry points (main.go for services, CLI tools)
├── internal/           # Core application logic
│   ├── models/         # GORM models (Book, Author)
│   ├── repository/     # DB access layer (queries, CRUD)
│   ├── services/       # Business logic (rules, orchestration)
│   └── api/            # HTTP handlers, REST/GraphQL endpoints
├── migrations/         # Versioned SQL migration files
│   ├── 0001_init.up.sql
│   ├── 0001_init.down.sql
│   ├── 0002_add_alias.up.sql
│   └── 0002_add_alias.down.sql
├── configs/            # Config files (env, YAML, JSON)
├── pkg/                # Shared utilities
├── scripts/            # Helper scripts (run migrations, seed data)
└── go.mod / go.sum
```

Database schema must evolve to match the new struct. For that We create migration files:

```sql
-- 0002_add_alias.up.sql
ALTER TABLE authors ADD COLUMN alias VARCHAR(255);

-- 0002_add_alias.down.sql
ALTER TABLE authors DROP COLUMN alias;
```
Up script applies the change, Down script reverts it. These files are versioned in Git, just like code (Schema changes tracked, versioned, committed, reviewed).

When code is pushed, the pipeline runs:
- Build → compile Go code, run unit tests.
- Migrate DB → run migration tool (golang-migrate) against staging/prod DB.
- Deploy → release new container/image to Kubernetes or VM.
- Smoke Test → verify API endpoints work with new schema.

Example Jenkinsfile:
```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'go build ./cmd/service'
            }
        }

        stage('Test') {
            steps {
                sh 'go test ./...'
            }
        }

        stage('Migrate DB') {
            steps {
                sh 'migrate -path ./migrations -database $DB_URL up'
            }
        }

        stage('Deploy') {
            steps {
                sh './scripts/deploy.sh'
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl -f http://localhost:8080/health'
            }
        }
    }
}
```
DevOps engineers trigger rollback if needed. Rollbacks are not automatic (risk of data loss).


Migration tools track schema state in a special table (schema_migrations).

Example table:
- version → ID of migration applied.
- applied_at → timestamp of application.
- dirty → flag if migration failed halfway.

Rollbacks are step‑based:
- down 1 → revert last migration. down 1 means: revert the last applied migration file (the highest version number). It does not care how many structs you changed in Go code at that commit. Migration tools only track migration files, not your Go structs.
- down 2 → revert last two migrations.

```sql
-- 0003_add_alias_and_genre.up.sql
ALTER TABLE authors ADD COLUMN alias VARCHAR(255);
ALTER TABLE authors ADD COLUMN email VARCHAR(255);
ALTER TABLE books ADD COLUMN genre VARCHAR(255);

-- 0003_add_alias_and_genre.down.sql
ALTER TABLE authors DROP COLUMN alias;
ALTER TABLE authors DROP COLUMN email;
ALTER TABLE books DROP COLUMN genre;
```
When you run up, all three schema changes are applied together. When you run down 1, the tool executes the entire down script for migration 0003. That means all three columns are dropped together, even though they came from multiple struct changes.

Rollback in migration tools is step‑based. down 1 reverts the last migration file, not individual struct changes. If multiple struct changes were bundled into one migration, rollback will revert all of them together. That’s why in production we try to keep migrations small and focused — one logical schema change per migration file. This makes rollbacks safer and more predictable.

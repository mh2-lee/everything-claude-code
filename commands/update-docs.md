# Update Documentation

Sync documentation from source-of-truth:

1. Read `build.gradle.kts` tasks section
   - Generate tasks reference table
   - Include custom task descriptions

2. Read `application.yml` and `application-*.yml`
   - Extract all configuration properties
   - Document purpose and format

3. Generate `docs/CONTRIB.md` with:
   - Development workflow
   - Available Gradle tasks
   - Environment setup
   - Testing procedures

4. Generate `docs/RUNBOOK.md` with:
   - Deployment procedures
   - Monitoring and alerts (Actuator endpoints)
   - Common issues and fixes
   - Rollback procedures

5. Identify obsolete documentation:
   - Find docs not modified in 90+ days
   - List for manual review

6. Show diff summary

## Source Files to Analyze

```bash
# Build configuration
build.gradle.kts

# Application configuration
src/main/resources/application.yml
src/main/resources/application-local.yml
src/main/resources/application-prod.yml

# Database migrations
src/main/resources/db/migration/

# Docker configuration
docker-compose.yml
Dockerfile
```

## Example Output Format

```markdown
## Available Gradle Tasks

| Task | Description |
|------|-------------|
| `./gradlew build` | Build the project |
| `./gradlew test` | Run unit tests |
| `./gradlew bootRun` | Start the application |
| `./gradlew bootJar` | Create executable JAR |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection URL |
| `REDIS_HOST` | No | Redis host (default: localhost) |
| `JWT_SECRET` | Yes | JWT signing secret |
```

Single source of truth: `build.gradle.kts` and `application.yml`

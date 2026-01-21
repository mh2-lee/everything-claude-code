# Update Codemaps

Analyze the codebase structure and update architecture documentation:

1. Scan all source files for imports, dependencies, and relationships
2. Generate token-lean codemaps in the following format:
   - `codemaps/architecture.md` - Overall architecture
   - `codemaps/backend.md` - Backend structure (Controllers, Services, Repositories)
   - `codemaps/data.md` - Data models, entities, and DTOs

3. Calculate diff percentage from previous version
4. If changes > 30%, request user approval before updating
5. Add freshness timestamp to each codemap
6. Save reports to `.reports/codemap-diff.txt`

## Analysis Commands

```bash
# List all Kotlin source files
find src/main/kotlin -name "*.kt" | head -50

# Find all Spring components
grep -r "@Controller\|@Service\|@Repository\|@Component" src/main/kotlin

# List all entities
grep -r "@Entity" src/main/kotlin

# Show package structure
tree src/main/kotlin -d -L 3
```

## Codemap Format Example

```markdown
# Backend Codemap

Updated: 2024-01-15

## Controllers
- UserController.kt → /api/users
- OrderController.kt → /api/orders
- MarketController.kt → /api/markets

## Services
- UserService.kt
  - depends on: UserRepository, EmailService
- OrderService.kt
  - depends on: OrderRepository, UserService, PaymentService

## Repositories
- UserRepository.kt → users table
- OrderRepository.kt → orders table
```

Focus on high-level structure, not implementation details.

---
name: code-review
description: Review code for quality, readability, and best practices. Use when reviewing code changes, pull requests, or when asked to analyze code quality to ensure adherence to software engineering best practices including: (1) Code quality and maintainability (naming, structure, DRY, SOLID principles), (2) Security vulnerabilities (injection, authentication, secrets), (3) Performance issues (N+1 queries, memory leaks, inefficient algorithms), (4) Error handling and edge cases, (5) Testing coverage and quality, (6) Language-specific best practices, (7) Architecture and design patterns. Apply when reviewing PRs, refactoring code, or when asked for code quality feedback.
---

# Code Review Guidelines

## Review Workflow

### 1. Understand Context
- Read PR/commit description and linked issues
- Understand the problem being solved
- Check if implementation matches requirements
- Review related code and dependencies

### 2. Systematic Review Order
1. **Architecture & Design** - High-level structure and patterns
2. **Security** - Vulnerabilities and sensitive data handling
3. **Correctness** - Logic bugs and edge cases
4. **Performance** - Efficiency and scalability
5. **Code Quality** - Readability, maintainability, best practices
6. **Testing** - Coverage and test quality

### 3. Provide Feedback
- Prioritize issues: Critical → Warnings → Suggestions
- Explain the "why" behind recommendations
- Provide code examples when helpful
- Be constructive and respectful

## Code Quality

### Naming Conventions

**❌ Bad:**
```typescript
function getData(x: number): any {
  const d = new Date();
  const temp = x * 2;
  return temp;
}
```

**✅ Good:**
```typescript
function calculateDiscountedPrice(originalPrice: number): number {
  const today = new Date();
  const discountedPrice = originalPrice * 0.5;
  return discountedPrice;
}
```

**Guidelines:**
- Use descriptive names that reveal intent
- Functions: verbs (getUserById, calculateTotal, validateEmail)
- Variables: nouns (user, totalPrice, isValid)
- Booleans: predicates (isActive, hasPermission, canEdit)
- Constants: UPPER_SNAKE_CASE (MAX_RETRY_COUNT, API_BASE_URL)
- Avoid abbreviations unless universally understood (http, url, api)

### Function Design

**❌ Bad (too many responsibilities):**
```typescript
function processUser(userData: any) {
  // Validate
  if (!userData.email) throw new Error("Invalid");

  // Hash password
  const hash = bcrypt.hash(userData.password, 10);

  // Save to database
  const user = db.users.insert({ ...userData, password: hash });

  // Send email
  sendWelcomeEmail(user.email);

  // Log analytics
  analytics.track("user_created", user.id);

  return user;
}
```

**✅ Good (single responsibility):**
```typescript
function createUser(userData: CreateUserDto): Promise<User> {
  validateUserData(userData);
  const hashedPassword = hashPassword(userData.password);
  const user = saveUser({ ...userData, password: hashedPassword });

  // Side effects handled separately
  queueWelcomeEmail(user.id);
  trackUserCreation(user.id);

  return user;
}
```

**Guidelines:**
- One function, one purpose (Single Responsibility Principle)
- Limit function length (< 30 lines ideal)
- Limit parameters (≤ 3-4, use objects for more)
- Avoid side effects in pure functions
- Extract complex logic into helper functions

### DRY (Don't Repeat Yourself)

**❌ Bad:**
```typescript
function getUserByEmail(email: string) {
  const result = await db.query("SELECT * FROM users WHERE email = ?", [email]);
  if (!result) throw new Error("User not found");
  return result;
}

function getUserById(id: number) {
  const result = await db.query("SELECT * FROM users WHERE id = ?", [id]);
  if (!result) throw new Error("User not found");
  return result;
}
```

**✅ Good:**
```typescript
async function findUser<T>(
  field: string,
  value: T
): Promise<User> {
  const result = await db.query(
    `SELECT * FROM users WHERE ${field} = ?`,
    [value]
  );

  if (!result) {
    throw new UserNotFoundError(`User not found by ${field}`);
  }

  return result;
}

const getUserByEmail = (email: string) => findUser("email", email);
const getUserById = (id: number) => findUser("id", id);
```

### Error Handling

**❌ Bad:**
```typescript
try {
  const user = JSON.parse(data);
  updateUser(user);
} catch (e) {
  console.log("Error");
}
```

**✅ Good:**
```typescript
try {
  const user = JSON.parse(data);
  await updateUser(user);
} catch (error) {
  if (error instanceof SyntaxError) {
    logger.error("Invalid JSON format", { data, error });
    throw new ValidationError("Invalid user data format");
  }

  if (error instanceof UserNotFoundError) {
    logger.warn("User not found during update", { userId: user.id });
    throw error;
  }

  // Unexpected error
  logger.error("Failed to update user", { error });
  throw new InternalError("User update failed");
}
```

**Guidelines:**
- Catch specific exceptions, not generic
- Log errors with context
- Don't swallow errors silently
- Use custom error types
- Re-throw or handle, never both
- Clean up resources (finally block, try-with-resources)

## Security Review

### Input Validation

**❌ Vulnerable:**
```typescript
app.post("/api/users", async (req, res) => {
  const user = await db.query(
    `INSERT INTO users (email, role) VALUES ('${req.body.email}', '${req.body.role}')`
  );
  res.json(user);
});
```

**✅ Secure:**
```typescript
app.post("/api/users", async (req, res) => {
  // Validate input
  const schema = z.object({
    email: z.string().email(),
    role: z.enum(["user", "admin"]).default("user")
  });

  const validated = schema.parse(req.body);

  // Use parameterized queries
  const user = await db.query(
    "INSERT INTO users (email, role) VALUES (?, ?)",
    [validated.email, validated.role]
  );

  res.json(user);
});
```

**Check for:**
- SQL/NoSQL injection (use parameterized queries/ORMs)
- XSS attacks (sanitize HTML input, escape output)
- Command injection (never pass user input to shell)
- Path traversal (validate file paths)
- Always validate and sanitize input

### Authentication & Authorization

**❌ Insecure:**
```typescript
app.delete("/api/users/:id", async (req, res) => {
  await db.users.delete(req.params.id);
  res.sendStatus(204);
});
```

**✅ Secure:**
```typescript
app.delete("/api/users/:id",
  authenticateUser,
  async (req, res) => {
    const userId = req.params.id;
    const currentUser = req.user;

    // Authorization check
    if (currentUser.id !== userId && !currentUser.isAdmin) {
      throw new ForbiddenError("Cannot delete other users");
    }

    await db.users.delete(userId);
    res.sendStatus(204);
  }
);
```

**Check for:**
- All protected routes have authentication
- Authorization checks before sensitive operations
- Token validation and expiry
- Password hashing (bcrypt, argon2)
- Secure session management

### Sensitive Data

**❌ Exposed:**
```typescript
// .env committed to git
API_SECRET_KEY=abc123

// Logged sensitive data
logger.info("User login", { email, password });

// Returned in response
res.json({
  user: {
    id: user.id,
    email: user.email,
    passwordHash: user.passwordHash,  // ❌
    apiKey: user.apiKey                // ❌
  }
});
```

**✅ Protected:**
```typescript
// .env in .gitignore, use environment variables
const API_SECRET = process.env.API_SECRET_KEY;

// Log without sensitive data
logger.info("User login attempt", { email });

// Return only safe fields
res.json({
  user: {
    id: user.id,
    email: user.email,
    name: user.name
  }
});
```

**Check for:**
- No secrets in code or version control
- Environment variables for sensitive config
- No passwords/tokens in logs
- Sensitive data not exposed in API responses
- HTTPS for data transmission

## Performance Review

### N+1 Query Problem

**❌ Inefficient:**
```typescript
const posts = await db.posts.findAll();

for (const post of posts) {
  post.author = await db.users.findById(post.authorId);  // N queries!
}

return posts;
```

**✅ Efficient:**
```typescript
const posts = await db.posts.findAll({
  include: [{ model: User, as: "author" }]  // 1 query with JOIN
});

return posts;

// Or with DataLoader (GraphQL)
const posts = await db.posts.findAll();
const authorIds = posts.map(p => p.authorId);
const authors = await userLoader.loadMany(authorIds);

return posts.map(post => ({
  ...post,
  author: authors.find(a => a.id === post.authorId)
}));
```

### Inefficient Algorithms

**❌ O(n²):**
```typescript
function findDuplicates(arr: number[]): number[] {
  const duplicates = [];

  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j] && !duplicates.includes(arr[i])) {
        duplicates.push(arr[i]);
      }
    }
  }

  return duplicates;
}
```

**✅ O(n):**
```typescript
function findDuplicates(arr: number[]): number[] {
  const seen = new Set<number>();
  const duplicates = new Set<number>();

  for (const num of arr) {
    if (seen.has(num)) {
      duplicates.add(num);
    } else {
      seen.add(num);
    }
  }

  return Array.from(duplicates);
}
```

### Memory Leaks

**❌ Leak:**
```typescript
class UserService {
  private cache = new Map();

  getUser(id: number) {
    if (!this.cache.has(id)) {
      this.cache.set(id, fetchUser(id));  // Cache grows indefinitely!
    }
    return this.cache.get(id);
  }
}
```

**✅ Fixed:**
```typescript
import LRU from "lru-cache";

class UserService {
  private cache = new LRU({ max: 500, ttl: 1000 * 60 * 5 });

  getUser(id: number) {
    if (!this.cache.has(id)) {
      this.cache.set(id, fetchUser(id));
    }
    return this.cache.get(id);
  }
}
```

**Check for:**
- N+1 queries (use eager loading, DataLoader)
- Inefficient algorithms (consider complexity)
- Unnecessary database queries (caching)
- Large data loaded into memory
- Event listeners not cleaned up
- Timers/intervals not cleared

## Testing Review

### Test Coverage

**❌ Incomplete:**
```typescript
describe("calculateDiscount", () => {
  it("calculates 10% discount", () => {
    expect(calculateDiscount(100, 10)).toBe(90);
  });
});
```

**✅ Comprehensive:**
```typescript
describe("calculateDiscount", () => {
  it("calculates correct discount for valid inputs", () => {
    expect(calculateDiscount(100, 10)).toBe(90);
    expect(calculateDiscount(50, 20)).toBe(40);
  });

  it("returns original price for 0% discount", () => {
    expect(calculateDiscount(100, 0)).toBe(100);
  });

  it("throws error for negative price", () => {
    expect(() => calculateDiscount(-100, 10)).toThrow("Invalid price");
  });

  it("throws error for discount > 100%", () => {
    expect(() => calculateDiscount(100, 150)).toThrow("Invalid discount");
  });

  it("handles floating point precision", () => {
    expect(calculateDiscount(99.99, 10)).toBeCloseTo(89.99);
  });
});
```

### Test Quality

**❌ Fragile:**
```typescript
it("creates user", async () => {
  const user = await createUser({ email: "test@example.com" });

  expect(user.id).toBe(123);  // Hardcoded ID
  expect(user.createdAt).toBe("2024-01-04");  // Hardcoded date
});
```

**✅ Robust:**
```typescript
it("creates user with correct attributes", async () => {
  const userData = { email: "test@example.com", name: "Test User" };
  const user = await createUser(userData);

  expect(user.id).toBeDefined();
  expect(user.email).toBe(userData.email);
  expect(user.name).toBe(userData.name);
  expect(user.createdAt).toBeInstanceOf(Date);
  expect(user.createdAt.getTime()).toBeCloseTo(Date.now(), -2);
});
```

**Check for:**
- Edge cases covered (empty, null, boundary values)
- Error cases tested
- Integration tests for critical paths
- Tests are isolated (no shared state)
- Tests are deterministic (no flakiness)
- Mock external dependencies
- Meaningful assertions

## Language-Specific Patterns

### TypeScript

**Check for:**
- Type safety (avoid `any`, use proper types)
- Interface over type for objects
- Enum for fixed sets of values
- Optional chaining (`?.`) and nullish coalescing (`??`)
- Strict mode enabled

**❌ Bad:**
```typescript
function getUser(id: any): any {
  const user = users.find(u => u.id === id);
  return user.name;  // Unsafe!
}
```

**✅ Good:**
```typescript
function getUser(id: number): User | undefined {
  return users.find(u => u.id === id);
}

function getUserName(id: number): string {
  const user = getUser(id);
  return user?.name ?? "Unknown";
}
```

### Python

**Check for:**
- Type hints (PEP 484)
- List comprehensions over loops
- Context managers for resources
- `pathlib` over string paths
- f-strings over format()

**❌ Bad:**
```python
def process_data(data):
    result = []
    for item in data:
        if item > 0:
            result.append(item * 2)
    return result
```

**✅ Good:**
```python
def process_data(data: list[int]) -> list[int]:
    return [item * 2 for item in data if item > 0]
```

### React/Vue

**Check for:**
- Hooks rules (React)
- Proper key props in lists
- Effect cleanup (unmount)
- Memoization for expensive computations
- State immutability

**❌ Bad:**
```typescript
function UserList({ users }) {
  const [filteredUsers, setFilteredUsers] = useState([]);

  // Runs on every render!
  const filtered = users.filter(u => u.isActive);
  setFilteredUsers(filtered);

  return filtered.map(user => <User user={user} />);  // No key!
}
```

**✅ Good:**
```typescript
function UserList({ users }: { users: User[] }) {
  const filteredUsers = useMemo(
    () => users.filter(u => u.isActive),
    [users]
  );

  return (
    <>
      {filteredUsers.map(user => (
        <User key={user.id} user={user} />
      ))}
    </>
  );
}
```

## Review Feedback Format

### Prioritization

**Critical (Must Fix):**
- Security vulnerabilities
- Data corruption risks
- Breaking changes
- Crashes/unhandled errors

**Warning (Should Fix):**
- Performance issues
- Code quality problems
- Missing error handling
- Poor test coverage

**Suggestion (Consider):**
- Style improvements
- Refactoring opportunities
- Alternative approaches
- Documentation additions

### Feedback Template

```markdown
## Critical Issues

### 🔴 SQL Injection Vulnerability (security)
**File:** `src/api/users.ts:45`

Using string concatenation for SQL queries allows injection attacks.

**Current:**
```typescript
db.query(`SELECT * FROM users WHERE id = '${userId}'`)
```

**Recommended:**
```typescript
db.query('SELECT * FROM users WHERE id = ?', [userId])
```

---

## Warnings

### ⚠️ N+1 Query Problem (performance)
**File:** `src/services/posts.ts:23-28`

Loading authors in a loop causes N+1 queries.

**Suggestion:** Use eager loading or DataLoader pattern.

---

## Suggestions

### 💡 Consider Extracting Function (quality)
**File:** `src/utils/validation.ts:15-45`

The validation logic is 30 lines and does multiple things. Consider extracting to separate functions:
- `validateEmail()`
- `validatePassword()`
- `validateAge()`
```

### Example Review Comments

**❌ Unhelpful:**
```
- This is bad
- Don't do this
- Fix this
```

**✅ Helpful:**
```
Consider extracting this validation logic into a separate function.
This would improve readability and make it easier to test in isolation.

Example:
[code snippet]

This follows the Single Responsibility Principle and makes the code more maintainable.
```

## Common Anti-Patterns

### Magic Numbers

**❌ Bad:**
```typescript
if (user.age >= 18 && user.accountAge > 30) {
  // What do these numbers mean?
}
```

**✅ Good:**
```typescript
const MINIMUM_AGE = 18;
const TRIAL_PERIOD_DAYS = 30;

if (user.age >= MINIMUM_AGE && user.accountAge > TRIAL_PERIOD_DAYS) {
  // Clear intent
}
```

### God Objects/Functions

**❌ Bad:**
```typescript
class UserManager {
  createUser() { }
  deleteUser() { }
  sendEmail() { }
  processPayment() { }
  generateReport() { }
  logAnalytics() { }
  // ... 50 more methods
}
```

**✅ Good:**
```typescript
class UserService {
  createUser() { }
  deleteUser() { }
}

class EmailService {
  sendEmail() { }
}

class PaymentService {
  processPayment() { }
}
```

### Premature Optimization

**❌ Premature:**
```typescript
// Optimizing before profiling
const cache = new Map();
function fibonacci(n: number): number {
  if (cache.has(n)) return cache.get(n);
  // Complex memoization for unused code
}
```

**✅ Better:**
```typescript
// Start simple, optimize if needed
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

## Quick Review Checklist

**Architecture & Design:**
- [ ] Changes follow existing patterns
- [ ] SOLID principles respected
- [ ] Appropriate design patterns used
- [ ] No god objects/functions

**Security:**
- [ ] Input validation present
- [ ] No SQL/NoSQL injection
- [ ] Authentication/authorization correct
- [ ] No sensitive data exposed
- [ ] No secrets in code

**Correctness:**
- [ ] Logic is correct
- [ ] Edge cases handled
- [ ] Error handling present
- [ ] No race conditions

**Performance:**
- [ ] No N+1 queries
- [ ] Efficient algorithms
- [ ] No memory leaks
- [ ] Proper caching strategy

**Code Quality:**
- [ ] Clear, descriptive names
- [ ] Functions are focused (SRP)
- [ ] No code duplication (DRY)
- [ ] Commented where necessary
- [ ] Consistent style

**Testing:**
- [ ] Tests present and passing
- [ ] Edge cases covered
- [ ] Good test coverage
- [ ] Tests are maintainable

**Documentation:**
- [ ] Public APIs documented
- [ ] Complex logic explained
- [ ] README updated if needed

---
name: performance-review
description: Review code for performance issues and identify optimization opportunities across frontend, backend, database, and mobile applications. Use when application performance degrades, response times exceed targets, or before major releases. Apply for analyzing bundle sizes, API latency, database query performance, memory leaks, and mobile app responsiveness. Essential for maintaining fast, efficient applications that meet performance SLAs and provide excellent user experience.
---

# Performance Review

Systematically identify and resolve performance bottlenecks across the application stack.

## Performance Targets

### Frontend (Web)
- **LCP (Largest Contentful Paint)**: < 2.5s
- **FID (First Input Delay)**: < 100ms
- **CLS (Cumulative Layout Shift)**: < 0.1
- **TTI (Time to Interactive)**: < 3.5s
- **Bundle size**: < 1MB initial load
- **API calls**: < 200ms P95

### Backend (API)
- **Response time**: < 500ms P95, < 1s P99
- **Throughput**: > 1000 req/s per instance
- **Error rate**: < 0.1%
- **Database queries**: < 100ms P95

### Mobile (Android/iOS)
- **App startup**: < 2s cold start
- **Screen transitions**: < 100ms
- **Memory usage**: < 200MB average
- **Battery drain**: < 5% per hour active use
- **API calls**: < 300ms on 4G

## Frontend Performance

### Check route lazy loading

```typescript
// ❌ BAD: Eager loading all routes
import Dashboard from './pages/Dashboard.vue'
import Settings from './pages/Settings.vue'

const routes = [
  { path: '/dashboard', component: Dashboard },
  { path: '/settings', component: Settings }
]

// ✅ GOOD: Lazy load routes
const routes = [
  {
    path: '/dashboard',
    component: () => import('./pages/Dashboard.vue')
  },
  {
    path: '/settings',
    component: () => import('./pages/Settings.vue')
  }
]
```

### Analyze bundle size

```bash
# Build with analysis
npm run build -- --report

# Check bundle sizes
npm run build
ls -lh dist/assets/*.js

# Look for:
# - Vendor bundle > 500KB (split large dependencies)
# - Duplicate code in multiple chunks
# - Unused dependencies in bundle
```

**Optimization strategies**:
- Code splitting by route
- Dynamic imports for heavy components
- Tree-shaking unused code
- Replace heavy libraries (moment → day.js, lodash → lodash-es)

### Optimize images

```vue
<!-- ❌ BAD: Large unoptimized image -->
<img src="/images/hero.jpg" alt="Hero">

<!-- ✅ GOOD: Responsive + lazy loading -->
<img
  srcset="
    /images/hero-400.webp 400w,
    /images/hero-800.webp 800w,
    /images/hero-1200.webp 1200w
  "
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  src="/images/hero-800.webp"
  alt="Hero"
  loading="lazy"
  decoding="async"
>
```

**Checklist**:
- [ ] Images compressed (WebP/AVIF format)
- [ ] Responsive images (srcset/sizes)
- [ ] Lazy loading for below-fold images
- [ ] Appropriate image dimensions (not serving 4K for mobile)

### Prevent unnecessary re-renders

**Vue 3**:
```vue
<script setup>
import { computed } from 'vue'

// ❌ BAD: Recomputes on every render
const expensiveValue = () => {
  return data.map(item => heavyComputation(item))
}

// ✅ GOOD: Computed caching
const expensiveValue = computed(() => {
  return data.value.map(item => heavyComputation(item))
})

// ✅ Use v-memo for lists
<template>
  <div v-for="item in items" :key="item.id" v-memo="[item.id, item.updated]">
    <!-- Only re-render if id or updated changes -->
  </div>
</template>
</script>
```

**React**:
```typescript
// ❌ BAD: Creates new object on every render
function MyComponent({ data }) {
  const config = { theme: 'dark', size: 'large' }
  return <Child config={config} />  // Child re-renders every time
}

// ✅ GOOD: Memoized object
const config = useMemo(() => ({
  theme: 'dark',
  size: 'large'
}), [])

// ✅ Memoize expensive computations
const filteredData = useMemo(() => {
  return data.filter(item => item.active)
}, [data])

// ✅ Memo for pure components
const MemoizedChild = React.memo(Child)
```

### Configure code splitting

```javascript
// vite.config.ts / rollup config
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['vue', 'vue-router', 'pinia'],
          'ui': ['element-plus'],
          'utils': ['lodash-es', 'dayjs']
        }
      }
    }
  }
}
```

## Backend Performance

### Measure API response time

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)

    # Log slow requests
    if process_time > 0.5:
        logger.warning(f"Slow request: {request.url} took {process_time:.2f}s")

    return response
```

### Eliminate N+1 queries

```python
# ❌ BAD: N+1 queries
users = await db.users.find({}).to_list(100)
for user in users:
    user.posts = await db.posts.find({"user_id": user.id}).to_list(10)

# ✅ GOOD: Single aggregation query
users = await db.users.aggregate([
    {
        "$lookup": {
            "from": "posts",
            "localField": "_id",
            "foreignField": "user_id",
            "as": "posts"
        }
    },
    {"$limit": 100}
]).to_list()
```

### Use database indexes

```python
# Check query execution plan
explain = await db.users.find({"status": "active"}).explain()

# Look for:
# - "stage": "IXSCAN" (✅ using index)
# - "stage": "COLLSCAN" (❌ full collection scan)

# Create missing indexes
await db.users.create_index([("status", 1), ("created_at", -1)])
```

### Implement caching (Redis)

```python
from redis import asyncio as aioredis

redis = aioredis.from_url("redis://localhost")

async def get_user_profile(user_id: int):
    # Try cache first
    cached = await redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # Cache miss: fetch from DB
    user = await db.users.find_one({"_id": user_id})

    # Store in cache (5 min TTL)
    await redis.setex(
        f"user:{user_id}",
        300,
        json.dumps(user, default=str)
    )

    return user
```

**Caching strategy**:
- Cache frequently accessed, rarely changing data
- Use appropriate TTL (time-to-live)
- Invalidate cache on updates
- Consider cache warming for critical data

### Use async operations

```python
# ❌ BAD: Sequential blocking calls
def get_dashboard():
    user = fetch_user()          # 50ms
    posts = fetch_posts()        # 100ms
    stats = fetch_stats()        # 80ms
    return {"user": user, "posts": posts, "stats": stats}
# Total: 230ms

# ✅ GOOD: Parallel async execution
async def get_dashboard():
    user, posts, stats = await asyncio.gather(
        fetch_user(),
        fetch_posts(),
        fetch_stats()
    )
    return {"user": user, "posts": posts, "stats": stats}
# Total: 100ms (limited by slowest operation)
```

## Database Performance

### Use appropriate indexes

```javascript
// Check existing indexes
db.users.getIndexes()

// Analyze query performance
db.users.find({ status: "active" }).explain("executionStats")

// Key metrics:
// - totalDocsExamined: Should be close to nReturned
// - executionTimeMillis: Should be < 100ms
// - stage: Should be "IXSCAN" not "COLLSCAN"

// Create compound index for common query pattern
db.users.createIndex({ status: 1, created_at: -1 })
```

### Avoid full table scans

```javascript
// ❌ BAD: No filters, scans entire collection
db.users.find({})

// ✅ GOOD: Filtered query uses index
db.users.find({ status: "active" })

// ✅ GOOD: Limit results
db.users.find({ status: "active" }).limit(20)
```

### Use projections

```javascript
// ❌ BAD: Fetches all fields including large ones
db.users.find({ status: "active" })

// ✅ GOOD: Only fetch needed fields
db.users.find(
  { status: "active" },
  { name: 1, email: 1, _id: 0 }
)

// Reduces network transfer and memory usage
```

### Configure connection pooling

```python
# MongoDB connection with pooling
client = motor.motor_asyncio.AsyncIOMotorClient(
    "mongodb://localhost:27017",
    maxPoolSize=50,
    minPoolSize=10
)
```

## Mobile Performance

### Reduce app startup time

```kotlin
// Use lazy initialization
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        // ❌ BAD: Heavy initialization blocks startup
        initializeDatabase()
        initializeAnalytics()
        loadUserPreferences()

        // ✅ GOOD: Defer non-critical initialization
        lifecycleScope.launch {
            initializeAnalytics()  // Run in background
        }
    }
}
```

### Optimize memory usage

```kotlin
// Use appropriate data structures
// ❌ BAD: Loads entire list into memory
val allUsers = database.getAllUsers()  // 100,000 users

// ✅ GOOD: Paginated loading
val pagedUsers = Pager(
    config = PagingConfig(pageSize = 20),
    pagingSourceFactory = { UserPagingSource(database) }
).flow
```

### Minimize battery consumption

```kotlin
// ❌ BAD: Continuous location updates
locationManager.requestLocationUpdates(
    LocationManager.GPS_PROVIDER,
    1000,  // Every second
    0f,
    locationListener
)

// ✅ GOOD: Appropriate update interval
locationManager.requestLocationUpdates(
    LocationManager.GPS_PROVIDER,
    300000,  // Every 5 minutes
    100f,    // Or 100 meters movement
    locationListener
)
```

### Batch network requests

```kotlin
// ❌ BAD: Individual requests in loop
for (userId in userIds) {
    fetchUser(userId)  // N network requests
}

// ✅ GOOD: Batch request
fetchUsers(userIds)  // Single network request
```

## Performance Review Checklist

### Frontend
- [ ] Route lazy loading implemented
- [ ] Bundle size < 1MB initial load
- [ ] Images optimized/lazy loaded
- [ ] No unnecessary re-renders
- [ ] Code splitting configured
- [ ] LCP < 2.5s, FID < 100ms

### Backend
- [ ] API P95 response time < 500ms
- [ ] No N+1 query problems
- [ ] Database indexes properly configured
- [ ] Caching implemented for hot paths
- [ ] Async operations used appropriately
- [ ] Connection pooling configured

### Database
- [ ] Queries use indexes (IXSCAN not COLLSCAN)
- [ ] No full table scans on large collections
- [ ] Compound indexes for multi-field queries
- [ ] Projections used to limit fields
- [ ] Query execution time < 100ms P95

### Mobile
- [ ] App cold start < 2s
- [ ] Memory usage < 200MB average
- [ ] Battery drain < 5% per hour active
- [ ] Network requests batched
- [ ] Images cached appropriately

## Performance Testing Tools

- **Frontend**: Lighthouse, WebPageTest, Chrome DevTools
- **Backend**: Apache Bench, k6, Artillery
- **Database**: MongoDB Atlas profiler, explain() plans
- **Mobile**: Android Profiler, Xcode Instruments
- **APM**: New Relic, Datadog, Sentry Performance

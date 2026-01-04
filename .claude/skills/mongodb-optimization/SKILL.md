---
name: mongodb-optimization
description: MongoDB query optimization and indexing strategies for improved performance. Use when experiencing slow queries, optimizing database operations, designing indexes for collections, analyzing query execution plans, or resolving N+1 query problems. Apply when database performance monitoring shows high query times, when adding new query patterns, or when collection size grows significantly. Essential for maintaining fast, efficient MongoDB operations at scale.
---

# MongoDB Optimization

Optimize MongoDB queries and indexes for maximum performance and scalability.

## Index Strategy

### Single Field Index

Use for queries filtering/sorting on one field.

```javascript
// Create single field index
db.users.createIndex({ "email": 1 }, { unique: true })

// Use case: Fast lookup by email
db.users.find({ email: "user@example.com" })

// Ascending (1) vs Descending (-1)
db.posts.createIndex({ "created_at": -1 })  // Recent posts first
```

### Compound Index

Use for queries involving multiple fields. **Order matters** - place most selective fields first.

```javascript
// Compound index: status + created_at
db.users.createIndex({ "status": 1, "created_at": -1 })

// ✅ Uses index efficiently
db.users.find({ status: "active" }).sort({ created_at: -1 })

// ✅ Uses index (prefix match)
db.users.find({ status: "active" })

// ❌ Cannot use index (missing prefix)
db.users.find({}).sort({ created_at: -1 })
```

**Index prefix rule**: Compound index `{a: 1, b: 1, c: 1}` supports queries on:
- `{a}`
- `{a, b}`
- `{a, b, c}`

But NOT: `{b}`, `{c}`, or `{b, c}`

### Text Search Index

Enable full-text search on string fields.

```javascript
// Create text index on multiple fields
db.posts.createIndex({
  "title": "text",
  "content": "text",
  "tags": "text"
}, {
  weights: {
    title: 10,     // Title matches rank higher
    content: 5,
    tags: 3
  },
  name: "post_text_search"
})

// Search with text index
db.posts.find({
  $text: {
    $search: "mongodb optimization",
    $caseSensitive: false
  }
})

// Sort by relevance score
db.posts.find(
  { $text: { $search: "mongodb" }},
  { score: { $meta: "textScore" }}
).sort({ score: { $meta: "textScore" }})
```

### Partial Index

Index only documents matching a filter (saves space and improves performance).

```javascript
// Index only active users
db.users.createIndex(
  { email: 1 },
  {
    partialFilterExpression: { status: "active" },
    name: "active_users_email"
  }
)

// ✅ Uses partial index
db.users.find({ email: "user@example.com", status: "active" })

// ❌ Cannot use partial index (missing status filter)
db.users.find({ email: "user@example.com" })
```

### Sparse Index

Index only documents with the indexed field (skips null/missing values).

```javascript
// Index users with phone numbers (many users don't have phone)
db.users.createIndex({ phone: 1 }, { sparse: true })

// Efficient for queries specifically looking for phone numbers
db.users.find({ phone: { $exists: true }})
```

### Unique Index

Enforce uniqueness constraint.

```javascript
// Unique email constraint
db.users.createIndex({ email: 1 }, { unique: true })

// Compound unique index
db.sessions.createIndex(
  { user_id: 1, device_id: 1 },
  { unique: true }
)
```

## Query Optimization

### Use Projections

Only retrieve fields you need.

```javascript
// ❌ BAD: Retrieve all fields
db.users.find({ status: "active" })

// ✅ GOOD: Retrieve only needed fields
db.users.find(
  { status: "active" },
  { name: 1, email: 1, _id: 0 }  // Include name/email, exclude _id
)

// Reduces network transfer and memory usage
```

### Limit Result Set

```javascript
// ✅ GOOD: Use limit for pagination
db.users.find({ status: "active" })
  .sort({ created_at: -1 })
  .limit(20)
  .skip(40)  // Page 3 (skip = page * limit)

// For large skip values, use range queries instead
db.users.find({
  status: "active",
  created_at: { $lt: lastSeenDate }
})
  .sort({ created_at: -1 })
  .limit(20)
```

### Avoid N+1 Queries

**Problem**: Fetching related data in a loop.

```javascript
// ❌ BAD: N+1 queries (1 for posts + N for users)
const posts = await db.posts.find({}).toArray()
for (const post of posts) {
  post.author = await db.users.findOne({ _id: post.user_id })
}

// ✅ GOOD: Use aggregation or fetch in bulk
const posts = await db.posts.find({}).toArray()
const userIds = posts.map(p => p.user_id)
const users = await db.users.find({ _id: { $in: userIds }}).toArray()
const userMap = Object.fromEntries(users.map(u => [u._id, u]))
posts.forEach(p => p.author = userMap[p.user_id])
```

### Use Covered Queries

Query entirely satisfied by index (no document access needed).

```javascript
// Create index
db.users.createIndex({ status: 1, email: 1, name: 1 })

// ✅ Covered query (all fields in index)
db.users.find(
  { status: "active" },
  { email: 1, name: 1, _id: 0 }  // Must exclude _id unless indexed
)

// Verify with explain: "totalDocsExamined": 0
```

## Aggregation Pipeline

Use for complex transformations and joins.

### Basic Pipeline

```javascript
db.users.aggregate([
  // Stage 1: Filter
  { $match: { status: "active" } },

  // Stage 2: Join with posts
  { $lookup: {
      from: "posts",
      localField: "_id",
      foreignField: "user_id",
      as: "posts"
  }},

  // Stage 3: Add computed field
  { $addFields: {
      post_count: { $size: "$posts" }
  }},

  // Stage 4: Project only needed fields
  { $project: {
      email: 1,
      name: 1,
      post_count: 1,
      _id: 0
  }},

  // Stage 5: Sort and limit
  { $sort: { post_count: -1 }},
  { $limit: 10 }
])
```

### Performance Tips

1. **Put $match early**: Reduce documents processed by later stages
   ```javascript
   // ✅ GOOD: Match first
   [
     { $match: { status: "active" }},
     { $lookup: {...}},
     { $sort: {...}}
   ]

   // ❌ BAD: Match after expensive operations
   [
     { $lookup: {...}},
     { $sort: {...}},
     { $match: { status: "active" }}
   ]
   ```

2. **Use indexes**: $match and $sort can use indexes if early in pipeline

3. **Limit memory usage**: Use `allowDiskUse: true` for large aggregations
   ```javascript
   db.collection.aggregate(pipeline, { allowDiskUse: true })
   ```

### Complex Example: User Analytics

```javascript
db.posts.aggregate([
  // Stage 1: Filter last 30 days
  {
    $match: {
      created_at: {
        $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
      }
    }
  },

  // Stage 2: Group by user and count
  {
    $group: {
      _id: "$user_id",
      post_count: { $sum: 1 },
      total_views: { $sum: "$views" },
      avg_views: { $avg: "$views" }
    }
  },

  // Stage 3: Join user info
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user"
    }
  },

  // Stage 4: Unwind user array
  { $unwind: "$user" },

  // Stage 5: Project final structure
  {
    $project: {
      user_email: "$user.email",
      user_name: "$user.name",
      post_count: 1,
      total_views: 1,
      avg_views: { $round: ["$avg_views", 2] }
    }
  },

  // Stage 6: Sort by activity
  { $sort: { post_count: -1 }},

  // Stage 7: Top 50 users
  { $limit: 50 }
])
```

## Query Profiling

### Enable Profiling

```javascript
// Level 0: Off
// Level 1: Slow queries (> slowms threshold)
// Level 2: All queries

db.setProfilingLevel(2, { slowms: 100 })

// View profiled queries
db.system.profile.find().sort({ ts: -1 }).limit(10)

// Find slow queries
db.system.profile.find({
  millis: { $gt: 100 }
}).sort({ millis: -1 })
```

### Use Explain

Analyze query execution plan.

```javascript
// Get execution stats
db.users.find({ status: "active" }).explain("executionStats")

/**
 * Key metrics to check:
 * - executionTimeMillis: Total query time
 * - totalDocsExamined: Documents scanned
 * - totalKeysExamined: Index entries scanned
 * - stage: "IXSCAN" (index) vs "COLLSCAN" (full scan)
 *
 * Goal: totalDocsExamined ≈ nReturned (minimal extra scanning)
 */

// Example output analysis:
// ✅ GOOD: Using index
{
  "stage": "IXSCAN",
  "totalDocsExamined": 10,
  "nReturned": 10,
  "executionTimeMillis": 2
}

// ❌ BAD: Full collection scan
{
  "stage": "COLLSCAN",
  "totalDocsExamined": 100000,
  "nReturned": 10,
  "executionTimeMillis": 450
}
```

## Index Management

### List Indexes

```javascript
db.users.getIndexes()
```

### Drop Index

```javascript
// By name
db.users.dropIndex("email_1")

// By specification
db.users.dropIndex({ email: 1 })

// Drop all except _id
db.users.dropIndexes()
```

### Monitor Index Usage

```javascript
// Check index statistics
db.users.aggregate([{ $indexStats: {}}])

// Find unused indexes
db.users.aggregate([
  { $indexStats: {}},
  {
    $match: {
      "accesses.ops": { $lt: 100 }  // Used less than 100 times
    }
  }
])
```

## Best Practices

1. **Index Selectively**: Each index slows writes and uses disk space
   - Create indexes for frequent queries
   - Remove unused indexes

2. **Compound Index Order**: Most selective fields first
   - `{status: 1, created_at: -1}` if status has few values
   - Use `{created_at: -1, status: 1}` if most docs have same status

3. **Use Projections**: Only fetch needed fields

4. **Avoid Large Skip**: Use range queries instead
   - Replace `skip(10000)` with `{_id: {$gt: lastId}}`

5. **Monitor Performance**:
   - Enable slow query logging
   - Regular `explain()` analysis
   - Track index usage with `$indexStats`

6. **Connection Pooling**: Reuse connections
   ```javascript
   const client = new MongoClient(uri, {
     maxPoolSize: 50,
     minPoolSize: 10
   })
   ```

## Optimization Checklist

- [ ] Indexes created for frequent query patterns
- [ ] Compound index field order optimized
- [ ] Unused indexes removed
- [ ] Queries use projections to limit fields
- [ ] N+1 queries eliminated (use aggregation or bulk fetch)
- [ ] Slow query profiling enabled
- [ ] Regular `explain()` analysis performed
- [ ] Pagination uses limit + range queries (not large skip)
- [ ] Connection pooling configured
- [ ] Aggregation pipelines have $match early

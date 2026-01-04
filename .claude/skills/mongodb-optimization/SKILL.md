---
name: mongodb-optimization
description: MongoDB query optimization, indexing strategies.
allowed-tools: Read, Bash
---

# MongoDB Optimization

## Index Strategy
```javascript
// Single field index
db.users.createIndex({ "email": 1 }, { unique: true })

// Compound index
db.users.createIndex({ "status": 1, "created_at": -1 })

// Text search index
db.posts.createIndex({ "title": "text", "content": "text" })
```

## Query Optimization
```javascript
// Good: Uses index
db.users.find({ status: "active", created_at: { $gte: ISODate("2024-01-01") } })
  .sort({ created_at: -1 })
  .limit(20)

// Bad: No index, full scan
db.users.find({})
```

## Aggregation Pipeline
```javascript
db.users.aggregate([
  { $match: { status: "active" } },
  { $lookup: {
      from: "posts",
      localField: "_id",
      foreignField: "user_id",
      as: "posts"
  }},
  { $project: { email: 1, posts: 1 } }
])
```

## Profiling
```javascript
// Enable profiling
db.setProfilingLevel(2, { slowms: 100 })

// View slow queries
db.system.profile.find().sort({ts: -1}).limit(10)
```

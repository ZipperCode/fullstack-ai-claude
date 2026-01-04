---
name: performance-review
description: Review code for performance issues and optimization opportunities.
allowed-tools: Read, Grep, Bash
---

# Performance Review Skill

## Performance Checklist

### Frontend
- [ ] Route lazy loading implemented
- [ ] Bundle size optimized (< 1MB)
- [ ] Images optimized/lazy loaded
- [ ] No unnecessary re-renders
- [ ] Code splitting configured

### Backend
- [ ] API response time < 500ms (P95)
- [ ] No N+1 query problems
- [ ] Proper database indexes used
- [ ] Caching implemented (Redis)
- [ ] Async operations where appropriate

### Database
- [ ] Queries use indexes
- [ ] No full table scans
- [ ] Compound indexes for multi-field queries
- [ ] Connection pooling configured

### Mobile
- [ ] App startup < 2s
- [ ] Memory usage optimized
- [ ] Battery consumption minimal
- [ ] Network requests batched

## Performance Targets
- LCP (Frontend): < 2.5s
- API P95: < 500ms
- DB queries: < 100ms
- Mobile startup: < 2s

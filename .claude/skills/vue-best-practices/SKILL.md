---
name: vue-best-practices
description: Vue 3 Composition API best practices and patterns.
allowed-tools: Read, Edit
---

# Vue 3 Best Practices

## Composition API
```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>
```

## Composables
```typescript
// composables/useUser.ts
export function useUser() {
  const user = ref<User | null>(null)
  const loading = ref(false)
  
  async function fetch() {
    loading.value = true
    user.value = await api.fetchUser()
    loading.value = false
  }
  
  return { user, loading, fetch }
}
```

## Pinia Store
```typescript
export const useAuthStore = defineStore('auth', () => {
  const token = ref<string | null>(null)
  const isAuthenticated = computed(() => !!token.value)
  
  function login(credentials) {
    // ...
  }
  
  return { token, isAuthenticated, login }
})
```

## Best Practices
- Use `<script setup>` syntax
- Prefer Composition API over Options API
- Use TypeScript for type safety
- Extract reusable logic to composables
- Use Pinia for state management

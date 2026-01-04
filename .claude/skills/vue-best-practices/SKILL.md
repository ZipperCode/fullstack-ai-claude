---
name: vue-best-practices
description: Vue 3 Composition API best practices, Pinia state management, and modern Vue development patterns. Use when developing Vue 3 applications, creating reusable composables, implementing state management, or reviewing Vue code for best practices. Apply for component design, reactivity patterns, performance optimization, and TypeScript integration. Essential for building maintainable, scalable Vue 3 applications following modern standards.
---

# Vue 3 Best Practices

Build modern, maintainable Vue 3 applications using Composition API and best practices.

## Composition API Fundamentals

### Script Setup Syntax

```vue
<script setup lang="ts">
// ✅ GOOD: Modern <script setup> syntax (recommended)
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
}

onMounted(() => {
  console.log('Component mounted')
})
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="increment">Increment</button>
  </div>
</template>
```

Benefits:
- Less boilerplate
- Better TypeScript inference
- Automatic component registration
- Cleaner code organization

### Props and Emits with TypeScript

```vue
<script setup lang="ts">
// Define props with types
interface Props {
  user: {
    id: number
    name: string
    email: string
  }
  editable?: boolean
  maxLength?: number
}

const props = withDefaults(defineProps<Props>(), {
  editable: false,
  maxLength: 100
})

// Define emits with types
interface Emits {
  (e: 'update', user: Props['user']): void
  (e: 'delete', userId: number): void
  (e: 'cancel'): void
}

const emit = defineEmits<Emits>()

// Use props and emit
function handleUpdate() {
  emit('update', props.user)
}

function handleDelete() {
  emit('delete', props.user.id)
}
</script>
```

### Reactive State Management

```vue
<script setup lang="ts">
import { ref, reactive, computed, watch } from 'vue'

// ✅ Use ref for primitives
const count = ref(0)
const message = ref('Hello')

// ✅ Use reactive for objects (alternative to multiple refs)
const state = reactive({
  user: null as User | null,
  loading: false,
  error: null as string | null
})

// ✅ Computed properties for derived state
const userDisplayName = computed(() => {
  return state.user ? state.user.name : 'Guest'
})

// ✅ Watch for side effects
watch(() => state.user, (newUser, oldUser) => {
  console.log('User changed:', oldUser, '->', newUser)
})

// ❌ BAD: Don't destructure reactive
const { user } = state  // Loses reactivity!

// ✅ GOOD: Use toRefs for destructuring
import { toRefs } from 'vue'
const { user, loading } = toRefs(state)
</script>
```

## Composables (Reusable Logic)

### Creating Composables

```typescript
// composables/useUser.ts
import { ref, computed } from 'vue'
import type { Ref } from 'vue'

export interface User {
  id: number
  name: string
  email: string
}

export function useUser(userId: Ref<number> | number) {
  // Make userId reactive if it isn't
  const id = ref(userId)

  const user = ref<User | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  const isLoaded = computed(() => user.value !== null)

  async function fetchUser() {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(`/api/users/${id.value}`)
      if (!response.ok) throw new Error('Failed to fetch user')

      user.value = await response.json()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
    } finally {
      loading.value = false
    }
  }

  async function updateUser(updates: Partial<User>) {
    if (!user.value) return

    loading.value = true
    try {
      const response = await fetch(`/api/users/${id.value}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates)
      })

      if (!response.ok) throw new Error('Failed to update user')

      user.value = await response.json()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
      throw e
    } finally {
      loading.value = false
    }
  }

  // Auto-fetch on mount
  onMounted(() => {
    fetchUser()
  })

  // Return reactive state and methods
  return {
    user: readonly(user),  // Prevent external mutations
    loading: readonly(loading),
    error: readonly(error),
    isLoaded,
    fetchUser,
    updateUser
  }
}
```

### Using Composables

```vue
<script setup lang="ts">
import { useUser } from '@/composables/useUser'

const userId = ref(123)
const { user, loading, error, updateUser } = useUser(userId)

async function handleNameChange(newName: string) {
  try {
    await updateUser({ name: newName })
  } catch (e) {
    alert('Failed to update name')
  }
}
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error }}</div>
  <div v-else-if="user">
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
    <button @click="handleNameChange('New Name')">Update Name</button>
  </div>
</template>
```

### Common Composable Patterns

**Async Data Fetching**:
```typescript
// composables/useFetch.ts
export function useFetch<T>(url: Ref<string> | string) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  const execute = async () => {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(unref(url))
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  // Watch url changes
  watch(() => unref(url), execute, { immediate: true })

  return { data, loading, error, execute }
}
```

**Local Storage**:
```typescript
// composables/useLocalStorage.ts
export function useLocalStorage<T>(key: string, defaultValue: T) {
  const data = ref<T>(defaultValue)

  // Load from localStorage on init
  const stored = localStorage.getItem(key)
  if (stored) {
    try {
      data.value = JSON.parse(stored)
    } catch (e) {
      console.error('Failed to parse localStorage:', e)
    }
  }

  // Save to localStorage on change
  watch(data, (newValue) => {
    localStorage.setItem(key, JSON.stringify(newValue))
  }, { deep: true })

  return data
}
```

## Pinia State Management

### Store Definition

```typescript
// stores/auth.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  // State (use ref/reactive)
  const token = ref<string | null>(null)
  const user = ref<User | null>(null)

  // Getters (use computed)
  const isAuthenticated = computed(() => !!token.value)
  const userDisplayName = computed(() => user.value?.name ?? 'Guest')

  // Actions (functions)
  async function login(email: string, password: string) {
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
      })

      if (!response.ok) throw new Error('Login failed')

      const data = await response.json()
      token.value = data.access_token
      user.value = data.user

      // Persist token
      localStorage.setItem('token', data.access_token)
    } catch (error) {
      console.error('Login error:', error)
      throw error
    }
  }

  function logout() {
    token.value = null
    user.value = null
    localStorage.removeItem('token')
  }

  async function fetchCurrentUser() {
    if (!token.value) return

    try {
      const response = await fetch('/api/users/me', {
        headers: { Authorization: `Bearer ${token.value}` }
      })

      if (!response.ok) throw new Error('Failed to fetch user')

      user.value = await response.json()
    } catch (error) {
      console.error('Fetch user error:', error)
      logout()  // Invalid token, logout
    }
  }

  // Initialize from localStorage
  const storedToken = localStorage.getItem('token')
  if (storedToken) {
    token.value = storedToken
    fetchCurrentUser()
  }

  return {
    // State
    token: readonly(token),
    user: readonly(user),

    // Getters
    isAuthenticated,
    userDisplayName,

    // Actions
    login,
    logout,
    fetchCurrentUser
  }
})
```

### Using Pinia Store

```vue
<script setup lang="ts">
import { useAuthStore } from '@/stores/auth'
import { storeToRefs } from 'pinia'

const authStore = useAuthStore()

// ✅ GOOD: Use storeToRefs for reactive state
const { user, isAuthenticated, userDisplayName } = storeToRefs(authStore)

// ❌ BAD: Direct destructuring loses reactivity
// const { user, isAuthenticated } = authStore

// Actions can be destructured (they're not reactive)
const { login, logout } = authStore

async function handleLogin() {
  try {
    await login('user@example.com', 'password')
  } catch (error) {
    alert('Login failed')
  }
}
</script>

<template>
  <div v-if="isAuthenticated">
    <p>Welcome, {{ userDisplayName }}!</p>
    <button @click="logout">Logout</button>
  </div>
  <div v-else>
    <button @click="handleLogin">Login</button>
  </div>
</template>
```

## Component Design Patterns

### Smart vs Presentational Components

**Presentational (UI-only)**:
```vue
<!-- components/UserCard.vue -->
<script setup lang="ts">
interface Props {
  user: User
  editable?: boolean
}

const props = defineProps<Props>()
const emit = defineEmits<{
  (e: 'edit', user: User): void
  (e: 'delete', userId: number): void
}>()
</script>

<template>
  <div class="user-card">
    <h3>{{ user.name }}</h3>
    <p>{{ user.email }}</p>
    <button v-if="editable" @click="emit('edit', user)">Edit</button>
    <button v-if="editable" @click="emit('delete', user.id)">Delete</button>
  </div>
</template>
```

**Smart (Data-aware)**:
```vue
<!-- pages/UserProfile.vue -->
<script setup lang="ts">
import { useUser } from '@/composables/useUser'
import UserCard from '@/components/UserCard.vue'

const route = useRoute()
const userId = computed(() => Number(route.params.id))

const { user, loading, updateUser, deleteUser } = useUser(userId)

async function handleEdit(updatedUser: User) {
  await updateUser(updatedUser)
}

async function handleDelete(userId: number) {
  if (confirm('Delete user?')) {
    await deleteUser(userId)
    router.push('/users')
  }
}
</script>

<template>
  <div v-if="loading">Loading...</div>
  <UserCard
    v-else-if="user"
    :user="user"
    :editable="true"
    @edit="handleEdit"
    @delete="handleDelete"
  />
</template>
```

### Async Components (Code Splitting)

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

// ✅ GOOD: Lazy load heavy components
const HeavyChart = defineAsyncComponent(() =>
  import('@/components/HeavyChart.vue')
)

const show HeavyChart = ref(false)
</script>

<template>
  <button @click="showHeavyChart = true">Show Chart</button>

  <!-- Only loads when shown -->
  <Suspense v-if="showHeavyChart">
    <template #default>
      <HeavyChart :data="chartData" />
    </template>
    <template #fallback>
      <div>Loading chart...</div>
    </template>
  </Suspense>
</template>
```

## Performance Optimization

### Computed vs Methods

```vue
<script setup lang="ts">
// ❌ BAD: Method recalculates on every render
function expensiveCalculation() {
  return items.value.reduce((sum, item) => sum + item.price, 0)
}

// ✅ GOOD: Computed caches result
const totalPrice = computed(() => {
  return items.value.reduce((sum, item) => sum + item.price, 0)
})
</script>

<template>
  <!-- Recalculates every render -->
  <p>Total: {{ expensiveCalculation() }}</p>

  <!-- Cached, only updates when items change -->
  <p>Total: {{ totalPrice }}</p>
</template>
```

### v-memo for List Optimization

```vue
<template>
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id, item.updated_at]"
  >
    <!-- Only re-renders if id or updated_at changes -->
    <ItemComponent :item="item" />
  </div>
</template>
```

### Virtual Scrolling for Long Lists

```vue
<script setup lang="ts">
import { useVirtualList } from '@vueuse/core'

const allItems = ref([...])  // 10,000 items

const { list, containerProps, wrapperProps } = useVirtualList(
  allItems,
  { itemHeight: 50 }
)
</script>

<template>
  <div v-bind="containerProps" style="height: 600px">
    <div v-bind="wrapperProps">
      <div
        v-for="{ data, index } in list"
        :key="index"
        style="height: 50px"
      >
        {{ data.name }}
      </div>
    </div>
  </div>
</template>
```

## TypeScript Integration

### Component Props Validation

```vue
<script setup lang="ts">
import type { PropType } from 'vue'

enum UserRole {
  Admin = 'admin',
  User = 'user',
  Guest = 'guest'
}

interface User {
  id: number
  name: string
  role: UserRole
}

// ✅ GOOD: Type-safe props with validation
const props = defineProps({
  user: {
    type: Object as PropType<User>,
    required: true
  },
  role: {
    type: String as PropType<UserRole>,
    default: UserRole.Guest,
    validator: (value: UserRole) => {
      return Object.values(UserRole).includes(value)
    }
  },
  maxItems: {
    type: Number,
    default: 10,
    validator: (value: number) => value > 0 && value <= 100
  }
})
</script>
```

### Type-safe Router

```typescript
// router/index.ts
import type { RouteRecordRaw } from 'vue-router'

export const routes: RouteRecordRaw[] = [
  {
    path: '/users/:id',
    name: 'user-profile',
    component: () => import('@/pages/UserProfile.vue'),
    props: (route) => ({ userId: Number(route.params.id) })
  }
]

// Type-safe navigation
router.push({ name: 'user-profile', params: { id: 123 }})
```

## Best Practices Checklist

- [ ] Use `<script setup>` syntax for components
- [ ] Prefer Composition API over Options API
- [ ] Use TypeScript for type safety
- [ ] Extract reusable logic into composables
- [ ] Use Pinia for global state management
- [ ] Implement smart/presentational component pattern
- [ ] Use `computed` for derived state (not methods)
- [ ] Apply `v-memo` for expensive list items
- [ ] Lazy load heavy components with `defineAsyncComponent`
- [ ] Use virtual scrolling for long lists (> 1000 items)
- [ ] Provide proper TypeScript types for props/emits
- [ ] Use `readonly()` to prevent external mutations
- [ ] Validate props with custom validators
- [ ] Use `toRefs` when destructuring reactive objects
- [ ] Implement proper error handling in composables
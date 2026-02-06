# Vue.js Basics Interview Questions

## Table of Contents

### Core Concepts
- [Q1: What is Vue.js and how does it work?](#q1)
- [Q2: What is the Vue instance lifecycle?](#q2)
- [Q3: What are Vue directives?](#q3)
- [Q4: How does Vue's reactivity system work?](#q4)

### Components
- [Q5: How do you create components in Vue?](#q5)
- [Q6: Explain props and events (parent-child communication)](#q6)
- [Q7: What are slots in Vue?](#q7)

### Composition API
- [Q8: What is the Composition API?](#q8)
- [Q9: Explain ref, reactive, and computed](#q9)
- [Q10: What are composables (custom hooks)?](#q10)

### State Management
- [Q11: What is Vuex and how does it work?](#q11)
- [Q12: What is Pinia?](#q12)

### Advanced Topics
- [Q13: How do you handle routing in Vue?](#q13)
- [Q14: What are watchers in Vue?](#q14)
- [Q15: How do you optimize Vue performance?](#q15)

---

## Core Concepts

<a id="q1"></a>
### Q1: What is Vue.js and how does it work?
**Answer:**

Vue.js is a progressive JavaScript framework for building user interfaces.

**Key features:**
| Feature | Description |
|---------|-------------|
| Reactive data binding | Data changes automatically update the view |
| Component-based | Build UIs with reusable components |
| Virtual DOM | Efficient DOM updates |
| Directives | Special attributes for DOM manipulation |
| Single-file components | HTML, CSS, JS in one .vue file |

```vue
<!-- Single File Component (SFC) -->
<template>
  <div class="greeting">
    <h1>{{ message }}</h1>
    <button @click="updateMessage">Update</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const message = ref('Hello, Vue!')

function updateMessage() {
  message.value = 'Updated!'
}
</script>

<style scoped>
.greeting {
  color: blue;
}
</style>
```

```javascript
// Creating a Vue app
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Global configuration
app.config.errorHandler = (err) => {
  console.error(err)
}

// Global components
app.component('GlobalButton', Button)

// Plugins
app.use(router)
app.use(pinia)

// Mount
app.mount('#app')

// Vue 3 vs Vue 2 differences:
// - Composition API (setup, ref, reactive)
// - Multiple root elements in templates
// - Teleport, Suspense components
// - Better TypeScript support
// - Smaller bundle size
// - createApp instead of new Vue
```

<a id="q2"></a>
### Q2: What is the Vue instance lifecycle?
**Answer:**

```
Creation Phase          Mounting Phase         Updating Phase          Destruction Phase
      │                       │                       │                       │
      ▼                       ▼                       ▼                       ▼
 beforeCreate            beforeMount            beforeUpdate            beforeUnmount
      │                       │                       │                       │
 setup() runs                 │                       │                       │
      │                       │                       │                       │
   created                mounted                 updated                 unmounted
```

```vue
<script>
// Options API lifecycle hooks
export default {
  beforeCreate() {
    // Called before instance is initialized
    // No access to data, computed, methods
  },
  created() {
    // Instance created, data is reactive
    // No DOM access yet
    console.log(this.message) // Works
  },
  beforeMount() {
    // Before mounting to DOM
    // Template compiled, but not rendered
  },
  mounted() {
    // Component mounted to DOM
    // Can access DOM elements
    console.log(this.$el) // DOM element
  },
  beforeUpdate() {
    // Data changed, before DOM update
  },
  updated() {
    // DOM has been updated
    // Avoid changing state here (infinite loop)
  },
  beforeUnmount() {
    // Before component is destroyed
    // Clean up timers, event listeners
  },
  unmounted() {
    // Component destroyed
    // All directives unbound
  }
}
</script>
```

```vue
<script setup>
// Composition API lifecycle hooks
import { 
  onBeforeMount, 
  onMounted, 
  onBeforeUpdate, 
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  onActivated,      // keep-alive activated
  onDeactivated,    // keep-alive deactivated
  onErrorCaptured   // Error from descendant
} from 'vue'

// No beforeCreate/created - setup() runs at that time

onBeforeMount(() => {
  console.log('Before mount')
})

onMounted(() => {
  console.log('Mounted')
  // DOM is available
  // Fetch data, set up subscriptions
})

onBeforeUpdate(() => {
  console.log('Before update')
})

onUpdated(() => {
  console.log('Updated')
})

onBeforeUnmount(() => {
  console.log('Before unmount')
  // Clean up
})

onUnmounted(() => {
  console.log('Unmounted')
})

// Error handling
onErrorCaptured((err, component, info) => {
  console.error(err)
  return false // Prevent propagation
})
</script>
```

<a id="q3"></a>
### Q3: What are Vue directives?
**Answer:**

Directives are special attributes with the `v-` prefix that apply reactive behavior to the DOM.

```vue
<template>
  <!-- v-bind: Bind attributes/props -->
  <img v-bind:src="imageUrl" />
  <img :src="imageUrl" />  <!-- Shorthand -->
  <div :class="{ active: isActive, error: hasError }"></div>
  <div :style="{ color: textColor, fontSize: size + 'px' }"></div>
  
  <!-- v-on: Event handling -->
  <button v-on:click="handleClick">Click</button>
  <button @click="handleClick">Click</button>  <!-- Shorthand -->
  <button @click.prevent="handleSubmit">Submit</button>
  <input @keyup.enter="search" />
  
  <!-- Event modifiers -->
  <button @click.stop="handler">Stop propagation</button>
  <form @submit.prevent="handler">Prevent default</form>
  <button @click.once="handler">Trigger once</button>
  <div @click.self="handler">Only if clicked element itself</div>
  
  <!-- v-model: Two-way binding -->
  <input v-model="text" />
  <input v-model.trim="text" />  <!-- Trim whitespace -->
  <input v-model.number="age" /> <!-- Cast to number -->
  <input v-model.lazy="text" />  <!-- Sync on change, not input -->
  
  <!-- v-if/v-else-if/v-else: Conditional rendering -->
  <div v-if="type === 'A'">A</div>
  <div v-else-if="type === 'B'">B</div>
  <div v-else>Other</div>
  
  <!-- v-show: Toggle visibility (CSS display) -->
  <div v-show="isVisible">Visible</div>
  
  <!-- v-for: List rendering -->
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  <li v-for="(item, index) in items" :key="item.id">
    {{ index }}: {{ item.name }}
  </li>
  <li v-for="(value, key, index) in object" :key="key">
    {{ key }}: {{ value }}
  </li>
  
  <!-- v-slot: Named slots -->
  <MyComponent>
    <template v-slot:header>Header content</template>
    <template #footer>Footer content</template>  <!-- Shorthand -->
  </MyComponent>
  
  <!-- v-pre: Skip compilation -->
  <span v-pre>{{ this will not be compiled }}</span>
  
  <!-- v-once: Render once, no updates -->
  <span v-once>{{ staticContent }}</span>
  
  <!-- v-memo: Memoize part of template (Vue 3.2+) -->
  <div v-memo="[item.id]">
    Expensive content that only updates when item.id changes
  </div>
</template>
```

```javascript
// Custom directive
const app = createApp(App)

// Global directive
app.directive('focus', {
  mounted(el) {
    el.focus()
  }
})

// Usage: <input v-focus />

// Directive with hooks
app.directive('highlight', {
  // Called before bound element is mounted
  created(el, binding, vnode) {},
  
  // When bound element is inserted into DOM
  mounted(el, binding, vnode) {
    el.style.backgroundColor = binding.value || 'yellow'
  },
  
  // When containing component updates
  updated(el, binding, vnode, prevVnode) {
    el.style.backgroundColor = binding.value
  },
  
  // Before bound element is unmounted
  beforeUnmount(el, binding, vnode) {},
  
  // When bound element is unmounted
  unmounted(el, binding, vnode) {}
})

// Usage: <div v-highlight="'red'">Highlighted</div>

// Directive shorthand (mounted + updated)
app.directive('color', (el, binding) => {
  el.style.color = binding.value
})
```

<a id="q4"></a>
### Q4: How does Vue's reactivity system work?
**Answer:**

Vue 3 uses JavaScript Proxies to track changes and trigger updates.

```javascript
// Vue 3 Reactivity (Proxy-based)
import { reactive, ref, computed, watch, watchEffect } from 'vue'

// reactive - for objects
const state = reactive({
  count: 0,
  user: { name: 'John' }
})

state.count++           // Reactive
state.user.name = 'Jane' // Deep reactive

// ref - for primitives (and objects)
const count = ref(0)
count.value++          // Need .value in JS
// In template: {{ count }} - auto unwrapped

// Why reactive and ref?
// reactive: Proxy can't track reassignment
let state = reactive({ count: 0 })
state = { count: 1 }  // Lost reactivity!

// ref: Wraps in object, tracks .value
const count = ref(0)
count.value = 1       // Still reactive

// computed - derived state
const doubled = computed(() => count.value * 2)
console.log(doubled.value) // Read-only

// Writable computed
const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(value) {
    const parts = value.split(' ')
    firstName.value = parts[0]
    lastName.value = parts[1]
  }
})

// How Proxies work (simplified)
const data = { count: 0 }

const proxy = new Proxy(data, {
  get(target, key) {
    track(target, key)  // Track dependency
    return target[key]
  },
  set(target, key, value) {
    target[key] = value
    trigger(target, key) // Trigger updates
    return true
  }
})

// Reactivity caveats
const state = reactive({ items: [] })

// Adding new properties is reactive (unlike Vue 2)
state.newProp = 'value' // Reactive in Vue 3

// Array methods are reactive
state.items.push(1)     // Reactive
state.items[0] = 2      // Reactive

// Destructuring loses reactivity
const { count } = state  // count is not reactive!

// Solution: toRefs
import { toRefs } from 'vue'
const { count } = toRefs(state) // count.value is reactive

// Or use storeToRefs for Pinia
import { storeToRefs } from 'pinia'
const { count } = storeToRefs(store)
```

---

## Components

<a id="q5"></a>
### Q5: How do you create components in Vue?
**Answer:**

```vue
<!-- Composition API with <script setup> (Vue 3.2+, recommended) -->
<script setup>
import { ref, computed } from 'vue'
import ChildComponent from './ChildComponent.vue'

// Props
const props = defineProps({
  title: String,
  count: {
    type: Number,
    required: true,
    default: 0,
    validator: (value) => value >= 0
  }
})

// Emits
const emit = defineEmits(['update', 'delete'])

// State
const message = ref('Hello')

// Methods
function handleClick() {
  emit('update', message.value)
}

// Computed
const uppercaseTitle = computed(() => props.title.toUpperCase())

// Expose (for template refs)
defineExpose({
  message,
  handleClick
})
</script>

<template>
  <div>
    <h1>{{ uppercaseTitle }}</h1>
    <p>{{ message }}</p>
    <button @click="handleClick">Update</button>
    <ChildComponent />
  </div>
</template>
```

```vue
<!-- Composition API with setup() -->
<script>
import { ref, computed } from 'vue'

export default {
  props: {
    title: String
  },
  emits: ['update'],
  setup(props, { emit, attrs, slots, expose }) {
    const count = ref(0)
    
    const increment = () => {
      count.value++
      emit('update', count.value)
    }
    
    // Expose to parent via template ref
    expose({ count })
    
    return {
      count,
      increment
    }
  }
}
</script>
```

```vue
<!-- Options API -->
<script>
export default {
  name: 'MyComponent',
  
  components: {
    ChildComponent
  },
  
  props: {
    title: {
      type: String,
      required: true
    }
  },
  
  emits: ['update'],
  
  data() {
    return {
      count: 0
    }
  },
  
  computed: {
    doubled() {
      return this.count * 2
    }
  },
  
  methods: {
    increment() {
      this.count++
      this.$emit('update', this.count)
    }
  },
  
  watch: {
    count(newValue, oldValue) {
      console.log(`Count changed from ${oldValue} to ${newValue}`)
    }
  },
  
  mounted() {
    console.log('Component mounted')
  }
}
</script>
```

<a id="q6"></a>
### Q6: Explain props and events (parent-child communication)
**Answer:**

```vue
<!-- Parent.vue -->
<script setup>
import { ref } from 'vue'
import Child from './Child.vue'

const parentMessage = ref('Hello from parent')
const items = ref([{ id: 1, name: 'Item 1' }])

function handleUpdate(newValue) {
  parentMessage.value = newValue
}

function handleDelete(id) {
  items.value = items.value.filter(item => item.id !== id)
}
</script>

<template>
  <!-- Pass props down -->
  <Child 
    :message="parentMessage"
    :items="items"
    title="Static prop"
    @update="handleUpdate"
    @delete="handleDelete"
  />
  
  <!-- v-model on custom component -->
  <Child v-model="parentMessage" />
  <!-- Equivalent to: -->
  <Child :modelValue="parentMessage" @update:modelValue="parentMessage = $event" />
  
  <!-- Multiple v-model -->
  <Child v-model:title="title" v-model:content="content" />
</template>
```

```vue
<!-- Child.vue -->
<script setup>
// Define props with types and validation
const props = defineProps({
  message: {
    type: String,
    required: true
  },
  items: {
    type: Array,
    default: () => []
  },
  title: String,
  modelValue: String  // For v-model
})

// Define emits
const emit = defineEmits(['update', 'delete', 'update:modelValue'])

// Or with validation
const emit = defineEmits({
  update: (value) => typeof value === 'string',
  delete: (id) => typeof id === 'number'
})

function updateMessage() {
  emit('update', 'New message')
}

function deleteItem(id) {
  emit('delete', id)
}

// For v-model
function updateModelValue(event) {
  emit('update:modelValue', event.target.value)
}
</script>

<template>
  <div>
    <p>{{ message }}</p>
    <button @click="updateMessage">Update</button>
    
    <ul>
      <li v-for="item in items" :key="item.id">
        {{ item.name }}
        <button @click="deleteItem(item.id)">Delete</button>
      </li>
    </ul>
    
    <!-- v-model support -->
    <input :value="modelValue" @input="updateModelValue" />
  </div>
</template>
```

```vue
<!-- TypeScript with defineProps -->
<script setup lang="ts">
interface Item {
  id: number
  name: string
}

interface Props {
  message: string
  items?: Item[]
  title?: string
}

const props = withDefaults(defineProps<Props>(), {
  items: () => [],
  title: 'Default Title'
})

const emit = defineEmits<{
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
}>()
</script>
```

<a id="q7"></a>
### Q7: What are slots in Vue?
**Answer:**

Slots allow parent components to inject content into child components.

```vue
<!-- Child.vue - Define slots -->
<template>
  <div class="card">
    <!-- Default slot -->
    <slot>Default content if nothing provided</slot>
    
    <!-- Named slots -->
    <header>
      <slot name="header"></slot>
    </header>
    
    <main>
      <slot></slot>  <!-- Default slot -->
    </main>
    
    <footer>
      <slot name="footer"></slot>
    </footer>
    
    <!-- Scoped slot - pass data to parent -->
    <ul>
      <li v-for="item in items" :key="item.id">
        <slot name="item" :item="item" :index="index">
          {{ item.name }}  <!-- Fallback -->
        </slot>
      </li>
    </ul>
  </div>
</template>
```

```vue
<!-- Parent.vue - Use slots -->
<template>
  <Card>
    <!-- Default slot content -->
    <p>This goes into the default slot</p>
    
    <!-- Named slots -->
    <template #header>
      <h1>Card Header</h1>
    </template>
    
    <template v-slot:footer>  <!-- Long form -->
      <p>Card Footer</p>
    </template>
    
    <!-- Scoped slot - receive data from child -->
    <template #item="{ item, index }">
      <span>{{ index }}: {{ item.name }}</span>
      <button @click="edit(item)">Edit</button>
    </template>
  </Card>
  
  <!-- Shorthand for default scoped slot -->
  <Card v-slot="{ item }">
    {{ item.name }}
  </Card>
</template>
```

```vue
<!-- Dynamic slot names -->
<template>
  <Component>
    <template #[dynamicSlotName]>
      Dynamic slot content
    </template>
  </Component>
</template>

<script setup>
const dynamicSlotName = ref('header')
</script>
```

```vue
<!-- Renderless component pattern -->
<!-- MouseTracker.vue -->
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const x = ref(0)
const y = ref(0)

function update(event) {
  x.value = event.pageX
  y.value = event.pageY
}

onMounted(() => window.addEventListener('mousemove', update))
onUnmounted(() => window.removeEventListener('mousemove', update))
</script>

<template>
  <slot :x="x" :y="y"></slot>
</template>

<!-- Usage -->
<MouseTracker v-slot="{ x, y }">
  Mouse position: {{ x }}, {{ y }}
</MouseTracker>
```

---

## Composition API

<a id="q8"></a>
### Q8: What is the Composition API?
**Answer:**

The Composition API is a set of functions for organizing component logic by feature rather than by option.

```vue
<!-- Options API - organized by option type -->
<script>
export default {
  data() {
    return {
      // User feature
      user: null,
      userLoading: false,
      // Posts feature  
      posts: [],
      postsLoading: false
    }
  },
  computed: {
    // User feature
    fullName() { /* ... */ },
    // Posts feature
    publishedPosts() { /* ... */ }
  },
  methods: {
    // User feature
    fetchUser() { /* ... */ },
    // Posts feature
    fetchPosts() { /* ... */ }
  },
  mounted() {
    this.fetchUser()
    this.fetchPosts()
  }
}
</script>
```

```vue
<!-- Composition API - organized by feature -->
<script setup>
import { useUser } from './composables/useUser'
import { usePosts } from './composables/usePosts'

// User feature - all together
const { user, loading: userLoading, fullName, fetchUser } = useUser()

// Posts feature - all together
const { posts, loading: postsLoading, publishedPosts, fetchPosts } = usePosts()

onMounted(() => {
  fetchUser()
  fetchPosts()
})
</script>
```

```javascript
// composables/useUser.js
import { ref, computed, onMounted } from 'vue'

export function useUser() {
  const user = ref(null)
  const loading = ref(false)
  
  const fullName = computed(() => {
    if (!user.value) return ''
    return `${user.value.firstName} ${user.value.lastName}`
  })
  
  async function fetchUser() {
    loading.value = true
    try {
      user.value = await api.getUser()
    } finally {
      loading.value = false
    }
  }
  
  return {
    user,
    loading,
    fullName,
    fetchUser
  }
}
```

**Benefits of Composition API:**
| Benefit | Description |
|---------|-------------|
| Better code organization | Group related logic together |
| Better reusability | Extract and share logic easily |
| Better TypeScript support | Full type inference |
| Smaller bundle size | Tree-shaking friendly |
| More flexible | Use only what you need |

<a id="q9"></a>
### Q9: Explain ref, reactive, and computed
**Answer:**

```javascript
import { ref, reactive, computed, readonly, shallowRef, shallowReactive } from 'vue'

// ref - wrap any value
const count = ref(0)
const user = ref({ name: 'John' })

// Access/modify with .value
count.value++
user.value.name = 'Jane'

// In template, auto-unwrapped
// {{ count }} works, not {{ count.value }}

// reactive - for objects only
const state = reactive({
  count: 0,
  user: { name: 'John' }
})

// Direct access, no .value
state.count++
state.user.name = 'Jane'

// Nested objects are also reactive
state.user.address = { city: 'NYC' } // Reactive

// ref vs reactive
// Use ref for:
// - Primitives (string, number, boolean)
// - When you need to reassign the whole value
// - When passing to functions

// Use reactive for:
// - Objects/arrays you won't reassign
// - When you want to avoid .value

// computed - derived reactive state
const firstName = ref('John')
const lastName = ref('Doe')

// Read-only computed
const fullName = computed(() => `${firstName.value} ${lastName.value}`)

// Writable computed
const fullNameWritable = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(newValue) {
    const parts = newValue.split(' ')
    firstName.value = parts[0]
    lastName.value = parts[1] || ''
  }
})

fullNameWritable.value = 'Jane Smith' // Sets firstName and lastName

// readonly - prevent modifications
const original = reactive({ count: 0 })
const copy = readonly(original)

copy.count++ // Warning in dev, ignored in prod

// shallowRef - only .value is reactive
const shallowUser = shallowRef({ name: 'John' })
shallowUser.value = { name: 'Jane' }  // Triggers update
shallowUser.value.name = 'Jim'        // Does NOT trigger update

// shallowReactive - only root level is reactive
const shallowState = shallowReactive({
  nested: { count: 0 }
})
shallowState.nested = { count: 1 }    // Triggers update
shallowState.nested.count = 2         // Does NOT trigger update

// toRef - create ref from reactive property
const state = reactive({ count: 0 })
const countRef = toRef(state, 'count')
// countRef.value and state.count are synced

// toRefs - convert all properties to refs
const { count, name } = toRefs(state)
// Useful for destructuring without losing reactivity
```

<a id="q10"></a>
### Q10: What are composables (custom hooks)?
**Answer:**

Composables are functions that encapsulate and reuse stateful logic using the Composition API.

```javascript
// composables/useMouse.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)
  
  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }
  
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))
  
  return { x, y }
}

// composables/useFetch.js
import { ref, watchEffect, toValue } from 'vue'

export function useFetch(url) {
  const data = ref(null)
  const error = ref(null)
  const loading = ref(false)
  
  async function fetchData() {
    loading.value = true
    error.value = null
    
    try {
      const response = await fetch(toValue(url))
      data.value = await response.json()
    } catch (err) {
      error.value = err
    } finally {
      loading.value = false
    }
  }
  
  // Refetch when url changes
  watchEffect(() => {
    fetchData()
  })
  
  return { data, error, loading, refetch: fetchData }
}

// composables/useLocalStorage.js
import { ref, watch } from 'vue'

export function useLocalStorage(key, defaultValue) {
  const stored = localStorage.getItem(key)
  const data = ref(stored ? JSON.parse(stored) : defaultValue)
  
  watch(data, (newValue) => {
    localStorage.setItem(key, JSON.stringify(newValue))
  }, { deep: true })
  
  return data
}

// composables/useDebounce.js
import { ref, watch } from 'vue'

export function useDebounce(value, delay = 300) {
  const debouncedValue = ref(value.value)
  let timeout
  
  watch(value, (newValue) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debouncedValue.value = newValue
    }, delay)
  })
  
  return debouncedValue
}

// Usage in component
<script setup>
import { ref } from 'vue'
import { useMouse } from '@/composables/useMouse'
import { useFetch } from '@/composables/useFetch'
import { useLocalStorage } from '@/composables/useLocalStorage'
import { useDebounce } from '@/composables/useDebounce'

// Mouse position
const { x, y } = useMouse()

// Fetch data
const url = ref('/api/users')
const { data: users, loading, error, refetch } = useFetch(url)

// Persistent state
const theme = useLocalStorage('theme', 'light')

// Debounced search
const search = ref('')
const debouncedSearch = useDebounce(search)
</script>
```

---

## State Management

<a id="q11"></a>
### Q11: What is Vuex and how does it work?
**Answer:**

Vuex is the official state management library for Vue.js (legacy, prefer Pinia for new projects).

```javascript
// store/index.js
import { createStore } from 'vuex'

export default createStore({
  // State - single source of truth
  state: {
    count: 0,
    user: null,
    todos: []
  },
  
  // Getters - computed properties for store
  getters: {
    doubleCount: (state) => state.count * 2,
    completedTodos: (state) => state.todos.filter(t => t.done),
    getTodoById: (state) => (id) => state.todos.find(t => t.id === id)
  },
  
  // Mutations - synchronous state changes
  mutations: {
    INCREMENT(state) {
      state.count++
    },
    SET_USER(state, user) {
      state.user = user
    },
    ADD_TODO(state, todo) {
      state.todos.push(todo)
    }
  },
  
  // Actions - async operations, commit mutations
  actions: {
    async fetchUser({ commit }) {
      const user = await api.getUser()
      commit('SET_USER', user)
    },
    async addTodo({ commit }, title) {
      const todo = await api.createTodo(title)
      commit('ADD_TODO', todo)
    }
  },
  
  // Modules - split store into modules
  modules: {
    auth: authModule,
    todos: todosModule
  }
})

// Modules
const authModule = {
  namespaced: true,
  state: () => ({
    user: null
  }),
  mutations: {
    SET_USER(state, user) {
      state.user = user
    }
  },
  actions: {
    async login({ commit }, credentials) {
      const user = await api.login(credentials)
      commit('SET_USER', user)
    }
  }
}
```

```vue
<!-- Usage in components -->
<script>
import { mapState, mapGetters, mapMutations, mapActions } from 'vuex'

// Options API
export default {
  computed: {
    // Map state
    ...mapState(['count', 'user']),
    ...mapState('auth', ['user']),  // Namespaced
    
    // Map getters
    ...mapGetters(['doubleCount', 'completedTodos']),
    
    // Direct access
    count() {
      return this.$store.state.count
    }
  },
  methods: {
    // Map mutations
    ...mapMutations(['INCREMENT']),
    
    // Map actions
    ...mapActions(['fetchUser']),
    ...mapActions('auth', ['login']),  // Namespaced
    
    // Direct access
    increment() {
      this.$store.commit('INCREMENT')
    },
    async loadUser() {
      await this.$store.dispatch('fetchUser')
    }
  }
}
</script>
```

```vue
<!-- Composition API -->
<script setup>
import { useStore } from 'vuex'
import { computed } from 'vue'

const store = useStore()

// State
const count = computed(() => store.state.count)
const user = computed(() => store.state.auth.user)

// Getters
const doubleCount = computed(() => store.getters.doubleCount)

// Mutations
function increment() {
  store.commit('INCREMENT')
}

// Actions
async function login(credentials) {
  await store.dispatch('auth/login', credentials)
}
</script>
```

<a id="q12"></a>
### Q12: What is Pinia?
**Answer:**

Pinia is the recommended state management library for Vue 3 (successor to Vuex).

```javascript
// stores/counter.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

// Option syntax
export const useCounterStore = defineStore('counter', {
  state: () => ({
    count: 0,
    name: 'Counter'
  }),
  
  getters: {
    doubleCount: (state) => state.count * 2,
    // Use other getters
    doubleCountPlusOne() {
      return this.doubleCount + 1
    }
  },
  
  actions: {
    increment() {
      this.count++
    },
    async fetchAndSet() {
      const data = await api.getData()
      this.count = data.count
    }
  }
})

// Setup syntax (recommended)
export const useCounterStore = defineStore('counter', () => {
  // State
  const count = ref(0)
  const name = ref('Counter')
  
  // Getters
  const doubleCount = computed(() => count.value * 2)
  
  // Actions
  function increment() {
    count.value++
  }
  
  async function fetchAndSet() {
    const data = await api.getData()
    count.value = data.count
  }
  
  return { count, name, doubleCount, increment, fetchAndSet }
})
```

```vue
<!-- Usage in components -->
<script setup>
import { useCounterStore } from '@/stores/counter'
import { storeToRefs } from 'pinia'

const store = useCounterStore()

// Direct access
console.log(store.count)
console.log(store.doubleCount)
store.increment()

// Destructure with storeToRefs (keeps reactivity)
const { count, name, doubleCount } = storeToRefs(store)
// Actions can be destructured directly
const { increment, fetchAndSet } = store

// Modify state directly (allowed in Pinia)
store.count++

// Or use $patch for multiple changes
store.$patch({
  count: store.count + 1,
  name: 'New Name'
})

// $patch with function
store.$patch((state) => {
  state.items.push({ id: 1 })
  state.count++
})

// Reset state
store.$reset()

// Subscribe to changes
store.$subscribe((mutation, state) => {
  console.log('State changed:', mutation.type)
  localStorage.setItem('counter', JSON.stringify(state))
})
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Double: {{ doubleCount }}</p>
    <button @click="increment">+1</button>
  </div>
</template>
```

**Pinia vs Vuex:**
| Feature | Pinia | Vuex |
|---------|-------|------|
| Mutations | Not needed | Required for state changes |
| TypeScript | Excellent support | Limited |
| Devtools | Full support | Full support |
| Modules | Flat structure | Nested namespaces |
| SSR | Built-in | Requires setup |
| Bundle size | Smaller | Larger |

---

## Advanced Topics

<a id="q13"></a>
### Q13: How do you handle routing in Vue?
**Answer:**

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/views/Home.vue')  // Lazy loading
  },
  {
    path: '/about',
    name: 'About',
    component: () => import('@/views/About.vue'),
    meta: { requiresAuth: false }
  },
  {
    path: '/users/:id',
    name: 'User',
    component: () => import('@/views/User.vue'),
    props: true  // Pass params as props
  },
  {
    path: '/dashboard',
    component: () => import('@/layouts/Dashboard.vue'),
    meta: { requiresAuth: true },
    children: [
      {
        path: '',
        name: 'DashboardHome',
        component: () => import('@/views/DashboardHome.vue')
      },
      {
        path: 'settings',
        name: 'Settings',
        component: () => import('@/views/Settings.vue')
      }
    ]
  },
  {
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: () => import('@/views/NotFound.vue')
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition
    return { top: 0 }
  }
})

// Navigation guards
router.beforeEach((to, from) => {
  const isAuthenticated = checkAuth()
  
  if (to.meta.requiresAuth && !isAuthenticated) {
    return { name: 'Login', query: { redirect: to.fullPath } }
  }
})

router.afterEach((to, from) => {
  document.title = to.meta.title || 'My App'
})

export default router
```

```vue
<!-- Usage in components -->
<script setup>
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

// Access route params
console.log(route.params.id)
console.log(route.query.search)
console.log(route.path)
console.log(route.name)

// Programmatic navigation
function goToUser(id) {
  router.push(`/users/${id}`)
  // Or with name
  router.push({ name: 'User', params: { id } })
  // With query
  router.push({ path: '/search', query: { q: 'vue' } })
}

function goBack() {
  router.back()
  // Or: router.go(-1)
}

// Replace (no history entry)
router.replace('/new-path')
</script>

<template>
  <!-- Router links -->
  <router-link to="/">Home</router-link>
  <router-link :to="{ name: 'User', params: { id: 1 } }">User 1</router-link>
  
  <!-- Active class -->
  <router-link to="/about" active-class="active">About</router-link>
  
  <!-- Router view (renders matched component) -->
  <router-view />
  
  <!-- Named views -->
  <router-view name="sidebar" />
  <router-view name="main" />
  
  <!-- Keep-alive with router -->
  <router-view v-slot="{ Component }">
    <keep-alive>
      <component :is="Component" />
    </keep-alive>
  </router-view>
</template>
```

<a id="q14"></a>
### Q14: What are watchers in Vue?
**Answer:**

Watchers react to data changes and perform side effects.

```vue
<script setup>
import { ref, watch, watchEffect, watchPostEffect } from 'vue'

const count = ref(0)
const user = ref({ name: 'John' })
const items = ref([1, 2, 3])

// Basic watch
watch(count, (newValue, oldValue) => {
  console.log(`Count changed from ${oldValue} to ${newValue}`)
})

// Watch with options
watch(count, (newValue) => {
  console.log(newValue)
}, {
  immediate: true,  // Run immediately
  once: true        // Run only once (Vue 3.4+)
})

// Watch reactive object (deep by default)
watch(user, (newUser) => {
  console.log('User changed:', newUser)
})

// Watch specific property
watch(
  () => user.value.name,
  (newName) => {
    console.log('Name changed:', newName)
  }
)

// Watch multiple sources
watch(
  [count, () => user.value.name],
  ([newCount, newName], [oldCount, oldName]) => {
    console.log('Count or name changed')
  }
)

// Deep watch (for nested changes in ref)
watch(items, (newItems) => {
  console.log('Items changed')
}, { deep: true })

// Immediate + async
watch(
  () => route.params.id,
  async (newId) => {
    const data = await fetchUser(newId)
    user.value = data
  },
  { immediate: true }
)

// watchEffect - auto-tracks dependencies
watchEffect(() => {
  // Tracks any reactive ref used inside
  console.log(`Count is ${count.value}`)
  console.log(`User is ${user.value.name}`)
})

// watchEffect with cleanup
watchEffect((onCleanup) => {
  const controller = new AbortController()
  
  fetch(url.value, { signal: controller.signal })
    .then(data => { /* ... */ })
  
  onCleanup(() => {
    controller.abort()
  })
})

// watchPostEffect - runs after DOM update
watchPostEffect(() => {
  // DOM is updated here
  console.log(element.value.textContent)
})

// Stop watching
const stop = watch(count, () => {})
stop() // Stop watching

// Watch in Options API
export default {
  watch: {
    count(newValue, oldValue) {
      console.log('Count changed')
    },
    'user.name'(newName) {
      console.log('Name changed')
    },
    items: {
      handler(newItems) {
        console.log('Items changed')
      },
      deep: true,
      immediate: true
    }
  }
}
</script>
```

<a id="q15"></a>
### Q15: How do you optimize Vue performance?
**Answer:**

```vue
<script setup>
import { shallowRef, shallowReactive, markRaw, computed } from 'vue'

// 1. Use v-once for static content
// <div v-once>{{ staticContent }}</div>

// 2. Use v-memo to skip updates (Vue 3.2+)
// <div v-for="item in list" :key="item.id" v-memo="[item.id === selected]">

// 3. Use shallowRef/shallowReactive for large objects
const largeList = shallowRef([])
const shallowState = shallowReactive({ nested: { data: [] } })

// 4. Mark objects as non-reactive
const heavyObject = markRaw({ /* large object */ })

// 5. Computed properties cache values
const expensiveValue = computed(() => {
  return heavyComputation(data.value)
})

// 6. Debounce expensive operations
import { useDebounceFn } from '@vueuse/core'

const debouncedSearch = useDebounceFn((query) => {
  performSearch(query)
}, 300)
</script>

<template>
  <!-- 7. Use v-show for frequent toggles (cheaper than v-if) -->
  <div v-show="isVisible">Toggles visibility with CSS</div>
  
  <!-- 8. Use v-if for rarely changed conditions -->
  <div v-if="isAdmin">Admin content</div>
  
  <!-- 9. Avoid v-if with v-for on same element -->
  <!-- BAD -->
  <li v-for="item in items" v-if="item.active">{{ item.name }}</li>
  
  <!-- GOOD - filter in computed -->
  <li v-for="item in activeItems" :key="item.id">{{ item.name }}</li>
  
  <!-- 10. Use KeepAlive for cached components -->
  <KeepAlive :max="10">
    <component :is="currentComponent" />
  </KeepAlive>
  
  <!-- 11. Lazy load images -->
  <img v-lazy="imageUrl" />
</template>

<script setup>
// 12. Code splitting with dynamic imports
const HeavyComponent = defineAsyncComponent(() =>
  import('./HeavyComponent.vue')
)

// 13. Virtual scrolling for long lists
import { RecycleScroller } from 'vue-virtual-scroller'

// 14. Use production build
// npm run build (enables minification, dead code elimination)

// 15. Analyze bundle size
// npm run build -- --report
</script>
```

```javascript
// Additional optimizations

// 16. Use functional components for simple presentational components
// (Less overhead, no instance)

// 17. Object freeze for non-reactive data
const staticData = Object.freeze({ /* ... */ })

// 18. Lazy load routes
const routes = [
  {
    path: '/dashboard',
    component: () => import('./views/Dashboard.vue')
  }
]

// 19. Preload critical resources
// <link rel="preload" href="/api/critical-data" as="fetch">

// 20. Use SSR/SSG for better initial load
// Nuxt.js for Vue
```

---

[← React Basics](react-basics.md) | [Back to Frontend Index](README.md)

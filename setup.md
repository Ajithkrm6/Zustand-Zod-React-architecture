# Zustand-Zod-React-architecture
A complete guide to setting up scalable, maintainable projects using Zustand + Zod + React architecture.


## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Folder Structure](#folder-structure)
3. [Core Concepts](#core-concepts)
4. [Zod Utilities System](#zod-utilities-system)
5. [Step-by-Step Implementation](#step-by-step-implementation)
6. [Configuration Options](#configuration-options)
7. [Single-Step Setup](#single-step-setup)
8. [Multi-Step Setup](#multi-step-setup)
9. [Complete Working Examples](#complete-working-examples)
10. [Best Practices](#best-practices)

---

## High-Level Architecture

### The Dependency Flow (Correct Direction)

```
┌─────────────────────────────────┐
│      React Components           │
│  (UI rendering, user input)     │
└──────────────┬──────────────────┘
               │ uses
               ↓
┌─────────────────────────────────┐
│    Zustand Store (index.ts)     │
│  (state management, actions)    │
└──────────────┬──────────────────┘
               │ calls & validates
               ↓
┌──────────────────────────────────────┐
│  Schemas + API-Wrappers              │
│  (validation, business logic)        │
└──────────────┬───────────────────────┘
               │ uses
               ↓
┌──────────────────────────────────────┐
│  External Dependencies               │
│  (Zod, apiClient, utilities)         │
└──────────────────────────────────────┘
```

### Why This Structure?

| Layer | Responsibility | Benefits |
|-------|---|---|
| **Components** | Display data, capture user input | Reusable, testable, declarative |
| **Store** | Manage state, orchestrate flows | Single source of truth, predictable |
| **Schemas** | Validate & transform data | Reusable validation, type safety |
| **API-Wrappers** | Call endpoints, map responses | No framework dependencies, testable |

---

## Folder Structure

### Complete Project Layout

```
app/stores/my-feature/
├── api-wrappers/              ← Pure API functions
│   ├── step-1.ts              ← API calls for step 1
│   ├── step-2.ts              ← API calls for step 2
│   └── step-3.ts              ← API calls for step 3
│
├── step-1/                    ← Step 1 schema & defaults
│   └── schema/
│       ├── index.ts           ← Combined schema
│       ├── step-1.ts          ← Step 1 fields
│       └── step-2.ts          ← Step 2 fields (if sub-steps)
│
├── step-2/                    ← Step 2 schema & defaults
│   └── schema/
│       └── index.ts
│
├── step-3/                    ← Step 3 schema & defaults
│   └── schema/
│       └── index.ts
│
└── index.ts                   ← Main Zustand store (comes last!)
```

### Minimal Single-Step Layout

```
app/stores/my-simple-form/
├── api-wrappers/
│   └── submit.ts              ← Just one API call
│
├── schema/
│   └── index.ts               ← Single schema with defaults
│
└── index.ts                   ← Store (minimal)
```

---

## Core Concepts

### 1. Schemas (The Truth Source)

```typescript
// Schemas define:
// 1. What fields exist (shape)
// 2. What values are valid (rules)
// 3. What happens on submit (transforms)
// 4. Default values (initialization)
// 5. Conditional requirements (complex logic)

const schema = z.object({
  email: z.string().email("Invalid email"),
  age: z.number().min(18, "Must be 18+"),
  country: z.string().default("US"),
})

const defaults = {
  email: "",
  age: 0,
  country: "US",
}
```

### 2. API-Wrappers (Reusable Functions)

```typescript
// API-wrappers:
// 1. Are pure functions (no state, testable)
// 2. Call API endpoints
// 3. Transform responses
// 4. Handle errors
// 5. Can be reused in multiple stores

export async function submitForm(data: FormData) {
  try {
    const { data: result, error } = await apiClient.POST(endpoint, { body: data })
    if (error) throw error
    return result
  } catch (error) {
    throw new Error("Failed to submit")
  }
}
```

### 3. Store (Orchestration)

```typescript
// Store:
// 1. Holds all form state
// 2. Validates with schemas
// 3. Calls API-wrappers
// 4. Updates state based on API response
// 5. Manages step progression

export const useMyForm = create((set, get) => ({
  // State
  email: "",
  age: 0,
  currentStep: 1,
  
  // Actions
  submitStep: async (formData) => {
    const valid = await schema.parseAsync(formData)
    const result = await submitFormAPI(valid)
    set({ ...result, currentStep: 2 })
  },
}))
```

---

## Zod Utilities System

### Quick Reference

| Utility | Purpose | Example |
|---------|---------|---------|
| `wrapInName()` | Name objects for destructuring | `const { email } = wrapInName("email", {...})` |
| `formText()` | Text field validator | `const { name } = formText("name").alwaysRequired` |
| `formNumber()` | Number field validator | `const { age } = formNumber("age").optionalByDefault` |
| `formCheckbox()` | Boolean field validator | `const { agree } = formCheckbox("agree").alwaysRequired` |
| `formSelect()` | Select dropdown validator | `const { country } = formSelect("country").alwaysRequired` |
| `RequiredDetails` | Requirement tracker | `new RequiredDetails(true, false, null)` |
| `createRequirementsFromSchema()` | Requirement builder | Combines step requirements |

### `wrapInName()` - How It Works

```typescript
// BEFORE
const name = {
  schema: z.string(),
  required: new RequiredDetails(true, false, null),
}
// Problem: Can't destructure, no name attached

// AFTER
const { name } = wrapInName("name", {
  schema: z.string(),
  required: new RequiredDetails(true, false, null),
})
// Now: name.schema, name.required, name.name all exist
// Can destructure: const { name } = wrapInName(...)
```

### `RequiredDetails` - Tracking Requirements

```typescript
// Simple required field
new RequiredDetails(
  true,     // Is required?
  false,    // Is array?
  null      // Subfields?
)

// Complex field with subfields
new RequiredDetails(
  true,
  false,
  {
    street: new RequiredDetails(true, false, null),
    city: new RequiredDetails(true, false, null),
    zip: new RequiredDetails(false, false, null),
  }
)

// Array of items
new RequiredDetails(
  true,
  true,     // ← This is an array
  () => new RequiredDetails(true, false, null)
)
```

---

## Step-by-Step Implementation

### Phase 1: Design Your Feature (No Code Yet!)

**Ask these questions:**

1. **How many steps?**
   - Single form? → Single-step setup
   - Wizard? → Multi-step setup

2. **What are the fields?**
   - List them: email, name, country, etc.

3. **What are the API calls?**
   - Create account? Search? Upload?

4. **What's conditional?**
   - Required if? Disabled if? Valid if?

### Phase 2: Create Folder Structure

```bash
# Create folders
mkdir -p app/stores/my-feature/{api-wrappers,step-1/schema}
```

### Phase 3: Create Schemas (Bottom Layer)

**Start here because it shapes everything else!**

```typescript
// app/stores/my-feature/step-1/schema/index.ts

import { z } from "zod"
import { formText, formNumber } from "~/schema/fields"
import { createRequirementsFromSchema, RequiredDetails, wrapInName } from "~/schema/utils"

// 1. Create fields using helpers
const { email } = formText("email").alwaysRequired
const { age } = formNumber("age").optionalByDefault

// 2. Combine into schema
const schemaBase = z.object({
  [email.name]: email.schema,
  [age.name]: age.schema,
})

// 3. Define requirements
const requirements = createRequirementsFromSchema(schemaBase, async (_data, _ctx) => {
  return new RequiredDetails(true, false, {
    [email.name]: email.required,
    [age.name]: age.required,
  })
})

// 4. Add validation logic
const schema = schemaBase.superRefine(async (data, ctx) => {
  await requirements(data, ctx)
})

// 5. Define defaults
const defaults = {
  [email.name]: "",
  [age.name]: NaN,
}

// 6. Export with wrapper
export const { step1 } = wrapInName("step1", {
  schema,
  fields: schemaBase.shape,
  requirements,
  defaultValues: defaults,
})
```

### Phase 4: Create API-Wrappers (Middle Layer)

```typescript
// app/stores/my-feature/api-wrappers/step-1.ts

import { apiClient } from "~/fetch/client"
import { appLogger } from "~/utils/logging"

const console = appLogger.withTag("api-wrapper").withTag("step-1")

export async function submitStep1(data: {
  email: string
  age?: number
}) {
  if (!data.email) {
    throw new Error("Email is required")
  }

  console.start(`Submitting step 1: ${data.email}`)

  try {
    const { data: result, error } = await apiClient.POST(
      "/api/v1/MyFeature/Step1",
      { body: data }
    )

    if (error) {
      console.error("Failed to submit step 1:", error)
      throw error
    }

    console.success("Step 1 submitted successfully")
    return result
  } catch (error) {
    console.error("Exception in submitStep1:", error)
    throw error
  }
}
```

### Phase 5: Create Store (Top Layer)

```typescript
// app/stores/my-feature/index.ts

import { step1 } from "./step-1/schema"
import { submitStep1 } from "./api-wrappers/step-1"
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"
import { devtools } from "zustand/middleware"

// 1. Combine defaults
const defaults = {
  ...step1.defaultValues,
  currentStep: 1,
  featureId: null,
}

// 2. Define types
type State = typeof defaults & {
  currentStep: number
  featureId: string | null
}

type Actions = {
  reset: () => void
  submitStep1: (data: any) => Promise<void>
  complete: () => Promise<string>
}

// 3. Create store
const storeName = "my-feature"

export const useMyFeature = create<State & Actions>()(
  immer(
    devtools(
      (set, get) => ({
        ...defaults,

        reset() {
          set(defaults)
        },

        async submitStep1(given) {
          // Validate with schema
          const valid = await step1.schema.parseAsync(given)
          
          // Call API-wrapper
          const result = await submitStep1(valid)
          
          // Update state
          set((state) => {
            state.email = valid.email
            state.age = valid.age
            state.featureId = result.id
            state.currentStep = 2
          })
        },

        async complete() {
          const state = get()
          if (!state.featureId) throw new Error("Feature not created")
          return state.featureId
        },
      }),
      { name: storeName }
    )
  )
)
```

### Phase 6: Use in Components

```typescript
// app/routes/my-feature/index.tsx

import { useMyFeature } from "~/stores/my-feature"

export function MyFeaturePage() {
  const { email, age, currentStep, submitStep1 } = useMyFeature()

  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      submitStep1({ email, age })
    }}>
      <input
        value={email}
        onChange={(e) => ??? }  // We need to add setters!
      />
      <button type="submit">Submit</button>
    </form>
  )
}
```

**Wait! We need setters in the store:**

```typescript
// Add to Actions
type Actions = {
  setEmail: (email: string) => void
  setAge: (age: number) => void
}

// Add to store
const useMyFeature = create((set) => ({
  // ... existing code
  
  setEmail(email: string) {
    set((state) => { state.email = email })
  },
  
  setAge(age: number) {
    set((state) => { state.age = age })
  },
}))

// Now use it
export function MyFeaturePage() {
  const { email, setEmail, age, setAge, submitStep1 } = useMyFeature()
  
  return (
    <input value={email} onChange={(e) => setEmail(e.target.value)} />
  )
}
```

---

## Configuration Options

### Option 1: Single Step (Simplest)

**Use when:** Simple form, no multi-step

```typescript
// app/stores/simple-form/index.ts

const defaults = {
  email: "",
  name: "",
}

export const useSimpleForm = create(
  immer(
    devtools((set) => ({
      ...defaults,
      
      setEmail: (email: string) => set((state) => { state.email = email }),
      setName: (name: string) => set((state) => { state.name = name }),
      
      async submit() {
        const state = ??? // How to get state?
        // ↑ Problem with (set) only!
      },
    }))
  )
)
```

**Better - use `(set, get)`:**

```typescript
export const useSimpleForm = create(
  immer(
    devtools((set, get) => ({
      ...defaults,
      
      setEmail: (email: string) => set((state) => { state.email = email }),
      setName: (name: string) => set((state) => { state.name = name }),
      
      async submit() {
        const formData = get()  // ← Now you can access state
        await submitFormAPI(formData)
        set(defaults)  // Reset
      },
    }))
  )
)
```

### Option 2: Multi-Step (Recommended for Growth)

**Use when:** Wizard, multiple pages, complex logic

```typescript
// Already shown above - this is the full pattern
```

### Option 3: Hybrid (Best of Both)

**Single store, but add conditional step logic:**

```typescript
type State = {
  // All fields from all steps
  step1Email: string
  step1Name: string
  step2Country: string
  
  // Step tracking
  currentStep: number
  completedSteps: number[]
}

type Actions = {
  // All setters
  setStep1Email: (v: string) => void
  setStep2Country: (v: string) => void
  
  // Navigation
  goToNextStep: () => Promise<void>
  goToPreviousStep: () => void
}
```

---

## Single-Step Setup

### Absolute Minimum Code

```typescript
// app/stores/form/index.ts

import { z } from "zod"
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
})

const defaults = { email: "", name: "" }

export const useForm = create(
  immer((set, get) => ({
    ...defaults,
    
    setEmail: (email: string) => set((state) => { state.email = email }),
    setName: (name: string) => set((state) => { state.name = name }),
    
    async submit() {
      const data = get()
      const valid = await schema.parseAsync(data)
      console.log("Submitting:", valid)
      // API call here
      set(defaults)
    },
  }))
)
```

### In Component

```typescript
function Form() {
  const { email, setEmail, name, setName, submit } = useForm()

  return (
    <form onSubmit={(e) => { e.preventDefault(); submit() }}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <button>Submit</button>
    </form>
  )
}
```

---

## Multi-Step Setup

### Complete Example: 2-Step Wizard

**Structure:**
```
app/stores/wizard/
├── api-wrappers/
│   ├── step-1.ts
│   └── step-2.ts
├── step-1/schema/index.ts
├── step-2/schema/index.ts
└── index.ts
```

**Step 1 Schema:**
```typescript
// app/stores/wizard/step-1/schema/index.ts

import { z } from "zod"
import { formText } from "~/schema/fields"
import { createRequirementsFromSchema, RequiredDetails, wrapInName } from "~/schema/utils"

const { email } = formText("email").alwaysRequired
const { name } = formText("name").alwaysRequired

const schemaBase = z.object({
  [email.name]: email.schema,
  [name.name]: name.schema,
})

const requirements = createRequirementsFromSchema(schemaBase, async (_data, _ctx) => {
  return new RequiredDetails(true, false, {
    [email.name]: email.required,
    [name.name]: name.required,
  })
})

const schema = schemaBase.superRefine(async (data, ctx) => {
  await requirements(data, ctx)
})

export const { step1 } = wrapInName("step1", {
  schema,
  fields: schemaBase.shape,
  requirements,
  defaultValues: {
    [email.name]: "",
    [name.name]: "",
  },
})
```

**Step 2 Schema:**
```typescript
// app/stores/wizard/step-2/schema/index.ts

import { z } from "zod"
import { formSelect } from "~/schema/fields"
import { createRequirementsFromSchema, RequiredDetails, wrapInName } from "~/schema/utils"

const { country } = formSelect("country").alwaysRequired

const schemaBase = z.object({
  [country.name]: country.schema,
})

const requirements = createRequirementsFromSchema(schemaBase, async (_data, _ctx) => {
  return new RequiredDetails(true, false, {
    [country.name]: country.required,
  })
})

const schema = schemaBase.superRefine(async (data, ctx) => {
  await requirements(data, ctx)
})

export const { step2 } = wrapInName("step2", {
  schema,
  fields: schemaBase.shape,
  requirements,
  defaultValues: {
    [country.name]: "US",
  },
})
```

**Main Store:**
```typescript
// app/stores/wizard/index.ts

import { step1 } from "./step-1/schema"
import { step2 } from "./step-2/schema"
import { submitStep1API, submitStep2API } from "./api-wrappers"
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"
import { devtools } from "zustand/middleware"

const defaults = {
  ...step1.defaultValues,
  ...step2.defaultValues,
  currentStep: 1,
  wizardId: null,
}

type State = typeof defaults & {
  currentStep: number
  wizardId: string | null
}

type Actions = {
  reset: () => void
  setField: (field: string, value: any) => void
  submitStep1: (data: any) => Promise<void>
  submitStep2: (data: any) => Promise<void>
  goToNextStep: () => void
  goToPreviousStep: () => void
  complete: () => Promise<string>
}

export const useWizard = create<State & Actions>()(
  immer(
    devtools(
      (set, get) => ({
        ...defaults,

        reset() {
          set(defaults)
        },

        setField(field: string, value: any) {
          set((state) => {
            (state as any)[field] = value
          })
        },

        async submitStep1(given) {
          const valid = await step1.schema.parseAsync(given)
          const result = await submitStep1API(valid)
          set((state) => {
            Object.assign(state, valid)
            state.wizardId = result.id
          })
        },

        async submitStep2(given) {
          const valid = await step2.schema.parseAsync(given)
          const result = await submitStep2API(valid)
          set((state) => {
            Object.assign(state, valid)
            state.wizardId = result.id
          })
        },

        goToNextStep() {
          set((state) => {
            state.currentStep = Math.min(state.currentStep + 1, 2)
          })
        },

        goToPreviousStep() {
          set((state) => {
            state.currentStep = Math.max(state.currentStep - 1, 1)
          })
        },

        complete() {
          const state = get()
          if (!state.wizardId) throw new Error("Wizard not completed")
          return Promise.resolve(state.wizardId)
        },
      }),
      { name: "wizard" }
    )
  )
)
```

**In Components:**
```typescript
// app/routes/wizard/index.tsx

import { useWizard } from "~/stores/wizard"

export function WizardPage() {
  const { currentStep, email, setField, submitStep1, goToNextStep, submitStep2, goToPreviousStep } = useWizard()

  if (currentStep === 1) {
    return (
      <form onSubmit={(e) => {
        e.preventDefault()
        submitStep1({ email })
        goToNextStep()
      }}>
        <input
          value={email}
          onChange={(e) => setField("email", e.target.value)}
          placeholder="Email"
        />
        <button>Next</button>
      </form>
    )
  }

  if (currentStep === 2) {
    return (
      <form onSubmit={(e) => {
        e.preventDefault()
        submitStep2({ country: ??? })
      }}>
        <select onChange={(e) => setField("country", e.target.value)}>
          <option>US</option>
          <option>CA</option>
        </select>
        <button onClick={goToPreviousStep}>Back</button>
        <button>Complete</button>
      </form>
    )
  }
}
```

---

## Complete Working Examples

### Example 1: Simple Contact Form (Single Step)

```typescript
// SCHEMA: app/stores/contact/schema.ts
import { z } from "zod"

export const contactSchema = z.object({
  email: z.string().email("Invalid email"),
  subject: z.string().min(5, "Subject too short"),
  message: z.string().min(10, "Message too short"),
})

export const contactDefaults = {
  email: "",
  subject: "",
  message: "",
}

// API-WRAPPER: app/stores/contact/api-wrappers.ts
import { apiClient } from "~/fetch/client"

export async function sendContactForm(data: {
  email: string
  subject: string
  message: string
}) {
  const { data: result, error } = await apiClient.POST("/api/v1/Contact", { body: data })
  if (error) throw error
  return result
}

// STORE: app/stores/contact/index.ts
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"
import { contactSchema, contactDefaults } from "./schema"
import { sendContactForm } from "./api-wrappers"

export const useContact = create(
  immer((set, get) => ({
    ...contactDefaults,
    loading: false,
    error: null,

    setEmail: (email: string) => set((state) => { state.email = email }),
    setSubject: (subject: string) => set((state) => { state.subject = subject }),
    setMessage: (message: string) => set((state) => { state.message = message }),

    async submit() {
      set((state) => { state.loading = true })
      try {
        const data = get()
        const valid = await contactSchema.parseAsync(data)
        await sendContactForm(valid)
        set({ ...contactDefaults, loading: false })
      } catch (error) {
        set((state) => { state.error = (error as Error).message; state.loading = false })
      }
    },
  }))
)

// COMPONENT: app/routes/contact/index.tsx
import { useContact } from "~/stores/contact"

export function ContactPage() {
  const { email, subject, message, loading, error, setEmail, setSubject, setMessage, submit } = useContact()

  return (
    <form onSubmit={(e) => { e.preventDefault(); submit() }}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" />
      <input value={subject} onChange={(e) => setSubject(e.target.value)} placeholder="Subject" />
      <textarea value={message} onChange={(e) => setMessage(e.target.value)} />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <button disabled={loading}>{loading ? "Sending..." : "Send"}</button>
    </form>
  )
}
```

### Example 2: Signup Wizard (Multi-Step)

```typescript
// STEP 1 SCHEMA: app/stores/signup/step-1/schema/index.ts
import { z } from "zod"
import { formText, formEmail } from "~/schema/fields"
import { createRequirementsFromSchema, RequiredDetails, wrapInName } from "~/schema/utils"

const { email } = formEmail("email").alwaysRequired
const { password } = formText("password").alwaysRequired

const schemaBase = z.object({
  [email.name]: email.schema,
  [password.name]: password.schema,
}).refine((data) => data.password.length >= 8, {
  message: "Password must be 8+ chars",
  path: ["password"],
})

export const { step1 } = wrapInName("step1", {
  schema: schemaBase,
  fields: schemaBase.shape,
  defaultValues: {
    email: "",
    password: "",
  },
})

// STEP 2 SCHEMA: app/stores/signup/step-2/schema/index.ts
import { z } from "zod"
import { formText, formSelect } from "~/schema/fields"
import { wrapInName } from "~/schema/utils"

const { firstName } = formText("firstName").alwaysRequired
const { country } = formSelect("country").alwaysRequired

const schemaBase = z.object({
  [firstName.name]: firstName.schema,
  [country.name]: country.schema,
})

export const { step2 } = wrapInName("step2", {
  schema: schemaBase,
  fields: schemaBase.shape,
  defaultValues: {
    firstName: "",
    country: "US",
  },
})

// API-WRAPPERS: app/stores/signup/api-wrappers/index.ts
import { apiClient } from "~/fetch/client"

export async function validateEmail(email: string) {
  const { data, error } = await apiClient.GET("/api/v1/Auth/validate-email", {
    params: { query: { email } }
  })
  if (error) throw error
  return data.available
}

export async function createAccount(data: { email: string; password: string; firstName: string; country: string }) {
  const { data: result, error } = await apiClient.POST("/api/v1/Auth/signup", { body: data })
  if (error) throw error
  return result
}

// STORE: app/stores/signup/index.ts
import { step1 } from "./step-1/schema"
import { step2 } from "./step-2/schema"
import { validateEmail, createAccount } from "./api-wrappers"
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"
import { devtools } from "zustand/middleware"

const defaults = {
  ...step1.defaultValues,
  ...step2.defaultValues,
  currentStep: 1,
  userId: null,
}

export const useSignup = create<any>()(
  immer(
    devtools(
      (set, get) => ({
        ...defaults,

        setField: (field: string, value: any) => set((state: any) => { state[field] = value }),

        async submitStep1(given: any) {
          const valid = await step1.schema.parseAsync(given)
          const available = await validateEmail(valid.email)
          if (!available) throw new Error("Email already exists")
          set((state: any) => {
            state.email = valid.email
            state.password = valid.password
            state.currentStep = 2
          })
        },

        async submitStep2(given: any) {
          const valid = await step2.schema.parseAsync(given)
          const state = get()
          const result = await createAccount({
            email: state.email,
            password: state.password,
            firstName: valid.firstName,
            country: valid.country,
          })
          set((state: any) => {
            state.firstName = valid.firstName
            state.country = valid.country
            state.userId = result.userId
            state.currentStep = 3
          })
        },

        reset: () => set(defaults),
      }),
      { name: "signup" }
    )
  )
)

// COMPONENT: app/routes/signup/index.tsx
import { useSignup } from "~/stores/signup"
import { useNavigate } from "react-router-dom"

export function SignupPage() {
  const navigate = useNavigate()
  const { currentStep, email, password, firstName, country, setField, submitStep1, submitStep2, userId } = useSignup()

  if (currentStep === 3) {
    return (
      <div>
        <h1>Welcome, {firstName}!</h1>
        <button onClick={() => navigate("/dashboard")}>Go to Dashboard</button>
      </div>
    )
  }

  if (currentStep === 2) {
    return (
      <form onSubmit={(e) => {
        e.preventDefault()
        submitStep2({ firstName, country })
      }}>
        <input value={firstName} onChange={(e) => setField("firstName", e.target.value)} />
        <select value={country} onChange={(e) => setField("country", e.target.value)}>
          <option>US</option><option>CA</option><option>UK</option>
        </select>
        <button>Complete</button>
      </form>
    )
  }

  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      submitStep1({ email, password })
    }}>
      <input value={email} onChange={(e) => setField("email", e.target.value)} type="email" />
      <input value={password} onChange={(e) => setField("password", e.target.value)} type="password" />
      <button>Next</button>
    </form>
  )
}
```

---

## Best Practices

### 1. **Always Start with Schemas**

❌ **Wrong Order:**
```
Code components → Add state → Add validation → Try to organize
```

✅ **Correct Order:**
```
Design schemas → Create API-wrappers → Build store → Code components
```

**Why?** Schemas are your ground truth. Everything else derives from them.

### 2. **Keep API-Wrappers Pure**

❌ **BAD:**
```typescript
export async function submitForm(data: any) {
  const store = useMyStore.getState()  // ← Don't access store!
  const result = await apiClient.POST(...)
  store.submit(result)  // ← Don't call store methods!
}
```

✅ **GOOD:**
```typescript
export async function submitForm(data: DataType) {
  return await apiClient.POST("/endpoint", { body: data })
}
// Store handles calling this and updating state
```

### 3. **Validate Early, Transform Late**

```typescript
// Bad: Validate in multiple places
const schema1 = z.object({ email: z.string().email() })
const schema2 = z.object({ email: z.string().email() })

// Good: Single schema, reuse everywhere
const emailSchema = z.string().email()
const userSchema = z.object({ email: emailSchema })
  .transform(data => ({ ...data, email: data.email.toLowerCase() }))
```

### 4. **Use Partial Data When Possible**

```typescript
// In components, fetch only what you need
const { email, setEmail } = useMyStore()  // ← Don't destructure everything

// Not:
const store = useMyStore()  // ← Causes re-render on any change
```

### 5. **Error Boundaries & Logging**

```typescript
// API-wrapper with logging
export async function submitForm(data: any) {
  console.start("Submitting form")
  try {
    const { data: result, error } = await apiClient.POST(...)
    if (error) {
      console.error("API error:", error)
      throw new Error(error.message)
    }
    console.success("Form submitted")
    return result
  } catch (error) {
    console.error("Exception:", error)
    throw error
  }
}
```

### 6. **Test Schemas Independently**

```typescript
// Easy to test because schemas are pure
describe("emailSchema", () => {
  it("should accept valid emails", () => {
    expect(emailSchema.safeParse("test@example.com").success).toBe(true)
  })
  it("should reject invalid emails", () => {
    expect(emailSchema.safeParse("invalid").success).toBe(false)
  })
})
```

### 7. **Document Your Store**

```typescript
/**
 * User signup store
 * 
 * Features:
 * - 2-step signup wizard (email/password → profile)
 * - Email validation
 * - Password strength checking
 * 
 * Usage:
 * ```
 * const { email, setEmail, submitStep1 } = useSignup()
 * ```
 */
export const useSignup = create(...)
```

### 8. **Handle Loading & Error States**

```typescript
type State = {
  data: UserType | null
  loading: boolean
  error: string | null
}

type Actions = {
  async fetch(): Promise<void>
  reset(): void
}

export const useUser = create((set) => ({
  data: null,
  loading: false,
  error: null,

  async fetch() {
    set({ loading: true, error: null })
    try {
      const data = await fetchUser()
      set({ data, loading: false })
    } catch (error) {
      set({ error: (error as Error).message, loading: false })
    }
  },

  reset() {
    set({ data: null, loading: false, error: null })
  },
}))
```

---

## Checklist: Before You Start Coding

- [ ] **Understand the feature** - What problem does it solve?
- [ ] **List all fields** - What data do you need?
- [ ] **Identify API calls** - What endpoints?
- [ ] **Plan the flow** - Single step or multi-step?
- [ ] **Define defaults** - What are initial values?
- [ ] **Decide validation** - What are the rules?
- [ ] **Plan error handling** - How to show errors?
- [ ] **Mock the store** - What will the final type look like?

## Template Files to Copy

### Minimal Single-Step Template

```typescript
// Store Template
import { z } from "zod"
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"

const schema = z.object({
  // YOUR FIELDS HERE
})

const defaults = {
  // YOUR DEFAULTS HERE
}

export const useMyFeature = create(
  immer((set, get) => ({
    ...defaults,
    loading: false,
    error: null,

    setField: (field: string, value: any) =>
      set((state) => { (state as any)[field] = value }),

    async submit() {
      set((state) => { state.loading = true })
      try {
        const data = get()
        const valid = await schema.parseAsync(data)
        // API CALL HERE
        set(defaults)
      } catch (error) {
        set((state) => {
          state.error = (error as Error).message
          state.loading = false
        })
      }
    },
  }))
)
```

---

## Quick Decision Tree

```
┌─ Single Form?
│  └─→ Use "Single-Step Setup" above
│
└─ Multi-Step Wizard?
   ├─ Complex Conditional Logic?
   │  └─→ Use "Hybrid" option with step tracking
   │
   └─ Simple Sequential Steps?
      └─→ Use "Multi-Step Setup" above
```

---

## Summary

| Component | File | Purpose |
|-----------|------|---------|
| Schemas | `step-*/schema/index.ts` | Define data structure & validation |
| API-Wrappers | `api-wrappers/*.ts` | Call endpoints, handle responses |
| Store | `index.ts` | Manage state, orchestrate actions |
| Components | `app/routes/*/index.tsx` | Display UI, handle user interaction |

**The Flow:**
```
Components render → User interacts → Call store action → 
Store validates with schema → Store calls API-wrapper → 
API-wrapper calls endpoint → Store updates state → 
Components re-render with new data
```

**Key Principle:**
```
Schemas → API-Wrappers → Store → Components
(Data)    (API calls)    (Logic)    (UI)
```

---

## Advanced: Creating Custom Field Utilities

When you need a field type that doesn't exist:

```typescript
// app/schema/fields/date.ts

import { RequiredDetails, wrapInName } from "../utils"
import { z } from "zod"

const dateType = z.coerce.date()

const alwaysRequired = dateType
  .refine((date) => date > new Date(), "Date must be in future")

const optionalByDefault = dateType.optional().default(new Date())

export const formDate = <const T extends string | null>(name: T) => ({
  alwaysRequired: wrapInName(name, {
    schema: alwaysRequired,
    required: new RequiredDetails(true, false, null),
  }),
  optionalByDefault: wrapInName(name, {
    schema: optionalByDefault,
    required: new RequiredDetails(false, false, null),
  }),
})

// Usage:
const { eventDate } = formDate("eventDate").alwaysRequired
```

---

## Debugging Tips

### 1. Check Schema Validation
```typescript
const result = await schema.safeParseAsync(formData)
console.log("Valid:", result.success)
console.log("Errors:", result.error?.errors)
```

### 2. Log Store State
```typescript
const { ...all } = useMyStore()
console.log("Current state:", useMyStore.getState())
```

### 3. Test API-Wrapper Independently
```typescript
try {
  const result = await myAPIWrapper({ test: "data" })
  console.log("Success:", result)
} catch (error) {
  console.log("Error:", error)
}
```

### 4. Use Redux DevTools Browser Extension
```typescript
// Already configured in devtools() middleware
// Open Redux DevTools in browser to see state changes
```

---

## Resources

- **Zod Documentation:** https://zod.dev
- **Zustand Documentation:** https://github.com/pmndrs/zustand
- **React Hook Form:** https://react-hook-form.com
- **React Router:** https://reactrouter.com

---

**Last Updated:** March 16, 2026

This blueprint covers 95% of real-world requirements. For edge cases, refer to the existing codebase (consignment-v2, merchandise, purchase stores).


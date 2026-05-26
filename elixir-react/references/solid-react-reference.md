# SOLID React — Clean Code Reference

> Source: *SOLID React* by Islem Maboud (solidreact.dev, 2024)
> Applies SOLID principles + clean code practices to React + TypeScript.
> Assumes working knowledge of React and TypeScript.

---

## Core Philosophy

- **Clean code** can be understood by anyone on the team, read and enhanced by developers other than its author.
- General rules: follow conventions, keep it simple (KISS), leave code cleaner than you found it (Boy Scout Rule), always find root causes.
- SOLID principles were designed for OOP but apply to React with small adaptations, leaning on **composition** instead of inheritance.
- **Nothing is absolute** — apply principles pragmatically; there are legitimate exceptions.

---

## 1. SOLID Principles

### S — Single Responsibility Principle (SRP)

**Definition:** Every component/function/module should do exactly one thing and have only one reason to change.

**Rule of thumb:**
- Break large components into smaller, focused ones.
- Extract state management and data fetching into custom hooks.
- Move API/fetching logic out of component bodies into services.
- Reusability is key.

**Bad → Good:**
```tsx
// BAD: Products fetches, filters, and renders all inline
export function Products() {
  const [products, setProducts] = useState([]);
  const [filterRate, setFilterRate] = useState(1);
  // fetchProducts, handleRating, filteredProducts useMemo... all in one component
}

// GOOD: each concern has its own home
export function Products() {
  const { products } = useProducts();           // fetching hook
  const { filterRate, handleRating } = useRateFilter(); // filter hook
  return (
    <div>
      <Filter filterRate={filterRate} handleRating={handleRating} />
      {filterProducts(products, filterRate).map(p => <Product product={p} />)}
    </div>
  );
}
```

**Advanced SRP tips:**
- Use `react-hook-form` + `zod`/`yup` instead of manual form state and validation — frees the component from those responsibilities.
- Create a `<ProfilePictureUploader />` instead of embedding file-input logic.
- Create a reusable `<InputField />` instead of repeating label + input + error markup.
- Move API calls to a `services.ts` file; call the service from the component.

---

### O — Open-Closed Principle (OCP)

**Definition:** A component should be easily extended without modifying its underlying source code.

**Rule of thumb:**
- Use props (especially `children` or render props) to inject dynamic content.
- Avoid internal `if/else` or switch logic that forces edits every time a new variant is added.
- Favor **Compound Components Pattern** for truly extensible UI components.

**Bad → Good (simple case):**
```tsx
// BAD: adding a new role forces a source code change
{role === "forward" && <ArrowRight />}
{role === "back" && <ArrowLeft />}

// GOOD: caller provides the icon; no source changes needed
export function Button({ text, icon }) {
  return <button>{text}<div>{icon}</div></button>;
}
// Usage:
<Button text="Go Home" icon={<ArrowRight />} />
```

**Advanced OCP — Compound Components Pattern:**
```tsx
// Instead of a monolithic <Dropdown items={[...]} hideIcons />,
// expose building blocks via shared context:
<Dropdown.Dropdown>
  <Dropdown.Button>Create +</Dropdown.Button>
  <Dropdown.List>
    <Dropdown.Item icon={<Icon />} description="...">New Project</Dropdown.Item>
    {/* Add a footer note without touching Dropdown source */}
    <span className="text-xs text-gray-400">All projects auto-saved</span>
  </Dropdown.List>
</Dropdown.Dropdown>
```
Each sub-component reads shared state via `useContext()` from the parent provider. New content slots in without modifying any existing component. (See: Radix, Shadcn.)

---

### L — Liskov Substitution Principle (LSP)

**Definition:** A child component should be easily swappable for the same base type without breaking the caller.

**Rule of thumb:**
- Components wrapping a native element (e.g., `<input>`) should extend its native props via `React.InputHTMLAttributes<HTMLInputElement>` and spread `...restProps`.
- Do not rename standard props; keep their original names.
- Share a common TypeScript interface between swappable component variants.
- **Note:** LSP cannot and should not always be applied — assess per case.

**Bad → Good (native element wrapper):**
```tsx
// BAD: SearchBar accepts only (value, onChange), blocking all other input props
interface SearchBarProps {
  value: string;
  onChange: (e: React.ChangeEvent) => void;
}

// GOOD: extends all native input props
interface SearchBarProps extends React.InputHTMLAttributes<HTMLInputElement> {}

export function SearchBar({ value, onChange, ...restProps }: SearchBarProps) {
  return (
    <div>
      <SearchIcon />
      <input type="search" value={value} onChange={onChange} {...restProps} />
    </div>
  );
}
```

**Advanced LSP — swappable variants:**
```tsx
// Shared interface ensures PrivacyDialog and EUPrivacyDialog are interchangeable
interface PrivacyPolicyDialogProps {
  onAccept: (id: string) => void;
  onDeny: (id: string) => void;
}

// Both dialects implement the same contract → can swap at the call site:
{!user.isEuBased && <PrivacyPolicyDialog onAccept={handleAccept} onDeny={handleDeny} />}
{user.isEuBased  && <EUPrivacyPolicyDialog onAccept={handleAccept} onDeny={handleDeny} />}
```

---

### I — Interface Segregation Principle (ISP)

**Definition:** Components should not depend on props they do not need.

**Rule of thumb:**
- Only pass the specific props a child component actually uses — not an entire parent object.
- Split multi-functional components into smaller ones, each with a focused prop interface.
- TypeScript interfaces make this enforceable.

**Bad → Good:**
```tsx
// BAD: Thumbnail only needs an image URL but receives the entire Product object
interface IThumbnailProps { product: Product; }
<img src={product.image} />

// GOOD: pass only what's needed
interface IThumbnailProps { imageUrl: string; }
<Thumbnail imageUrl={product.image} />
```

**Advanced ISP — split components instead of multi-purpose ones:**
```tsx
// BAD: one Notification component handles both project and user notifications
// with if/else branching — only one prop used at a time
const Notification = ({ project, user }: { project?: Project; user?: User }) => { ... }

// GOOD: separate components, each with its own minimal interface
const ProjectNotification = ({ project }: { project: Project }) => { ... }
const UserNotification    = ({ user }: { user: User }) => { ... }
```

---

### D — Dependency Inversion Principle (DIP)

**Definition:** Components should not directly depend on concrete implementations; they should depend on abstractions (props/interfaces).

**Rule of thumb:**
- Extract concrete logic (API calls, data transformation) out of reusable components.
- Inject behaviour via props so the component is agnostic about what runs.
- This is as simple as "use props more."

**Bad → Good:**
```tsx
// BAD: LoginForm is tightly coupled to a specific API endpoint
const handleSubmit = async (e) => {
  await axios.post("https://localhost:3000/login", { email, password });
};

// GOOD: the form receives the handler as a prop (abstraction)
export function LoginForm({ onSubmit }: { onSubmit: (email: string, password: string) => void }) {
  const internalHandleSubmit = () => onSubmit(email, password);
  // ...form JSX using internalHandleSubmit
}

// Concrete API logic lives in a connector component:
export function ConnectedLoginForm() {
  const handleSubmit = async (email, password) => {
    await axios.post("https://localhost:3000/login", { email, password });
  };
  return <LoginForm onSubmit={handleSubmit} />;
}
```

**Advanced DIP — service injection:**
```tsx
// FeedbackForm depends only on a FeedbackService interface, not a concrete endpoint
interface FeedbackFormProps { feedbackService: FeedbackService; }

export function FeedbackForm({ feedbackService }: FeedbackFormProps) {
  const handleSubmit = async (e) => {
    await feedbackService.submitFeedback(formData); // abstracted call
  };
}

// Caller decides which implementation to inject:
const feedbackServiceV2 = new FeedbackService(endpoints.FEEDBACK.v2);
<FeedbackForm feedbackService={feedbackServiceV2} />
```

---

## 2. Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Components | `PascalCase` | `ProductCard`, `UserProfile` |
| Main pages | `PascalCase` + default export | `export default function Home()` |
| Regular components | `PascalCase` + named export | `export function Button()` |
| Filenames (components) | `PascalCase` or `camelCase` | `ProductCard.tsx`, `cartDrawer.tsx` |
| Filenames (Next.js routes) | `kebab-case` | `stripe-page.tsx` |
| Functions | `camelCase`, descriptive | `calculateRectangleArea` |
| Variables | `camelCase`, descriptive, `const` by default | `filteredProductsByRate` |
| Props | `camelCase`, optional JSDoc | `filterRate`, `handleRating` |
| Custom hooks | `camelCase`, always prefixed with `use` | `useStoreInfo`, `useRateFilter` |
| TypeScript interfaces | `PascalCase`, no leading `I` | `StoreProduct` not `IProduct` |
| Constants | `UPPER_SNAKE_CASE`, always `const` | `MAX_RETRY_ATTEMPTS`, `API_BASE_URL` |

**Folder structure (recommended):**
```
src/
  assets/        # static resources
  components/    # reusable UI components (each in its own subfolder)
  hooks/         # custom hooks
  pages/         # top-level page components
  services/      # API call functions / classes
  stores/        # state management
  typings/       # TypeScript type definitions
  utils/         # helper/utility functions
```

---

## 3. Error Handling

### 3.1 Error Types
- **Syntax errors** — caught at compile time (won't run).
- **Runtime errors** — accessing `undefined`, calling non-existent functions.
- **Logic errors** — wrong behavior, no crash; hardest to catch.
- **Network errors** — API failures, connectivity issues.
- **Async errors** — unhandled promise rejections.
- **Rendering errors** — React-specific; thrown during render or lifecycle.

### 3.2 State-Based Error Handling (Simple, Always Valid)
```tsx
const [error, setError] = useState<string | null>(null);

try {
  const items = await fetchCartItems();
  setCartItems(items);
} catch (err) {
  setError((err as Error).message);
}

if (error) return <ProductsFetchingError error={error} />;
```
**Limitations:** must manually handle every possible error; no safety net if you forget.

### 3.3 Standard Error Boundaries
- Implemented as a **class component** (React 19 still requires this for native boundaries).
- Catches errors during render and lifecycle methods.
- **Does NOT catch:** event handler errors, async/API errors.

```tsx
export class StandardErrorBoundary extends React.Component<any, any> {
  state = { hasError: false, error: undefined };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error(error, errorInfo); // log to analytics service
  }

  render() {
    return this.state.hasError
      ? <ProductsFetchingError error={this.state.error?.message || ""} />
      : this.props.children;
  }
}

// Usage: wrap any subtree
<StandardErrorBoundary><Checkout /></StandardErrorBoundary>
```

### 3.4 Enhanced Error Boundaries (`react-error-boundary`)
**Preferred.** Supports function components, catches async/event errors imperatively.

```tsx
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

// Inside async code — trigger the nearest boundary imperatively:
const { showBoundary } = useErrorBoundary();
try {
  const items = await fetchCartItems();
} catch (err) {
  showBoundary(err); // passes error to the boundary's fallback
}

// Wrap the component:
<ErrorBoundary
  FallbackComponent={ProductsFetchingError}
  onError={() => console.log('Error happened!')}
>
  <Checkout />
</ErrorBoundary>

// Fallback component receives error + reset function:
export function ProductsFetchingError({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div>
      <p>{error.message}</p>
      <button onClick={resetErrorBoundary}>Retry</button>
    </div>
  );
}
```

---

## 4. Tooling for Clean Code

### 4.1 Prettier
**What:** Opinionated code formatter. Parses code → AST → reprints with consistent rules.

**Setup:** Install VSCode extension → set as default formatter → enable "Format on Save".

**Config (`.prettierrc`):**
```json
{
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### 4.2 ESLint
**What:** Pluggable linter that identifies patterns violating coding standards.

**Setup:** Run `npx eslint --init` (or framework equivalent) → answer prompts → config generated.

**Key recommended rules:**
```js
// .eslintrc.js
module.exports = {
  extends: ['next', 'prettier'],
  plugins: ['unicorn'],   // 100+ additional rules
  rules: {
    'no-unused-vars': ['error', { args: 'after-used', ignoreRestSiblings: true }],
    'prefer-const': 'error',
    'react-hooks/exhaustive-deps': 'error',
    'unicorn/filename-case': ['error', { case: 'kebabCase' }],
  },
};
```

**Run linting:**
```json
// package.json scripts
"lint": "eslint .",
"prettier": "prettier --write --ignore-unknown .",
"prettier:check": "prettier --check --ignore-unknown .",
"test": "pnpm lint && pnpm prettier:check"
```

### 4.3 Husky + lint-staged
**What:** Husky manages Git hooks; lint-staged runs linters only on staged files before commits.

**Setup:**
```bash
pnpm add --save-dev husky lint-staged
pnpm exec husky init   # creates .husky/ directory with pre-commit hook
```

**`lint-staged.config.js`:**
```js
module.exports = {
  '*.{js,jsx,ts,tsx}': ['prettier --write', 'eslint --fix', 'eslint'],
  '**/*.ts?(x)': () => 'npm run check-types',
  '*.json': ['prettier --write'],
};
```

**Effect:** every `git commit` automatically formats and lints staged files; bad code is blocked before it reaches the repo.

**Bonus:** commitlint enforces [Conventional Commits](https://www.conventionalcommits.org/) format:
```
feat(ui): add dark mode support
fix(parser): handle edge cases
feat(api)!: update API endpoint to v2   # breaking change
```

---

## 5. SOLID Benefits Summary

| Benefit | Description |
|---|---|
| **Flexibility** | More generic code that works across different components |
| **Behavioral consistency** | Wrapper components uphold the same contract as the base |
| **Testability** | Components testable in isolation via mock dependencies |
| **Decoupling** | Reduced tight coupling; changes in one place don't cascade |
| **Scalability** | Modular structure simplifies adding new features |
| **Collaboration** | Clear responsibilities let team members work in parallel |

---

## 6. Quick Principle Reference

| Principle | React Definition | Key Pattern |
|---|---|---|
| **SRP** | Component does one thing | Custom hooks, service files, small components |
| **OCP** | Extend without modifying source | Render props, `children`, Compound Components |
| **LSP** | Subtypes are swappable | Extend native HTML props, shared TS interfaces |
| **ISP** | No unused props | Pass only what's needed; split large interfaces |
| **DIP** | Depend on abstractions, not concretions | Inject logic via props; service objects |

---

## 7. Clean Code Workflow

When adding features to any existing codebase:

1. **Plan** — understand requirements and impact.
2. **Review existing code** — find the right integration point.
3. **Refactor if needed** — but only when absolutely necessary for the new feature.
4. **Implement** — write clean, simple, consistent code.
5. **Test and evaluate** — test functionality and review broader architectural impact.

**Remember:** SOLID principles are guidelines, not laws. Sometimes tight deadlines, legacy constraints, or component-specific requirements make full adherence impractical. Apply them thoughtfully.

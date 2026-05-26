# Elixir Anti-Patterns Reference

> Source: https://hexdocs.pm/elixir/what-anti-patterns.html (Elixir v1.19.5)
> Catalog originally proposed by Lucas Vegi and Marco Tulio Valente (ASERG/DCC/UFMG).

Anti-patterns (code smells) are common mistakes or indicators of problems in code. Matching an anti-pattern doesn't necessarily mean code must be rewritten — sometimes it's still the best approach for the problem at hand.

Each anti-pattern follows the structure: **Problem → Example → Refactoring** (and sometimes Additional Remarks).

---

## 1. Code-Related Anti-Patterns

### 1.1 Comments Overuse
**Problem:** Overusing comments or commenting self-explanatory code makes code *less* readable.

**Fix:** Use clear, expressive function names, variable names, and module attributes instead. Prefer first-class `@doc`/`@moduledoc` documentation over inline comments.

```elixir
# Bad
unix_now + (60 * 5)  # Add five minutes in seconds

# Good
@five_min_in_seconds 60 * 5
unix_now + @five_min_in_seconds
```

---

### 1.2 Complex `else` Clauses in `with`
**Problem:** Flattening all error handling into a single `else` block in a `with` expression makes it hard to know which clause the error came from.

**Fix:** Normalize return types in private helper functions, so each step returns consistent `{:ok, _}` / `{:error, _}` tuples and the `with` block only handles the happy path.

```elixir
# Bad
with {:ok, encoded} <- File.read(path),
     {:ok, decoded} <- Base.decode64(encoded) do
  {:ok, String.trim(decoded)}
else
  {:error, _} -> {:error, :badfile}
  :error -> {:error, :badencoding}
end

# Good — normalize errors inside private functions
defp file_read(path) do
  case File.read(path) do
    {:ok, c} -> {:ok, c}
    {:error, _} -> {:error, :badfile}
  end
end
```

---

### 1.3 Complex Extractions in Clauses
**Problem:** When multi-clause functions extract many fields across arguments/clauses, it becomes unclear which extractions are for pattern matching/guards and which are for the function body.

**Fix:** Extract only pattern/guard-related variables in the function head; extract body-only variables inside the function body.

```elixir
# Prefer
def drive(%User{age: age} = user) when age >= 18 do
  %User{name: name} = user
  "#{name} can drive"
end
```

---

### 1.4 Dynamic Atom Creation
**Problem:** Atoms are not garbage collected and the BEAM has a limit of ~1,048,576 atoms. Dynamically creating atoms from untrusted external input (e.g., `String.to_atom/1`) can exhaust this limit and crash the system.

**Fix:**
- Use an explicit allow-list mapping strings to known atoms.
- Or use `String.to_existing_atom/1`, which only converts if the atom is already known to the runtime. Ensure the valid atoms are referenced somewhere in the same module at runtime (e.g., in a function body or list), not only at compile time.

```elixir
# Risky
%{status: String.to_atom(status)}

# Safe — explicit mapping
defp convert_status("ok"), do: :ok
defp convert_status("error"), do: :error

# Safe — existing atoms only
%{status: String.to_existing_atom(status)}
```

---

### 1.5 Long Parameter List
**Problem:** Functions with many parameters are confusing and error-prone to call.

**Fix:** Group related arguments into maps, structs, or keyword lists (for optional args), reducing arity and improving clarity.

```elixir
# Bad
def loan(user_name, email, password, alias, book_title, book_ed), do: ...

# Good
def loan(%{name: _, email: _, password: _, alias: _} = user, %{title: _, ed: _} = book), do: ...
```

---

### 1.6 Namespace Trespassing
**Problem:** A library defining modules outside its own namespace can conflict with other libraries or future versions of the same library. The BEAM can only load one instance of a module at a time.

**Fix:** Always prefix all module names with the library name.

```elixir
# Bad (in :plug_auth package)
defmodule Plug.Auth, do: ...

# Good
defmodule PlugAuth, do: ...
```

**Exceptions:** Protocol implementations (by design), Mix tasks under `Mix.Tasks`, and cases where you maintain both libraries.

---

### 1.7 Non-Assertive Map Access
**Problem:** Using `map[:key]` (dynamic/optional access) for keys that are expected to always exist causes silent `nil` propagation instead of early failures.

**Fix:** Use `map.key` (static notation) for required keys; use `map[:key]` only for truly optional keys.

| Notation | Key exists | Key missing | Use when |
|---|---|---|---|
| `map.key` | Returns value | Raises `KeyError` | Required atom keys (structs, known maps) |
| `map[:key]` | Returns value | Returns `nil` | Optional or dynamic keys |

Pattern matching and structs (`@enforce_keys`) are also strong alternatives for required fields.

---

### 1.8 Non-Assertive Pattern Matching
**Problem:** Defensive or imprecise code that silently returns incorrect values instead of crashing on unexpected input. This hides bugs and violates the "fail fast" philosophy.

**Fix:** Use pattern matching assertively so unexpected input raises immediately. Let the supervisor handle restarts.

```elixir
# Bad — silently returns wrong value on malformed input
key_value = String.split(pair, "=")
Enum.at(key_value, 0) == key && Enum.at(key_value, 1)

# Good — crashes loudly on unexpected format
[key, value] = String.split(pair, "=")
```

Also avoid catching all cases with `_` in `case`:
```elixir
# Risky — may hide future return values
case f(arg) do
  {:ok, v} -> ...
  _ -> ...   # avoid
end

# Better
case f(arg) do
  {:ok, v} -> ...
  {:error, _} -> ...
end
```

---

### 1.9 Non-Assertive Truthiness
**Problem:** Using `&&`, `||`, `!` (truthy operators) when all operands are known booleans is overly permissive and may accidentally treat Erlang values like `:undefined` or `:error` as truthy.

**Fix:** Use strict boolean operators `and`, `or`, `not` when all operands are guaranteed booleans, especially when interfacing with Erlang APIs.

```elixir
# Risky
if is_binary(name) && is_integer(age), do: ...

# Better
if is_binary(name) and is_integer(age), do: ...
```

---

### 1.10 Structs with 32 Fields or More
**Problem:** The BEAM uses flat maps (two tuples: one for keys, one for values) for maps with ≤31 keys, enabling key-sharing optimizations. At 32+ keys, it switches to a hash map, losing these optimizations and increasing memory usage.

**Fix:** Keep structs under 32 fields by:
- Grouping optional/nil fields under a `:metadata` or `:optionals` nested field.
- Nesting related fields into sub-structs.
- Grouping always-accessed-together fields into tuples.

---

## 2. Design-Related Anti-Patterns

### 2.1 Alternative Return Types
**Problem:** Functions that accept an option which drastically changes their return type are hard to use correctly, since options are often set dynamically.

**Fix:** Create a separate, clearly-named function for each return type instead of using a flag option.

```elixir
# Bad
def parse(string, options \\ [])  # returns integer | {integer, string} | :error

# Good
def parse(string)              # always {integer, string} | :error
def parse_discard_rest(string) # always integer | :error
```

---

### 2.2 Boolean Obsession
**Problem:** Using multiple boolean flags (e.g., `admin: true`, `editor: true`) for overlapping or related states is confusing and harder to extend.

**Fix:** Replace multiple booleans with a single atom-based value (e.g., `role: :admin`). Atoms are internally booleans with no performance penalty.

```elixir
# Bad
options[:admin] / options[:editor]

# Good
case Keyword.get(options, :role, :default) do
  :admin -> ...
  :editor -> ...
  :default -> ...
end
```

---

### 2.3 Exceptions for Control-Flow
**Problem:** Using `try/rescue` for expected errors (like missing files) forces callers to use exception handling even when the error is not exceptional.

**Fix:** Library authors should provide a non-raising version returning `{:ok, result}` / `{:error, reason}`, and optionally a `!` bang version that raises. Let callers decide if an error is exceptional.

```elixir
# Bad
try do
  IO.puts(File.read!(file))
rescue
  e -> IO.puts(:stderr, Exception.message(e))
end

# Good
case File.read(file) do
  {:ok, binary} -> IO.puts(binary)
  {:error, reason} -> IO.puts(:stderr, "could not read file: #{reason}")
end
```

**Exceptions where raising is fine:** invalid argument types, scripts/tests (use `!` versions), framework-controlled error handling (e.g., Phoenix).

---

### 2.4 Primitive Obsession
**Problem:** Using basic types (strings, integers, floats) to represent complex domain concepts leads to scattered ad-hoc manipulation and weak guarantees.

**Fix:** Create domain-specific structs or composite types and parse raw input into them at the boundary.

```elixir
# Bad
def extract_postal_code(address) when is_binary(address), do: ...

# Good
defmodule Address do
  defstruct [:street, :city, :state, :postal_code, :country]
end

def extract_postal_code(%Address{} = address), do: ...
```

Also applies to using floats for money — use a richer library like `ex_money`.

---

### 2.5 Unrelated Multi-Clause Function
**Problem:** Mixing completely unrelated business logic into one multi-clause function creates a function that is hard to document, understand, and maintain.

**Fix:** Split into separate, well-named functions (or modules). Multi-clause functions should only be used when clauses handle variations of the *same* concept.

```elixir
# Bad
def update(%Product{} = p), do: ...
def update(%Animal{} = a), do: ...  # Completely different behavior

# Good
def update_product(%Product{} = p), do: ...
def update_animal(%Animal{} = a), do: ...
```

---

### 2.6 Using Application Configuration for Libraries
**Problem:** `Application.fetch_env!/2` is a global singleton — only one value per key per application. If a library reads config this way, multiple apps depending on it cannot configure the library differently.

**Fix:** Accept configuration as function arguments (keyword lists or options), not from the application environment. Let users read from their own environment if needed.

```elixir
# Bad — library reads global config
parts = Application.fetch_env!(:app_config, :parts)

# Good — accept as parameter with default
def split(string, opts \\ []) do
  parts = Keyword.get(opts, :parts, 2)
  String.split(string, "-", parts: parts)
end
```

**For supervision trees:** Provide a child spec so users can start the process under their own supervisor with their own config.

**For Mix tasks:** Read per-project config from `Mix.Project.config/0` or accept CLI flags via `OptionParser`.

---

## 3. Process-Related Anti-Patterns

### 3.1 Code Organization by Process
**Problem:** Using a `GenServer` (or any process) purely for code organization — not for concurrency, shared state, or error isolation — creates an unnecessary bottleneck.

**Fix:** Organize code using plain modules and functions. Only introduce processes when you actually need runtime properties (state, concurrency, fault isolation).

```elixir
# Bad — GenServer wrapping pure functions
def add(a, b, pid), do: GenServer.call(pid, {:add, a, b})

# Good
def add(a, b), do: a + b
```

---

### 3.2 Scattered Process Interfaces
**Problem:** Spreading direct access to a process (Agent, GenServer) across many modules makes maintenance harder, increases the risk of bugs, and allows any data format to be used.

**Fix:** Centralize all interaction with a process abstraction in a single module. Expose a clean API; all other modules go through that API.

```elixir
# Good — centralized interface
defmodule Foo.Bucket do
  use Agent
  def start_link(_opts), do: Agent.start_link(fn -> %{} end)
  def get(bucket, key), do: Agent.get(bucket, &Map.get(&1, key))
  def put(bucket, key, value), do: Agent.update(bucket, &Map.put(&1, key, value))
end
```

---

### 3.3 Sending Unnecessary Data
**Problem:** Messages sent to processes (via `send/2`, `GenServer.call/3`, `spawn/1`, `Task.async/1`, etc.) are fully copied to the receiving process. Sending entire large structs when only a small piece is needed wastes CPU and memory.

**Subtle case:** Even `spawn(fn -> log(conn.remote_ip) end)` copies the whole `conn` because the closure captures the variable.

**Fix:** Extract only the needed data *before* passing it to the process.

```elixir
# Bad — copies all of conn
spawn(fn -> log_request_ip(conn) end)

# Good — copies only the IP
ip_address = conn.remote_ip
spawn(fn -> log_request_ip(ip_address) end)
```

---

### 3.4 Unsupervised Processes
**Problem:** Starting long-running processes outside a supervision tree makes their lifecycle uncontrolled — no guaranteed start order, no clean shutdown, no automatic restart on failure.

**Fix:** Always start processes inside a supervision tree. This also enables introspection tools like Observer and LiveDashboard.

```elixir
# Good
children = [
  Counter,
  Supervisor.child_spec({Counter, name: :other_counter, initial_value: 15}, id: :other_counter)
]
Supervisor.start_link(children, strategy: :one_for_one)
```

---

## 4. Meta-Programming Anti-Patterns

### 4.1 Compile-Time Dependencies (Unnecessary)
**Problem:** Using a module as an argument to a macro at compile-time makes it a compile-time dependency of the calling module, even if it's only needed at runtime. This inflates the recompilation graph.

**Fix:** Use `Macro.expand_literals/2` with the target function context to ensure the module reference is treated as a runtime dependency.

```elixir
defmacro plug(mod) do
  mod = Macro.expand_literals(mod, %{__CALLER__ | function: {:call, 2}})
  quote do: @plugs unquote(mod)
end
```

Use `mix xref trace path/to/file.ex` to inspect compile-time vs runtime dependencies.

---

### 4.2 Large Code Generation
**Problem:** Macros that expand to large amounts of code inside `quote` are compiled on every invocation, increasing compilation time and artifact size.

**Fix:** Delegate most work to a plain function called from the macro; keep the `quote` block minimal.

```elixir
# Bad — all logic inside quote
defmacro get(route, handler) do
  quote do
    if not is_binary(unquote(route)), do: raise ...
    @store_route_for_compilation {unquote(route), unquote(handler)}
  end
end

# Good — delegate to function
defmacro get(route, handler) do
  quote do: Routes.__define__(__MODULE__, unquote(route), unquote(handler))
end

def __define__(module, route, handler) do
  if not is_binary(route), do: raise ...
  Module.put_attribute(module, :store_route_for_compilation, {route, handler})
end
```

---

### 4.3 Unnecessary Macros
**Problem:** Using a macro when a plain function would suffice adds complexity, requires `require`, and makes code harder to read and maintain.

**Fix:** Default to functions. Only use macros when compile-time code transformation is genuinely needed (e.g., AST manipulation, code generation).

```elixir
# Bad
defmacro sum(v1, v2) do
  quote do: unquote(v1) + unquote(v2)
end

# Good
def sum(v1, v2), do: v1 + v2
```

---

### 4.4 `use` Instead of `import`
**Problem:** `use/1` injects arbitrary code (functions, imports, module attributes) from an external module, including transitive dependencies. This makes it hard to understand what's available in your module without reading the `__using__/1` implementation.

**Fix:** Prefer `import` or `alias` when all you need is to call functions from another module. Reserve `use` for when code injection is truly necessary.

When `use` is necessary, library authors should document it with a "Nutrition facts" `@moduledoc` admonition listing all public-API changes injected.

```elixir
# Bad — imports ModuleA transitively, causing conflicts
use Library

# Good — explicit and predictable
import Library
```

---

### 4.5 Untracked Compile-Time Dependencies
**Problem:** Building module names dynamically at compile-time (e.g., `Module.concat/2` or atom literals like `:"Elixir.Foo"`) prevents the compiler from tracking them as dependencies, leading to stale compiled artifacts after changes.

**Fix:** Use full module name literals directly. If dynamic dispatch is truly needed, use macros that expand the full alias at compile-time via `OtherModule.unquote(part)`.

```elixir
# Bad — compiler can't track OtherModule.Foo/Bar
for part <- [:Foo, :Bar], do: Module.concat(OtherModule, part).example()

# Good — full aliases are visible to the compiler
mods = [OtherModule.Foo, OtherModule.Bar]
for mod <- mods, do: mod.example()
```

Use `mix xref trace path/to/file.ex` to verify dependency tracking.

---

## Quick Reference Table

| # | Anti-Pattern | Category | Core Fix |
|---|---|---|---|
| 1.1 | Comments overuse | Code | Use expressive names; use `@doc`/`@moduledoc` |
| 1.2 | Complex `else` in `with` | Code | Normalize errors in private helpers |
| 1.3 | Complex extractions in clauses | Code | Extract pattern vars in head; body vars in body |
| 1.4 | Dynamic atom creation | Code | Allow-list or `String.to_existing_atom/1` |
| 1.5 | Long parameter list | Code | Group related args into maps/structs |
| 1.6 | Namespace trespassing | Code | Always use library name as module prefix |
| 1.7 | Non-assertive map access | Code | `map.key` for required, `map[:key]` for optional |
| 1.8 | Non-assertive pattern matching | Code | Match explicitly; crash on unexpected input |
| 1.9 | Non-assertive truthiness | Code | Use `and`/`or`/`not` for boolean operands |
| 1.10 | Structs with 32+ fields | Code | Keep under 32; nest optional fields |
| 2.1 | Alternative return types | Design | Separate function per return shape |
| 2.2 | Boolean obsession | Design | Use atoms instead of multiple booleans |
| 2.3 | Exceptions for control-flow | Design | Return `{:ok,_}/{:error,_}`; let callers decide |
| 2.4 | Primitive obsession | Design | Model domain with structs/composite types |
| 2.5 | Unrelated multi-clause function | Design | Split into separate named functions |
| 2.6 | App config for libraries | Design | Accept config as function args |
| 3.1 | Code organization by process | Process | Use modules/functions; process only for runtime needs |
| 3.2 | Scattered process interfaces | Process | Centralize all process interaction in one module |
| 3.3 | Sending unnecessary data | Process | Extract only needed data before sending |
| 3.4 | Unsupervised processes | Process | Always start under a supervision tree |
| 4.1 | Unnecessary compile-time deps | Meta | Use `Macro.expand_literals/2` for runtime modules |
| 4.2 | Large code generation | Meta | Delegate work to functions, keep `quote` thin |
| 4.3 | Unnecessary macros | Meta | Default to functions; macros only when necessary |
| 4.4 | `use` instead of `import` | Meta | Prefer `import`/`alias`; document `use` effects |
| 4.5 | Untracked compile-time deps | Meta | Use full module literals; avoid `Module.concat` |

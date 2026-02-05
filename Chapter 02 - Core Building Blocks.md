# Chapter 02 - Core Building Blocks

Application logic is organized into three primary types of components, each with a focused responsibility within a business operation.

## Commands

Commands handle write-side behavior only: creating, updating, or deleting data. A command is created for the purpose of changing state, not reading it.

A command represents one atomic business operation, such as:

- Creating a post
- Publishing content
- Sending a notification

Inside a command, you may modify multiple related domain entities when doing so logically belongs to a single operation.

A command may return the created or modified entity for optimization and code simplicity. This is especially useful for atomic operations where read-after-write is inherent (e.g., incrementing a counter and returning the new value). However, a command should not fetch additional data purely for presentation — eager loading relations for display belong in a Query.

A command may also dispatch domain events as side effects. These are acceptable outcomes of write operations.


## Queries

Queries are used exclusively for reading data and have no side effects.

They may:

- Perform complex SQL joins
- Use eager loading
- Aggregate data from multiple sources
- Filter and transform datasets for a specific scenario

A query returns data in whatever shape the caller needs: Eloquent models, collections, arrays, or DTOs.

Queries express what data should be retrieved for a particular use case, not the internal implementation details.

### Request-scoped memoization

Repeated access to the same read-only data within a single execution may be handled via request-scoped memoization at the Query level. This is useful when the same Query is called multiple times during a single request (e.g., fetching a configuration value for each item in a batch).

Rules:
- Memoization is an execution-level optimization and must not alter observable behavior.
- Request-scoped memoization must not be used for data that may change during execution.
- Prefer explicit data passing through Actions when the orchestrator controls the flow.
- Use memoization when the same Query is called from disconnected parts of the request.


## Actions

Actions encapsulate business logic and orchestrate flows that do not fit neatly into a single Command or Query. An Action may contain its own calculations and business rules, coordinate Commands and Queries, or do both — the mix depends on the use case.

Actions are used when a business scenario:

- Requires calculations, transformations, or business rule enforcement
- Spans multiple steps
- Coordinates several commands and queries
- Includes conditional logic or branching flows

Typical examples:

- Create and publish post with notifications
- User registration with welcome email
- Multi-step workflows with business rules
- Application-level data transformations or pipelines
- Calculate time for reading a post

An Action always executes as a single, complete operation. It may be a complex "operation" that contains other Actions, Commands, Queries, and its own business logic, but it should remain a clear logical unit. Avoid completely switching behavior via parameter flags.

An Action may read or write data directly — via Eloquent, HTTP clients, external APIs, etc. — without delegating to Commands or Queries. This is acceptable when:

- The data access is not the sole purpose of the Action (otherwise it would be a Command or Query)
- The operation is simple, without complex syntax or logic
- The operation does not require reuse elsewhere

This keeps Actions pragmatic: no need to create a Command or Query for every simple database call within a larger workflow.

### Middleware (Optional)

Commands, Queries, and Actions can use middleware for logging, auth checks, and other tasks.

### When to use Actions vs Commands and Queries

Use a **Command/Query** when:
- A single write operation (create, update, delete) → Command
- A single read operation (get, find, etc.) → Query
- Transaction wrapping is required
- The state change is isolated
- The logic should be reusable across multiple Actions

Use an **Action** when:
- Multiple Commands/Queries must run in sequence
- The workflow contains complex business logic or conditional branches
- Several operations must be coordinated together
- The logic is called from controllers or jobs

Do not wrap a single Command/Query in an Action.
If the operation is simple, use the Command/Query directly.


## Operating Level

Commands, Queries, and Actions typically operate at the level of business entities and use cases.

A business entity often spans multiple tables or represents a meaningful concept in the domain: an Order with its Items, a User with their Profile, a Subscription with its billing history. These components are a natural fit for such cases.

However, they can also be used for single-table operations when there is meaningful logic to encapsulate — complex filtering, validation rules, or conditional behavior that deserves explicit boundaries.

**Skip** Commands/Queries/Actions when:
- Simple CRUD with no business rules → use Eloquent directly
- Basic model lookups → `User::find($id)` is sufficient
- Trivial attribute updates → `$post->update(['status' => 'draft'])`

**Consider** Commands/Queries/Actions when:
- Operations involve business rules or validation
- Multi-model coordination is required (Order + OrderItems + Payment)
- Complex conditions or filtering logic needs encapsulation
- The use case might be called from multiple entry points
- The logic deserves a name and explicit boundaries

This keeps the architecture lightweight: Eloquent handles simple data access, while Commands/Queries/Actions handle operations with meaningful logic.

## Supporting Constructs (Optional)

These constructs are not core building blocks of the architecture.
They exist to improve correctness, reuse, or maintainability when needed.


## Value Objects (Optional)

Value objects elevate primitive scalar values into meaningful domain concepts with guaranteed invariants.

Value objects are entirely optional. Use them when primitive obsession causes bugs or confusion — for example, when an email string is passed unchecked through multiple layers, or when a slug format must be guaranteed. If primitives work well enough, there is no need to introduce value objects.

Example: Email

Instead of passing unvalidated strings throughout the system, a value object:

- Normalizes input (for example, trim, lowercase)
- Validates on creation
- Guarantees correctness by type

A value object is either valid or does not exist at all.

Principles:

- Fail fast
- Trust the type system
- Validation stops at domain boundaries

Value objects do not carry heavy business logic. Their role is correctness and expressiveness that primitives lack.

## Query Builder Modifiers (Optional)

A query builder modifier is a callable class that takes a query builder and returns a modified instance with certain filtering logic applied.

This is a lower-level construct focused on direct SQL composition rather than high-level use case semantics.

**Use sparingly.** Modifiers are justified only when the same SQL fragment is genuinely duplicated. Do not create modifiers preemptively or as a default pattern — inline queries are often clearer for one-off use.

They allow developers to:

- Reuse SQL fragments
- Avoid duplication
- Assemble complex queries from small, readable parts

In Laravel, for example, modifiers integrate cleanly into builder chains using the `tap()` helper without breaking fluency.

They are particularly useful for:

- Catalog and storefront filtering
- Search and aggregation flows
- Dynamic query composition independent of models

### When to create a modifier

- The same join and where are used more than once.
- SQL logic starts duplicating.
- You need to apply the same restriction to different attributes.
- You want to split part of a query for better control and easier future modifications.

### Compared to Model-Bound Query Builders

Some frameworks allow attaching custom filtering logic directly to a model's query builder — centralizing all query customizations in a single class bound to that model.

This works well for simple cases. As the model grows in complexity, that class tends to accumulate unrelated methods from different use cases, mixing concerns that belong apart.

Query Builder Modifiers take the opposite approach: each modifier is a focused, independent callable that applies one specific SQL fragment. Modifiers are not bound to any model and can be composed across different builders, raw queries, or query objects from multiple sources.

The distinction in practice:
- **Model-bound builder** — all query logic for a model lives in one class; easy to discover by model, but grows in scope over time
- **Modifier** — one responsibility per class; composable across different query contexts, independent of model hierarchy

In Laravel, for example, a custom `Builder` extension lives on the model and is implicitly available on all that model's queries. A modifier is a standalone class applied explicitly — by deliberate choice, not by default.

Use model-bound builders for model-specific convenience. Use modifiers when the same SQL logic must be applied across different query contexts, or when isolation and explicit composition matter more than proximity to the model.

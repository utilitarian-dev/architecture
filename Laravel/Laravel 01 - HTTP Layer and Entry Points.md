# Laravel 01 - HTTP Layer and Entry Points

In Laravel applications, the HTTP layer offers several permissible entry points: built-in routing methods, controllers, and route closures.

The choice depends on:

- Complexity of the scenario
- The need to interact with HTTP context
- How strictly business logic is isolated from transport concerns

A foundational rule remains consistent:

The HTTP layer is only responsible for receiving a request and returning a response. Business logic must live outside of it.

## Static content and navigation

`Route::view()`, `Route::redirect()`, `return view()`

These approaches are suitable only for pages that have:

- No user input handling
- No database interactions
- No conditional or computational logic

Use cases: landing pages, "About Us", privacy policy, terms of use.

Key benefits:

- No meaningless controllers
- Minimal boilerplate
- Reduced cognitive load
- Clean, readable route definitions

## Controllers

A controller acts as a transport coordination mechanism. It should not contain business logic.

A controller is appropriate when:

- Orchestrating HTTP request and response (form requests, authorization, status codes, headers, cookies)
- Transforming data (Blade views, API resources, DTO formatting)
- Selecting execution flows based on input (conditional branching based on request content)
- Accessing data added by middleware

Controllers always delegate business logic to actions, commands, or other application-level objects.

### Controller responsibility boundary

A controller may decide *which* Action to call based on request data — this is transport coordination. A controller must not decide *how* the Action should behave internally — that is business logic.

```php
// Acceptable: choosing which action to call
public function store(Request $request)
{
    if ($request->boolean('publish')) {
        $action = CreateAndPublishPostAction::make(...);
    } else {
        $action = CreatePostAction::make(...);
    }
    return $action->dispatch();
}

// Not acceptable: business logic in controller
public function store(Request $request)
{
    $discount = $request->user()->isPremium() ? 0.2 : 0;  // Business rule
    $finalPrice = $this->calculatePrice($request->items, $discount);  // Business logic
    // ...
}
```

If the controller needs to compute values, apply business rules, or make domain-level decisions, that logic belongs in an Action.

## Closures

Closures inside routes are acceptable in two distinct roles and should not be mixed.

### Closures with inline logic (limited use)

Valid only when:

- Logic is trivial and flat
- No branching or growing complexity expected
- The code will remain permanently small

Example: health checks, simple redirects, temporary system endpoints.

### Closures as entry points (recommended)

A closure may serve solely as a lightweight endpoint entry. It delegates execution entirely to the application layer.

In this case:

- The closure contains zero business rules
- It acts similarly to a thin controller
- The action owns the use case execution



## Direct action invocation

Route to action is acceptable when a use case is:

- Single-step
- Requires minimal HTTP processing
- Operates solely on route parameters and returns a result

Benefits:

- Minimal infrastructure code
- Highly testable
- Business logic isolated from HTTP
- Reuse from CLI, jobs, and tests

## Entry point selection guidance

| Scenario | Recommended entry point |
| --- | --- |
| Static page | `Route::view` |
| Redirect or navigation | `Route::redirect` |
| HTTP orchestration and formatting | Controller |
| Atomic use case | Closure to Action |
| Temporary trivial logic | Inline closure (explicitly justified) |

## Foundational principle

Closures and controllers are transport adapters, not business containers. Action objects are where the business meaning lives.

Invokable syntax is optional, not a requirement.

For routing rules and URL design principles, see Laravel 02 - Routing Rules.

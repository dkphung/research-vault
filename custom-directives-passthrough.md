---
tags: [graphql]
date: 2024-12-22
status: complete
---

# Custom Directives Pass-Through in Cosmo Gateway

**Research Date:** 2025-11-17
**Context:** Investigating whether custom directives (like `@constraint`) can pass through Cosmo Gateway composition into the supergraph schema.

## Summary

**TL;DR:** Cosmo Router **does NOT preserve custom directives** in the composed supergraph schema.

### Key Distinction

- **Apollo Federation 2.1+ Spec**: ✅ Supports `@composeDirective` for preserving custom directives
- **Apollo Router**: ✅ Implements `@composeDirective` (works as documented)
- **Cosmo Router**: ❌ Does NOT implement `@composeDirective` (claims Federation v2 support but missing this feature)

**This research is specific to Cosmo Router.** If using Apollo Router, custom directives CAN be preserved using `@composeDirective` as described in [Apollo's documentation](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives#managing-custom-directives).

## Current State

### Subgraph Schema (schemas/comments.graphql)

The comments subgraph defines a `@constraint` directive for input validation:

```graphql
directive @constraint(
  # String constraints
  minLength: Int
  maxLength: Int
  pattern: String
  format: String
  # Number constraints
  min: Float
  max: Float
  # ... etc
) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION

input CreateCommentInput {
  id: ID! @constraint(format: "uuid")
  targetType: TargetType!
  targetId: String! @constraint(minLength: 1)
  content: String! @constraint(minLength: 1, maxLength: 10000)
  parentCommentId: ID @constraint(format: "uuid")
  annotation: AnnotationInput!
}
```

### Composed Schema (router.json)

After running `pnpm compose`, the composed schema **strips the directive**:

```graphql
input CreateCommentInput {
  id: ID!
  targetType: TargetType!
  targetId: String!
  content: String!
  parentCommentId: ID
  annotation: AnnotationInput!
}
```

**Observation:** The `@constraint` directive definition and all usage instances are removed.

### Directives That ARE Preserved

Only specific directives appear in the composed schema:

```graphql
directive @authenticated on ENUM | FIELD_DEFINITION | INTERFACE | OBJECT | SCALAR
directive @inaccessible on ARGUMENT_DEFINITION | ENUM | ENUM_VALUE | FIELD_DEFINITION | INPUT_FIELD_DEFINITION | INPUT_OBJECT | INTERFACE | OBJECT | SCALAR | UNION
directive @requiresScopes(scopes: [[openfed__Scope!]!]!) on ENUM | FIELD_DEFINITION | INTERFACE | OBJECT | SCALAR
directive @tag(name: String!) repeatable on ARGUMENT_DEFINITION | ENUM | ENUM_VALUE | FIELD_DEFINITION | INPUT_FIELD_DEFINITION | INPUT_OBJECT | INTERFACE | OBJECT | SCALAR | UNION
```

These are **Cosmo-specific authorization directives** and standard Federation directives.

## Federation 2.1+ @composeDirective (Apollo Router Only)

Apollo Federation 2.1+ introduced `@composeDirective` to preserve custom directives during composition.

### How It Works (Apollo Router)

According to [Apollo's documentation](https://www.apollographql.com/docs/federation/federated-types/federated-directives), `@composeDirective` "indicates to composition that all uses of a particular custom type system directive in the subgraph schema should be preserved in the supergraph schema."

**Complete Example:**

```graphql
extend schema
  @link(url: "https://specs.apollo.dev/link/v1.0")
  @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@composeDirective"])
  @link(url: "https://specs.dev/constraint/v1.0", import: ["@constraint"])
  @composeDirective(name: "@constraint")

directive @constraint(
  minLength: Int
  maxLength: Int
  pattern: String
  format: String
  min: Float
  max: Float
) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION

input CreateCommentInput {
  id: ID! @constraint(format: "uuid")
  content: String! @constraint(minLength: 1, maxLength: 10000)
}
```

**Requirements (Apollo Router):**
1. Must use Federation v2.0+
2. Directive must be imported from a spec URL via `@link`
3. Must use `@composeDirective(name: "@directiveName")` on the schema
4. The directive name in `@composeDirective` must match exactly

**Result (Apollo Router):** ✅ The `@constraint` directive definition AND all usages are preserved in the supergraph schema.

**Purpose:** Tells Apollo's composition engine to include `@constraint` in the supergraph schema.

### Testing with Cosmo

I attempted to add `@composeDirective` to the comments schema and tested comprehensively:

#### Step 1: Update Schema

```graphql
extend schema
  @link(
    url: "https://specs.apollo.dev/federation/v2.8"
    import: ["@key", "@shareable", "@inaccessible", "@override", "@external", "@provides", "@requires", "@composeDirective"]
  )
  @composeDirective(name: "@constraint")
  @link(url: "https://specs.dev/constraint/v1.0", import: ["@constraint"])

directive @constraint(...) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION

input CreateCommentInput {
  id: ID! @constraint(format: "uuid")
  content: String! @constraint(minLength: 1, maxLength: 10000)
  # ...
}
```

#### Step 2: Recompose

```bash
pnpm compose:check  # ✅ No errors
pnpm compose        # ✅ Completed successfully
```

**Result:** The `@constraint` directive was **NOT** in `router.json`:

```bash
$ jq -r '.engineConfig.graphqlSchema' router.json | grep "directive @constraint"
# No output - directive not found
```

#### Step 3: Restart Router and Test Live Introspection

```bash
docker-compose restart router  # Restart with new router.json
```

Query the live router's introspection endpoint:

```bash
curl -X POST http://localhost:3001/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { directives { name } } }"}'
```

**Result:** Only these directives are available:

```json
{
  "data": {
    "__schema": {
      "directives": [
        { "name": "authenticated" },
        { "name": "inaccessible" },
        { "name": "requiresScopes" },
        { "name": "tag" },
        { "name": "include" },
        { "name": "skip" },
        { "name": "deprecated" },
        { "name": "specifiedBy" },
        { "name": "oneOf" }
      ]
    }
  }
}
```

**`@constraint` is NOT in the list.**

#### Step 4: Check Input Type

Queried the `CreateCommentInput` type - no directive metadata is preserved on fields.

**Conclusion:** Even after:
1. ✅ Adding `@composeDirective` to the schema
2. ✅ Successfully composing without errors
3. ✅ Restarting the router with the new `router.json`
4. ✅ Querying the live introspection endpoint

**The `@constraint` directive is completely stripped from the supergraph.**

Cosmo Router does **NOT** support `@composeDirective` or custom directive preservation.

## Why Custom Directives Are Removed

### Federation Composition Philosophy

1. **Type System Directives** (like `@constraint`) provide metadata about the schema structure
2. **Executable Directives** (like `@skip`, `@include`) affect query execution
3. By default, composition only preserves:
   - Federation directives (`@key`, `@shareable`, etc.)
   - Built-in GraphQL directives (`@deprecated`)
   - Framework-specific directives (Cosmo's `@authenticated`, `@requiresScopes`)

### Cosmo's Behavior

Cosmo follows the default composition behavior and does **not** implement `@composeDirective` support. Custom directives are treated as subgraph-specific metadata and stripped during composition.

## Workarounds & Alternatives

### Option 1: Validation at Subgraph Level (Recommended)

**Approach:** Handle `@constraint` validation in the subgraph resolver before executing the mutation.

```typescript
// In comments-server subgraph
const resolvers = {
  Mutation: {
    createComment: async (_, { input }, context) => {
      // Validate input using constraint rules
      validateConstraints(input);

      // Proceed with business logic
      return createCommentInDB(input);
    }
  }
}
```

**Pros:**
- ✅ Works with current Cosmo setup
- ✅ Validation happens closest to business logic
- ✅ Each subgraph controls its own validation rules

**Cons:**
- ❌ Directives not visible in router's public schema
- ❌ Clients can't see validation rules via introspection
- ❌ Validation logic must be manually implemented

### Option 2: Router-Level Validation Plugin

Cosmo Router supports custom modules (Go plugins) that can intercept requests.

**Approach:** Build a custom module that:
1. Reads directive metadata from a config file
2. Validates inputs before forwarding to subgraphs

**Pros:**
- ✅ Centralized validation logic
- ✅ Fails fast at the gateway (better performance)

**Cons:**
- ❌ Requires writing custom Go code
- ❌ Validation rules duplicated from schema
- ❌ More complex deployment

**Reference:** [Custom Modules Docs](https://cosmo-docs.wundergraph.com/router/custom-modules)

### Option 3: Schema Documentation Comments

**Approach:** Document constraints in field descriptions:

```graphql
input CreateCommentInput {
  """
  Comment ID (must be a valid UUID)
  """
  id: ID!

  """
  Comment content (1-10000 characters)
  """
  content: String!
}
```

**Pros:**
- ✅ Visible in introspection and GraphQL IDEs
- ✅ No code changes needed

**Cons:**
- ❌ Not machine-readable
- ❌ No automatic validation

### Option 4: Switch to Apollo Router

If directive preservation is critical, consider Apollo Router which supports `@composeDirective`.

**Pros:**
- ✅ Native support for custom directive composition per Federation 2.1+ spec
- ✅ Directives visible in public schema via introspection
- ✅ Full implementation of Federation 2.x features
- ✅ Apollo Router Core is open-source

**Cons:**
- ❌ Requires switching entire gateway stack
- ❌ Different configuration format than Cosmo
- ❌ May have different performance characteristics

**Note:** Apollo Router Core (the open-source version) fully supports `@composeDirective` as documented [here](https://www.apollographql.com/docs/federation/federated-types/federated-directives).

## Recommendations

### For Your Use Case (@constraint)

**Recommended Approach:** **Option 1 - Subgraph Validation**

1. Keep `@constraint` directives in `schemas/comments.graphql` for developer documentation
2. Implement validation in the comments-server subgraph using a library like:
   - [graphql-constraint-directive](https://github.com/confuser/graphql-constraint-directive)
   - [class-validator](https://github.com/typestack/class-validator)
3. Accept that the router's public schema won't show the directives

**Why:**
- Input validation belongs in the subgraph (closest to business logic)
- The router shouldn't know about subgraph-specific validation rules
- Clients don't typically rely on directive metadata for validation (they use TypeScript types, OpenAPI specs, etc.)

### Implementation Steps

1. **In comments-server (subgraph):**

```typescript
import { constraintDirective } from 'graphql-constraint-directive';

const schema = makeExecutableSchema({
  typeDefs,
  resolvers
});

// Apply constraint validation
const schemaWithConstraints = constraintDirective()(schema);
```

2. **In cosmo-gateway (router):**

No changes needed. The router forwards requests to subgraphs, which handle validation.

3. **Workflow:**

```
Client Request → Router (routes query) → Subgraph (validates @constraint, executes resolver)
                                              ↓
                                         Returns error if validation fails
```

## Why Doesn't Cosmo Support @composeDirective?

### Federation v2 vs Full Spec Compliance

Cosmo Router claims "Federation v1 & v2 compatibility," but this doesn't mean it implements **all** Federation 2.x features. Specifically:

1. **Core Federation Features (Cosmo ✅):**
   - `@key`, `@shareable`, `@inaccessible`, `@override`
   - Entity resolution and query planning
   - Basic composition

2. **Advanced Federation 2.x Features (Cosmo ❌):**
   - `@composeDirective` (custom directive preservation)
   - Some newer Federation 2.3+ directives

### Open Federation vs Apollo Federation

Cosmo follows the "Open Federation" specification, which is based on Apollo Federation but may not include all Apollo-specific extensions. The `@composeDirective` feature was introduced in Apollo Federation 2.1 and may not be part of the Open Federation core spec.

### Community Discussion

As of November 2025, there's no indication in Cosmo's documentation or GitHub issues that `@composeDirective` support is planned. If this feature is critical for your use case, consider:
1. Opening a feature request on [Cosmo's GitHub](https://github.com/wundergraph/cosmo/issues)
2. Using Apollo Router instead (full Federation 2.x compliance)
3. Implementing validation at the subgraph level (recommended for most cases)

## Open Questions

1. **Will Cosmo add @composeDirective support?**
   - No indication in docs or GitHub issues as of 2025-11-17
   - Cosmo follows Open Federation spec, which may intentionally differ from Apollo Federation 2.1+
   - Consider opening a feature request if this is important for your architecture

2. **Can we use schema extensions to preserve directives?**
   - Not tested, but unlikely to work given Cosmo's composition behavior
   - The composition process explicitly strips non-federation directives

3. **Do other Federation implementations support custom directives?**
   - **Apollo Router**: Yes (via `@composeDirective`) ✅
   - **Apollo Federation JS**: Yes (via `@composeDirective`) ✅
   - **Netflix DGS**: Partial support
   - **Hot Chocolate**: Yes (via schema stitching)
   - **Cosmo Router**: No ❌

## References

- [Apollo Federation Directives Docs](https://www.apollographql.com/docs/apollo-server/schema/directives)
- [Cosmo Federation Directives](https://cosmo-docs.wundergraph.com/federation/directives)
- [Cosmo Custom Modules](https://cosmo-docs.wundergraph.com/router/custom-modules)
- [graphql-constraint-directive](https://github.com/confuser/graphql-constraint-directive)

## Conclusion

### Your Understanding IS Correct (for Apollo)

You are **absolutely right** that Apollo Federation supports `@composeDirective` for preserving custom directives. The [Apollo documentation](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives#managing-custom-directives) you referenced is accurate and describes how this feature works in **Apollo Router** and **Apollo Federation**.

### But Cosmo Router Doesn't Implement It

The critical distinction is:

| Router | @composeDirective Support | Custom Directive Pass-Through |
|--------|--------------------------|-------------------------------|
| **Apollo Router** | ✅ Yes (since Federation 2.1) | ✅ Works as documented |
| **Cosmo Router** | ❌ No | ❌ Directives are stripped |

**Cosmo Router claims "Federation v2 compatibility" but doesn't implement all Federation 2.x features**, specifically `@composeDirective`.

### Recommendations for Cosmo Gateway

Since you're using **Cosmo Router**, custom directives like `@constraint` will NOT pass through composition. Your options are:

1. **✅ Recommended: Subgraph-level validation** - Implement `@constraint` validation in the comments-server subgraph using a library like `graphql-constraint-directive`

2. **Consider: Switch to Apollo Router** - If custom directive preservation is critical for your architecture, Apollo Router fully supports `@composeDirective` as you expected

3. **Request Feature**: Open an issue on [Cosmo's GitHub](https://github.com/wundergraph/cosmo/issues) requesting `@composeDirective` support

### Final Answer

- **Your understanding of Federation 2.1+**: ✅ Correct
- **Does it work in Cosmo?**: ❌ No
- **Does it work in Apollo?**: ✅ Yes

For future consideration: If directive preservation becomes critical across multiple use cases, evaluate switching to Apollo Router or contributing `@composeDirective` support to Cosmo.

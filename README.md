# Spacekit Apps Specification

> One declarative description of a full-stack web app — **data**, **business**, and **view** — that compiles to any stack.

Spacekit is a specification for describing an entire web application, its data model, its business logic, and its user interface, in a single, versioned document. A companion **target profile** says which stack to build it on. The `spacekit-cli` tool reads both and generates the database, the API/SDK, and the frontend from one source of truth.

```
  app.spacekit.yaml  (what the app is)  +  profile.yaml  (how to build it)
                              │
                         spacekit-cli
                              │
        data layer   +   business layer   +   frontend
```

---



## Why another spec

Spacekit is **not** the first attempt to describe a UI declaratively, and it doesn't pretend to be. Several mature projects already cover parts of this ground:

- **Framework-agnostic UI codegen**: [TeleportHQ UIDL](https://github.com/teleporthq/teleport-code-generators) and [Builder.io Mitosis](https://github.com/BuilderIO/mitosis) compile one component description to React, Vue, Svelte, and more.
- **Runtime-delivered declarative UI**: Adaptive Cards, server-driven UI, and newer generative-UI protocols stream a UI description that a host renders.
- **API description + codegen**: OpenAPI describes HTTP APIs and generates clients/servers.

Each of these describes **one layer**. Spacekit's bet is on the *seam between layers*. What none of the above gives you is a single document where a view's data binding, the capability it invokes, the entity that capability writes, and the workflow that entity's events trigger are all **cross-referenced and checkable together**. That unified contract, not "a declarative UI format", is the point.

Two design choices follow from it:

1. **One document, three layers, explicit links.** Every top-level definition (entity, capability, view, workflow, event) is referenceable by name via `@Name`. A view binds `@Book`, invokes `@PlaceOrder`, which writes `Order` and emits `OrderPlaced`, which triggers a `@FulfillOrder` workflow. Break a reference and generation fails before any code is emitted.
2. **The spec says *what*; the profile says *how*, and the profile can never change *what*.** The same app description generates a Vue + Pinia + Mongo app or a React Server Components + Postgres app by swapping profiles. The load-bearing invariant: **a profile changes the realization of an app, never its behavior.** `spacekit-cli conform` tests that two profiles produce behaviorally-equivalent apps.

---



## A taste

```yaml
spacekit: "0.2"
app: { name: Bookshop, version: 1.0.0 }

entities:
  Book:
    identity: id
    fields:
      id:    { type: id, generated: true }
      title: { type: text, required: true }
      price: { type: money, rules: { min: 0 } }
      stock: { type: integer, default: 0 }

capabilities:
  PlaceOrder:
    input:  { orderId: { type: id, required: true } }
    output: { order: @Order }
    writes: [{ updates: Order }]
    emits:  [OrderPlaced]
    policy: @OwnsOrder

workflows:
  FulfillOrder:                 # event-triggered saga with compensation
    on: OrderPlaced
    steps:
      - { do: ReserveStock,  with: { orderId: event.order.id }, compensate: ReleaseStock }
      - { do: ChargePayment, with: { orderId: event.order.id }, compensate: RefundPayment }
      - { do: ScheduleShipment, with: { orderId: event.order.id } }

views:
  Catalog:
    route: /
    data:  { books: { from: @ListBooks } }
    states: { loading: @Spinner, empty: { when: "books.length == 0", show: @EmptyState } }
    transitions: [{ to: @BookDetail }]
```

The full, runnable version lives in the specification document.

---



## Repository contents


| File                                                     | What it is                                                                                                                                                                                                                         |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[spacekit-spec-v0.2.md](./spacekit-spec-v0.2.md)`       | **The Spacekit Specification.** Defines the document format across all three layers — entities, capabilities, events, workflows, policies, widgets, views, flows, text, tokens — plus the reference model and compatibility rules. |
| `[spacekit-profile-v0.2.md](./spacekit-profile-v0.2.md)` | **The Spacekit Target Profile.** Defines how a profile binds a spec to a concrete stack (datastore, transport, framework, orchestration engine, architectural patterns), with inheritance, environments, and conformance.          |


---



## Quick start

```bash
# install the generator 
# download spacekit from website spacekit.xyz

# validate a spec + profile pairing
spacekit validate --spec bookshop.spacekit.yaml --profile react-postgres.profile.yaml

# generate the app for an environment
spacekit generate --spec bookshop.spacekit.yaml --profile react-postgres.profile.yaml --env prod

# prove two profiles produce the same app on different stacks
spacekit conform flux-vue-mongo.profile.yaml react-postgres.profile.yaml

# diff two spec versions for breaking changes
spacekit diff bookshop@1.0.yaml bookshop@1.1.yaml
```

---



## Relationship to OpenAPI

Spacekit capabilities are **transport-neutral**, they describe use cases, not endpoints. A profile with `business.emit_openapi: true` projects them into an OpenAPI document, so existing OpenAPI-based SDK generators plug straight in. Spacekit sits *above* OpenAPI: it can emit one, alongside the data layer and frontend that OpenAPI alone can't describe.

---



## Status & roadmap

**v0.2 (current, draft).** Adds workflows with saga compensation, field validation, view loading/empty/error states, a policy expression grammar, i18n strings, and compatibility/diffing rules. See each document's *Changes* section.

**Deferred to v0.3+.** Workflow branching beyond linear-with-compensation, policy functions, optimistic UI updates, constraint-based responsive layout, i18n pluralization, deployment/hosting targets in profiles, and drift detection against hand-edited output.

The format is a draft: expect breaking changes before v1.0, governed by the compatibility rules in the spec.

---

## License

Released under the **Apache License 2.0**, the conventional choice for a specification intended for free adoption.

## Contributing

Issues and proposals welcome. Changes to the format should state whether they are *compatible* or *breaking* under the spec's compatibility rules, and any spec addition that affects behavior should come with the matching profile realization so the two documents stay in lockstep.
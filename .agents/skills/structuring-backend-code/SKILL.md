---
name: structuring-backend-code
description: 'Sets where backend code lives and how each rule gets exactly one owner, in any language (Go, Rust, Java, Kotlin, TypeScript, Python): layers, domain rules and validators, shared packages, wire types, mappers, filters, input parsers and duplication. Use when writing, reviewing or refactoring backend services, handlers, domain logic, repository or store code, API types or CLIs, when a check or policy appears in more than one place, or when deciding where shared code goes.'
---

# Structuring backend code

Stack mechanics live in the stack skills (`writing-go-code`, `building-elysia-servers`). This skill says where things go and keeps each rule in one place.

## Layers

- Domain: types, invariants, validate and guard methods, policies (who may do what). No I/O.
- Service: one use case at a time. Calls domain methods, owns the transaction. Never re-implements a domain rule.
- Adapters: HTTP or RPC handlers, the store or repository, external clients. They map input and output, call the service, and add transport detail (status code, hint). No business rules.
- Dependencies point inward: adapters import the service and domain; the domain imports neither.

## One owner per rule

- The second time a predicate, policy, cap or check is needed, extract it into the layer that owns it (usually the domain) and call it everywhere. Never re-derive it inline. Copies drift: one handler answers 401 and three answer 403 for the same case.
- A domain validate or guard method is the path the code actually runs. A validator that only tests call is a defect: delete the hand-coded copies and call it.
- Each gate (deleted user, read-only credential, expired claim) is one function, checked at one point in the request. Check it before the lookup when the answer must not depend on whether the object exists.

## Where shared code goes

- Put shared code at the lowest layer that every user already imports, in a package named for what it holds (`money`, `pagination`), never a grab bag (`utils`, `common`, `helpers`).
- When packages of one layer import each other, move the shared piece into a helper package both import, never one importing the other. Enforce the direction with a guard (an import-rule test or a linter).
- Constants and enums used on both sides of a process boundary (client and server, CLI and API) live in one wire-types package both import. If one side may not import it, copy them and pin the copies with a test (see `writing-tests`).

## One shape per concept

- A filter or parameter set used by several routes, wire types and clients is one struct, embedded where needed, with one builder.
- Full and summary mappers for one entity: one builds on the other, so a new field cannot reach only one of them.
- A step repeated across handlers (emit an event, release a lock, create a link) is one function they call.
- A query fragment that must match an index exactly (a full-text expression) is one function.
- One parser per input kind (a CLI flag value, a query parameter). Every command or route that takes it uses it, so a value one accepts, all accept, and a missing required value is rejected the same way everywhere.

## Finding duplication

- Run a clone detector in the local gate (the stack skill names one). It catches copy-paste only.
- In every review, look for one rule written in different shapes: the same status, cap, permission or field list computed in two places. Fix it by extracting, never by syncing the copies.

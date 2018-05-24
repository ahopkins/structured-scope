# Structured Scope  (WIP)

## Abstract

This repository is meant to serve as a working outline of the generation of an abstract specification for the use of structured scopes in permission granting utilities.

## Introduction

The goal of this endeavor is to standardize and define the meaning, and usage of "scopes" for implementation in an authorization utility. It is a work in progress and **does not currently have any license**, and may be subject to change at any time. All copyright and other rights are hereby reserved.

## Purpose

The purpose of "scoping" is to provide a pass/fail response to a request for permission on a given, defined resource to authorized clients having the requisite permission level. A common application would be for permissioning on protected resources.

## Makeup of a scope

A scope is a `string` of characters encoded with `UTF-8` consisting of characters acceptable by [RFC 6749, Section 3.3](https://tools.ietf.org/html/rfc6749#section-3.3).

A scope has two components: (1) a namespace, and (2) actions. A scope's components (including multiple actions) are delimited by a colon:`:`.

A scope cannot be a `null` value, but should produce an `invalid_scope` error, as defined by [RFC, Section 4.1.2.1](https://tools.ietf.org/html/rfc6749#section-4.1.2.1)

> The requested scope is invalid, unknown, or malformed.

### Namespace

Every scope must have **zero** or **one** namespace. A namespace can either be **specific** or **global**. If a namespace is **global**, then it will be matched by all requests. A **specific** namespace can only be matched by requests carrying the same specific namespace.

The absense of a namespace (as discussed below) will indicate that no matching can be achieved on the namespace. This is **not** to be confused with a blank namespace, that will be inferred as being a **global namespace** (see below).

Valid characters: any UTF-8 accepted by [RFC 6749](https://tools.ietf.org/html/rfc6749#appendix-A) except a color `:`, and a space ` `.

The first delimited part of a scope is the namespace. An undefined namespace is in the **global namespace** (eg `:` or `:someaction` both produce the global namespace.) Alternatively, `global` is a protected word, and explicitly defines the namespace for a scope as the **global namespace**.

Possible namespaces:

- `foo` = `foo` as a Specific namespace
- `global` = Global namespace
- `:read` = Global namespace (inferred)
- `:` = Global namespace (inferred)
- ` ` = Empty string, NO namespace (ie, nothing can match)

### Actions

A scope may have either **zero** or **many** actions. The actions are a narrowing focus of the namespace. A scope without any actions is said to be at the **top level**. Actions are anything that follows the first delimited part of the scope (the namespace) and are separated by `:`.

Valid characters: any UTF-8 accepted by [RFC 6749](https://tools.ietf.org/html/rfc6749#appendix-A) except a colon `:`, and a space ` `.

Anything that follows any empty action (which is displayed as a double color: `::`) is a negatve, for example `::exclusion`. A negative action **only** impacts a specifically defined action, and has no impact upon (for example) a top level action. See below for discussion on matching. Negative actions are only permissible in base scopes, and not in tender scopes. If a tender scope contains a negation (`::`), then it should raise an error.

Where there are multiple actions, the order that they appear in shall have no consequence upon acceptance.

Possible actions on a scope:

- `foo` = Top level
- `:read:write` = Multiple actions, namely `read` and `write`
- `:` = Any action, wildcard
- `foo:` = Any action, wildcard
- `::read:write` = Any action except read and write

## Accepting a scope

### General

The pattern for acceptance is that a client offers a "tender" scope to an authorizing agent. That authorizing agent has a predefined "base" scope. The role of the authorizing agent is to match the tender against the base and produce a `pass`/`fail` result.

### Base

The base can be any valid scope, including negations.

### Tender

Tenders **may not** contain negations (`::`). And, if they do, the authorizing agent should produce and error without providing a `pass`/`fail` result.

### How scopes are matched

A tender shall `pass` against a given base if:

1. The namespace of the tender is equal to the namespace of the base, except a **global** namespace matches every other namespace, and an empty namespace is **unmatchable**.
2. If the base has a defined action, or actions, then the tender must:
    - be a top level scope with no defined actions, or
    - contain all of the same actions as the base*
3. If the base does NOT have a defined action (and therefore is top level), then the tender must be a top level scope also.
    
*Note: *An implementation of this specification may offer a validation mechanism that only requires **one** action to match*

### Multiple scopes

Scopes can be chained together as a single string separated by a space:

    admin user:read ::delete

Where there are multiple scopes, the order in which they appear shall have no importance and shall not play a part in the acceptance of the tender.

A validation of multiple scopes is valid if the tender matches **all** of the base scopes. *

*Note: *An implementation of this specification may offer a validation mechanism that only requires **one** tender to match the base*

## Examples of scopes

### Valid Scopes

- `admin`
- `user:read`
- `user:read:write`
- `:read`
- `:read:write`
- ` `
- `:`
- `::`
- `user:write:delete::read`

### Structured Scopes test cases

#### Simple single scopes - specific namespace

| Base scope | Tender scope | Intended outcome |
|---|---|---|
|user|something|fail|
|user|user|pass|
|user|user:read|fail|
|user:read|user|pass|
|user:read|user:read|pass|
|user:read|user:write|fail|
|user:read|user:read:write|pass|
|user:read:write|user:read|fail*|
|user:read:write|user:read:write|pass|
|user:read:write|user:write:read|pass|
|user:|user|pass|
|user:|user:read|pass|
|user:|user:write|pass|

#### Simple single scopes - global namespace

| Base scope | Tender scope | Intended outcome |
|---|---|---|
|:|:read|pass|
|:|admin|pass|
|:|*anything here*|pass|
|:read|admin|pass|
|:read|:read|pass|
|:read|:write|fail|

#### Simple multiple scopes

| Base scope | Tender scope | Intended outcome |
|---|---|---|
|user|something else|fail|
|user|something else user|pass|
|user:read|something:else user:read|pass|
|user:read|user:read something:else|pass|
|user foo|user|fail**|
|user foo|user foo|pass|
|user foo|foo user|pass|
|user:read foo|user foo|pass|
|user:read foo|user foo:read|fail|
|user:read foo|user:read foo|pass|
|user:read foo:bar|:read:bar|pass|

#### Complex scopes

| Base scope | Tender scope | Intended outcome |
|---|---|---|
|user::delete|user|pass|
|user::delete|user:read|pass|
|user::delete|user:delete|fail|
|user:read::delete|user|pass|
|user:read::delete|user:read|pass|
|user:read::delete|user:delete|fail|
|user:read::delete|user:read:delete|fail*|
|user:read user::delete|user:read:delete|fail**|
|user:read user::delete|user:read user:delete|fail**|
| |*anything here*|fail|
|::|*anything here*|fail|
    
Note: *The preferred method of failing any scope should be `::` and not ` ` for its explicit nature.

```
Legend
* This is the default outcome. However, the validator should be capable of receiving an instruction that instead of ALL actions being required, only one must match.
** This is the default outcome. However, the validator should be capable of receiving an instruction that instead of ALL required scopes being met, only ONE required scope is fulfilled.
```

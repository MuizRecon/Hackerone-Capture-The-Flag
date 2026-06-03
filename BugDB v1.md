# BugDB v1 - Hacker101 CTF

**Hunter:** Abdulmuiz (marex02)  
**Platform:** Hacker101  
**Difficulty:** Easy    
**Flags:** 1/1   
**Vulnerabilities:** GraphQL IDOR (Object-Level Authorization Bypass)

---

## Overview

BugDB v1 is a bug reporting application with a GraphQL API exposed directly through GraphiQL. There is no authentication of any kind. The challenge introduces GraphQL as an attack surface specifically how introspection can map an entire API, how private data can be marked without actually being protected, and how the `node` query can be used to bypass type-level filtering and read any object by ID.

---

## Reconnaissance

The landing page had one link **GraphiQL**. I clicked it. GraphiQL is an in-browser interface for sending GraphQL queries. No login page, no signup just an open input box and a message: **"Must provide query string."** The endpoint was live.

---

## Step 1 - Introspection

GraphQL has a built-in feature called **introspection**, it lets you ask the API to describe itself. What types of data it stores. What queries are available. What fields each type exposes. It's the first thing I run on any GraphQL target.

```graphql
{ __schema { types { name } } }
```

The response returned a full list of types. The ones that stood out:

- **Users** — user accounts
- **Bugs** — bug reports
- **Bugs_** — a second bugs type with an underscore 

> Two nearly identical types don't exist by accident. `Bugs_` looked like a leftover from a partial patch, something the developer forgot about.

[Introspection result showing Bugs_ type](./screenshots/recon-introspection.png)

---

## Step 2 - Mapping Each Type

I queried the fields on each suspicious type.

**Users:**
```graphql
{ __type(name: "Users") { fields { name } } }
```
Fields: `id`, `username`, `bugs`  no password.

**Bugs:**
```graphql
{ __type(name: "Bugs") { fields { name } } }
```
Fields: `id`, `reporterId`, `private`, `reporter`  no `text` field. I couldn't read bug content through this type.

**Bugs_:**
```graphql
{ __type(name: "Bugs_") { fields { name } } }
```
Fields: `id`, `reporterId`, `private`, `reporter`, **`text`** ← the difference.

> `Bugs_` exposed the actual content of a bug. The regular `Bugs` type did not. This told me the developer tried to hide bug content by removing the field but forgot they'd left a second type with it still exposed.

---

## Step 3 - Enumerating Users

```graphql
{ allUsers { edges { node { id username } } } }
```

Two users returned:
- **admin** — ID: `VXNlcnM6MQ==` → decoded: `Users:1`
- **victim** — ID: `VXNlcnM6Mg==` → decoded: `Users:2`

Sequential IDs. Simple to enumerate.

[allUsers query returning admin and victim accounts](./screenshots/recon-allusers.png)

---

## Step 4 - Finding the Private Bug

```graphql
{ allBugs { edges { node { id reporterId private reporter { id } } } } }
```

Two bugs returned:

| Bug | ID | Private | Reporter |
|---|---|---|---|
| Bug 1 | `QnVnczox` | false | admin |
| Bug 2 | `QnVnczoy` | **true** | victim |

Bug 2 was marked `private: true`. It appeared in the list, meaning I could see it existed but I couldn't read its content through `allBugs` because there was no `text` field on the `Bugs` type.

[allBugs query showing private true bug](./screenshots/recon-allbugs.png)

---

## Step 5 - Reading the Private Bug via `node`

GraphQL has a `node` query that fetches **any object directly by its ID**, regardless of type. Since the `text` field was on `Bugs_` and not `Bugs`, I used an **inline fragment** to tell GraphQL which type I expected:

```graphql
{ 
  node(id: "QnVnczoy") { 
    id 
    ... on Bugs_ { 
      text 
      private 
    } 
  } 
}
```

The API returned the full text of the private bug. No authentication check. No ownership check. Just the data.

**Flag was inside the `text` field.**

[node query returning private bug text and flag](./screenshots/flag1-node-idor.png)

---

## Why It Worked

The `private` field was just a **display label**. The app used it to filter bugs out of the `allBugs` list view, but there was no actual access control at the query level. The `node` query had no authorization check on it at all. Anyone who knew (or could derive) a bug's ID could read it directly, regardless of its `private` status.

> This is a textbook IDOR applied to a GraphQL API. The ID is the only thing standing between you and private data and there's no check confirming you're allowed to access it.

---

## Key Takeaways

1. **Run introspection on any GraphQL target first.** It maps the entire API surface in one go.

2. **Red flags are duplicate or leftover types like `Bugs_`.** Often they are left unprotected because the developer forgot them.

3. **The `node` query skips type-level filtering** if there is no object-level auth check behind it.

4. **`private: true` means nothing without access control.** A flag on data doesn't protect it, authorization logic does.

---

## Tools Used

| Tool | Purpose |
|---|---|
| GraphiQL | In-browser GraphQL query interface |
| Burp Suite | Request interception and analysis |
| Browser DevTools | Network inspection |

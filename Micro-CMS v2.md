# Micro-CMS v2 - Hacker101 CTF

**Hunter:** Abdulmuiz (marex02)  
**Platform:** Hacker101  
**Difficulty:** Moderate    
**Flags:** 2/2  
**Vulnerabilities:** UNION-based SQLi Auth Bypass · IDOR · HTTP Method Tampering · Insecure Client-Side Session

---

## Overview

Micro-CMS v2 is the "patched" version of v1. Authentication was added, edit access was locked behind a login page, and the changelog even claimed all prior security flaws were fixed. It wasn't fixed it was just dressed up. The login form itself was vulnerable to SQL injection, the session cookie stored privilege level client-side, and HTTP method checks were applied inconsistently. This challenge is about reading past the surface of a "secured" application.

---

## Reconnaissance

The app looked almost identical to v1, same two pages, same create button. But clicking **Edit** on any page redirected to a login form. Authentication had been added.

Before touching the login, I ran the same URL enumeration as always:

```
/page/3
/page/4
/page/5
```

`/page/3` returned **`403 Forbidden`** not `404`. That's a critical distinction:
- `404` = the page does not exist
- `403` = the page exists, but you're not allowed to see it

I noted `/page/3` as a target and moved to the login.

---

## Flag 1 — SQL Injection Auth Bypass + IDOR (Chained)

**Vulnerabilities:** UNION-based SQL Injection · IDOR

### Step 1 — Probing the login form

I tried common credentials first : `admin/admin`, `admin/password`, `test/test`. None worked. The error messages were informative:

- Wrong username → **"Unknown user"**
- Correct username, wrong password → **"Invalid password"**

Different error messages for different failure states = **user enumeration**. The app was leaking whether a username existed.

I then tested a single quote in the username field:

```
'
```

The app returned an **Internal Server Error**. The input was going straight into a SQL query.

### Step 2 — Testing OR-based bypass

Classic bypass attempt in the username field:

```sql
' OR 1=1-- -
```

Instead of a server error, I got **"Invalid password"**. A different response. The injection found a user in the database, but the password was being checked separately in application code, not purely in SQL. OR injection wasn't enough.

### Step 3 — UNION injection to control the query result

Since the password check was separate, I needed to control *what user the query returned* entirely, not just find a real one.

First, I needed to know how many columns the query returned. I tested:

```sql
' UNION SELECT 1-- -
```

Response: **"Invalid password"**  not an error. The UNION worked. One column.

Now I could make the query return any value I wanted as the username:

```
Username: ' UNION SELECT 'admin'-- -
Password: admin
```

The query no longer looked up a real user. It returned `'admin'` as a fake result. The application then checked if the password matched `'admin'` and it did, because I controlled both sides. **Login successful.**

> **The analogy:** The bouncer checks a guest list to find your name. Instead of giving your real name, you slip him a fake list with only the name you want on it. He finds it, checks your ID against it, and lets you in — because you wrote the list.

[UNION payload entered in login form and successful login](./screenshots/flag1-sqli-union-bypass-form.png)

### Step 4 — IDOR on /page/3

Once authenticated, the home page now showed three links. The third — **Private Page** — was at `/page/3`. The same page that returned `403` before I logged in.

I clicked it. The flag was there.

> **Why this matters:** Two vulnerabilities chained together. SQL injection alone got me authenticated. IDOR alone would have been blocked by the 403. Combined, they gave full access to a page meant for no one.

[Private page content at /page/3 after bypass](./screenshots/flag1-idor-page3.png)

---

## Flag 2 — HTTP Method Tampering on the Edit Endpoint

**Vulnerability:** HTTP Method Tampering

With a valid session I opened Burp Suite and sent requests to the edit endpoint through Repeater. A normal `GET /page/edit/1` loaded the edit form fine.

I then changed the HTTP method:

```
POST /page/edit/1
```

The response came back with the flag directly in it. The application applied access controls to `GET` requests on that endpoint but forgot to apply the same checks to `POST` requests on the same URL.

> **The vulnerability:** Developers often think about what happens when a user *visits* a page (GET) but forget that the same endpoint can respond differently to POST, PUT, or DELETE. If authorization is tied to the method rather than the endpoint, switching methods bypasses it entirely.

[GET vs POST response comparison in Burp Repeater](./screenshots/flag2-method-tampering.png)

---

## Bonus Finding — Insecure Client-Side Session Cookie

While working through v2 I noticed the session cookie looked unusual. I decoded the first part from base64 and it read:

```json
{"admin": true}
```

The application was storing **privilege level in the cookie itself** — on the client side. I verified this by:
1. Changing the cookie to `{"admin": false}` → locked out
2. Restoring it to `{"admin": true}` → access restored

> **The vulnerability:** Applications should never trust client-side data for authorization decisions. A user controls their own cookies and can change anything in them. Admin status must always be verified server-side against the database — not read from a value the attacker controls.

[Decoded session cookie showing admin true](./screenshots/bonus-cookie-tampering.png)

---

## Vulnerability Summary

| Flag | Location | Vulnerability | Status |
|---|---|---|---|
| Flag 1 | Login form + `/page/3` | UNION SQLi Auth Bypass + IDOR | ✓ |
| Flag 2 | `/page/edit/1` | HTTP Method Tampering | ✓ |


---

## Key Takeaways

1. **Scan for URLs before attacking the login.** Knowing /page/3 “was a” shaped the whole attack path.

2. **Error messages reveal information.** “Unknown user” versus “Invalid password” tells you the moment you have found a valid username.

3. **UNION injection gives you control over what a query returns, not just breaking it.** You can inject fake data and authenticate as that data.

4. **Chain vulnerabilities;** SQLi got me on my own. IDOR was mitigated. They both gave the flag.

5. **Test all HTTP methods for all endpoints.** GET, POST, PUT and DELETE. They can all behave differently.

6. **Do not store privilege level in a client-side cookie.** The user is in charge of their cookies and can change anything in them.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Burp Suite | Request interception, Repeater for method switching |
| Browser DevTools | Cookie inspection and page source review |
| Manual URL enumeration | Iterating page IDs in the address bar |
| base64decode.org | Decoding the session cookie |

# Micro-CMS v1 — Hacker101 CTF

**Hunter:** Abdulmuiz (marex02)  
**Platform:** Hacker101  
**Difficulty:** Easy    
**Flags:** 4/4   
**Vulnerabilities:** IDOR · Stored XSS · Attribute XSS · SQL Injection

---

## Overview

Micro-CMS v1 is a simple content management system with no authentication. Pages can be created, viewed, and edited freely. Under the surface, every single input was unprotected — user-supplied data went directly into the database and directly into SQL queries with no sanitization. This challenge covers four fundamental web vulnerabilities, each found in a different part of the same application.

---

## Reconnaissance

The landing page showed two links — a page called **Testing** at `/page/1` and **Markdown Test** at `/page/2`. A **Create Page** button was also available. No login required, everything was open.

Before touching any inputs I mapped the surface:
- Clicked through both pages to understand the application flow
- Tested the create page form and examined its fields
- Noticed edit functionality was accessible directly via URLs like `/page/edit/1`

**The rule I follow:** map first, attack second. You can't know what's exploitable until you know what exists.

---

## Flag 1 — IDOR on a Hidden Page

**Vulnerability:** Insecure Direct Object Reference (IDOR)

The home page listed two pages. But that doesn't mean only two pages exist, it means only two pages were *linked*. I manually iterated the page ID in the URL:

```
/page/3
/page/4
/page/5
```

Most returned `404 Not Found`. But `/page/5` came back with `403 Forbidden` — not a 404. That's a critical difference:
- `404` = the page doesn't exist
- `403` = the page exists, but you're not allowed to see it

The 403 told me something was there. IDOR on the view endpoint was blocked — the app was at least checking read access. But I wasn't done. If the view was protected, what about the **edit** endpoint? Developers often lock down one entry point and forget the other.

I tried:
```
/page/edit/5
```
No redirect. No 403. The edit form loaded directly with the page content — and the flag was inside it.

> **The vulnerability:** The app applied access control to viewing the page but forgot to apply the same check to editing it. Both endpoints reference the same object by ID — but only one was protected. Always test every endpoint that touches the same resource, not just the obvious one.

[Screenshot of IDOR on /page/edit/5](./screenshots/flag1-idor-edit.png)


---

## Flag 2 — Stored XSS in the Title Field

**Vulnerability:** Stored Cross-Site Scripting (XSS)

With surface mapping done, I shifted to testing inputs. The create page form had two fields, a **title** and a **body**. I tested whether either would execute code instead of displaying it.

I created a new page and entered this in the title field:

```html
<script>alert(1)</script>
```

When I returned to the home page, an alert box fired immediately. The script was stored in the database and rendered directly into the HTML without sanitization, executing for every user who loaded the home page.

> **The vulnerability:** Stored XSS means the payload is saved server-side and fires for every subsequent visitor, not just the attacker. In a real attack, `alert(1)` becomes cookie theft or malicious redirects.

[Screenshot of Stored XSS alert firing on home page](./screenshots/flag2-stored-xss.png)

---

## Flag 3 — SQL Injection on the Edit Endpoint

**Vulnerability:** SQL Injection (Error-based)

I turned to the URL structure for SQL injection testing. The edit page for page 1 was at `/page/edit/1`. I appended a single quote:

```
/page/edit/1'
```

The app returned a **database error**. The quote broke the underlying SQL query, which looked something like:

```sql
SELECT * FROM pages WHERE id = 1
```

With the quote appended, the query became syntactically invalid and the database threw an error — which the application exposed directly in the HTTP response. The flag was in that error output.

I also tested `/page/1'` but got a `404`. Same payload, different behavior — because each endpoint handles input differently.

> **The vulnerability:** User input was inserted directly into a SQL query without parameterization. A single quote is enough to break the syntax and cause unexpected behavior. Always test each endpoint separately.

[Screenshot of Database error on /page/edit/1'](./screenshots/flag3-sqli-error.png)

---

## Flag 4 — Attribute XSS in the Title Field

**Vulnerability:** Attribute Injection / XSS in HTML Context

I already knew the title field was vulnerable to Stored XSS via `<script>` tags. But I wanted to understand *how* the title was rendered in different contexts — specifically on the edit page where it appeared inside an HTML input element.

Checking the page source, the title was placed inside an attribute:

```html
<input type="text" name="title" value="Testing">
```

A `<script>` tag won't execute inside an attribute context. But if I could close the attribute early and inject an event handler, I could still run JavaScript. I entered this as the title:

```
" onmouseover="alert(1)
```

The rendered HTML became:

```html
<input type="text" name="title" value="" onmouseover="alert(1)">
```

The double quote closed the `value` attribute. `onmouseover` became a real event handler. When I hovered my mouse over the input, the alert fired and the flag appeared.

> **The vulnerability:** Many apps block `<script>` tags but forget that event handlers like `onmouseover`, `onclick`, and `onerror` can also execute JavaScript when injected into HTML attribute contexts. Input validation must account for all injection contexts, not just the obvious ones.

[Screenshot of Attribute XSS alert firing on mouseover](./screenshots/flag4-attribute-xss.png)

---

## Vulnerability Summary

| Flag | Location | Vulnerability | Impact |
|---|---|---|---|
| Flag 1 | `/page/edit/5` | IDOR | Unauthorized access to hidden content |
| Flag 2 | Title field (create page) | Stored XSS | Script execution for all visitors |
| Flag 3 | `/page/edit/1'` | SQL Injection | Database error disclosure |
| Flag 4 | Title field (edit page) | Attribute XSS | JavaScript execution via event handler |

---

## Key Takeaways

1. **Enumerate URLs manually.** The home page only shows what the developer wanted you to see — not everything that exists.
2. **Test every input for XSS.** Title fields, body fields, search bars — anything that accepts text and displays it back.
3. **A single quote is the fastest SQLi probe.** If the app breaks, input is going directly into a query.
4. **XSS is not only about `<script>` tags.** If your input lands inside an HTML attribute, try breaking out with a double quote and injecting an event handler.
5. **Test each endpoint separately.** The same payload that works on `/page/edit/1'` may return `404` on `/page/1'`.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Burp Suite | Request interception and Repeater |
| Browser DevTools | Page source review |
| Manual URL enumeration | Iterating page IDs directly in the address bar |

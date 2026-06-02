# HackerOne Capture The Flag


## About This Repo
 
This repository contains writeups for every CTF challenge I complete across platforms like Hacker101, PortSwigger Web Security Academy, and others. Each writeup documents my methodology, the vulnerabilities I found, the tools I used, and what I learned — written in a way that's useful to both recruiters and fellow learners.
 
 
---
 
## Progress Tracker
 
| Platform  | Challenge                                 | Difficulty | Status   | Flags |
|----------|-------------------------------------------|------------|----------|-------|
| Hacker101 | [Micro-CMS v1](./hacker101/micro-cms-v1/) | Easy       | Complete | 4/4   |
| Hacker101 | [Micro-CMS v2](./hacker101/micro-cms-v2/) | Moderate   | Complete | 3/3   |
| Hacker101 | [BugDB v1](./hacker101/bugdb-v1/)         | Easy       | Complete | 1/1   |
| Hacker101 | [BugDB v2](./hacker101/bugdb-v2/)         | Easy       | Complete | 1/1   |
 
 
---
 
## Vulnerability Index
 
A quick reference of every vulnerability type I've exploited so far:
 
| Vulnerability | Challenge |
|---|---|
| IDOR (Insecure Direct Object Reference) | Micro-CMS v1, Micro-CMS v2, BugDB v1 |
| Stored XSS | Micro-CMS v1 |
| Attribute XSS | Micro-CMS v1 |
| SQL Injection (Error-based) | Micro-CMS v1, Micro-CMS v2 |
| SQL Injection (UNION-based auth bypass) | Micro-CMS v2 |
| HTTP Method Tampering | Micro-CMS v2 |
| Insecure Client-Side Session | Micro-CMS v2 |
| GraphQL IDOR (node query) | BugDB v1 |
| GraphQL Broken Object-Level Authorization | BugDB v2 |
 
---
 
## Tools I Use
 
- **Burp Suite** — request interception, Repeater, Intruder
- **Browser DevTools** — cookie inspection, source review, network tab
- **GraphiQL** — GraphQL query interface
- **Manual enumeration** — because automation misses what curiosity finds
---
 
*Writeups are published after all flags in a challenge are found.*

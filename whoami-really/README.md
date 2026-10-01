# Whoami Really?

**Category:** Web Exploitation
**Difficulty:** Hard

## Challenge Description

> You have been given access to an internal office portal, but something is not quite right. Explore the portal, examine how authentication is handled, and follow the trail through the office's internal resources.

**Credentials provided:** `wiener:peter`

### Hint

> The Security Team was told there are "two vulnerabilities in the live system." I checked — it's 100% secure. Is it?

---

## Walkthrough

### 1. Initial Access

The challenge starts at a login page.

<img width="650" height="400" alt="wiener-login" src="https://github.com/user-attachments/assets/50d7a585-033b-4589-9992-2e9af801fe3e" />

Logging in with the provided credentials (`wiener:peter`) grants access to the internal office portal.

From the dashboard, an **Employee Panel** link is visible, but attempting to open it returns an access-denied response — `wiener` doesn't have the right privileges.

<img width="650" height="200" alt="Access denied to Employee Panel" src="https://github.com/user-attachments/assets/38bab147-6ff4-4b70-9fa2-88bf20a72a6f" />

### 2. Vulnerability #1 — Broken Authentication via Unsigned JWT

Inspecting the session cookie with a cookie editor revealed a JWT. Decoding it on [jwt.io](https://jwt.io) showed the header used `"alg": "none"` — meaning the token's signature isn't verified server-side, and the payload can be modified freely without needing a secret key.

<img width="1400" height="550" alt="JWT decoded showing alg none" src="https://github.com/user-attachments/assets/da0400f6-5621-4acf-aa7e-e7265eba0c61" />

The token's payload carried the authenticated username in plaintext, which the backend appeared to trust implicitly for authorization decisions:

```json
{"username": "wiener"}
```

Since the `alg: none` scheme requires no signature, the username field could be changed directly and the token re-encoded:

```
# Modified session token (alg: none, empty signature)
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VybmFtZSI6ImNhcmxvcyJ9.
```

Swapping the cookie value from `wiener` to `carlos` and reloading the panel granted access as a higher-privileged user.

<img width="950" height="400" alt="Access granted as carlos" src="https://github.com/user-attachments/assets/ac12b0c8-8658-41db-820e-3a1783f58e80" />

This confirmed **Vulnerability #1: JWT `alg:none` signature bypass / broken access control**, allowing privilege escalation through simple client-side token tampering.

### 3. Pivoting Through the Boss Room

With the forged `carlos` session, a new area — the **Boss Room** — became accessible, exposing internal files that should only be reachable by that account.

<img width="1592" height="492" alt="Boss Room file listing" src="https://github.com/user-attachments/assets/adfd4264-5522-43b2-b093-bf2e2aad3fc9" />

One of the exposed files contained a second set of credentials:

```
username: cabin-user
password-hash: f1bf67922c4ea5f0693888a1b3c07ff7847cbe9e
```

The hash format matched **SHA-1**. Running it through a cracking tool/lookup recovered the plaintext password: `cabin1987`.

### 4. The Cabin — User-Agent and Date-Based Access Control

Attempting to log in to the **Cabin** with `cabin-user:cabin1987` failed, but the response surfaced two clues:

1. **"Unsupported browser"** — implying a `User-Agent` check.
2. **"This browser only works in 1987"** — implying a `Date` header check.

Checking `robots.txt` revealed a disallowed path referencing a custom client: `H4S-Browser`. Combining that with the date clue, the following headers were added to the login, auth, and flag requests:

```
User-Agent: H4S-Browser
Date: Sun, 09 Sep 1987 00:00:00 GMT
```

<img width="1000" height="450" alt="Cabin access with spoofed headers" src="https://github.com/user-attachments/assets/419a098b-9cde-487b-a782-3ec6e7595bb3" />

With both headers spoofed, the Cabin login page became accessible. Logging in with `cabin-user:cabin1987` returned the flag.

<img width="950" height="500" alt="Flag captured" src="https://github.com/user-attachments/assets/6a3f0d28-32a6-4720-a1a5-c44ac12e6d1e" />

This confirmed **Vulnerability #2: Client-controlled header trust (`User-Agent` / `Date` used as an access-control mechanism)**, which is trivially bypassed since both headers are fully attacker-controlled.

---

## Vulnerabilities Found

| # | Vulnerability | Root Cause | Impact |
|---|---------------|------------|--------|
| 1 | Broken Authentication — JWT `alg:none` | Server accepted unsigned JWTs and trusted the `username` claim without verifying a signature | Full privilege escalation (`wiener` → `carlos`), unauthorized access to Employee/Boss Room data |
| 2 | Broken Access Control — Client-Controlled Headers | Access to the Cabin resource was gated on `User-Agent` and `Date` request headers, both fully attacker-controlled | Unauthorized access to restricted resource ("Cabin") bypassing intended access policy |

---

## Key Takeaways

- **Never trust client-side-verifiable JWTs.** If `alg:none` is accepted, or the signature isn't checked, the entire token becomes attacker-editable.
- **Hashes aren't secrets.** A SHA-1 hash of a weak, guessable password is only marginally better than storing it in plaintext.
- **Headers are not identity.** `User-Agent`, `Date`, `Referer`, etc. are trivially spoofed and should never gate access to sensitive resources.
- Chaining low-effort issues (tamperable token → exposed file → weak hash → spoofable headers) can still lead to full compromise — defense in depth matters at every layer.

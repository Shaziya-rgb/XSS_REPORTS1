---
## Title
Reflected Cross-Site Scripting (XSS) via `returnTo` Parameter in Redirect URI

---
## Vulnerability Type
Reflected XSS

---
## Summary
The `returnTo` parameter on the account confirmation page is intended to redirect users after they confirm their details. However the value is echoed directly into the `href` attribute of the Continue button without any validation or sanitization. The source code itself has a comment saying "$returnTo is echoed into the href WITHOUT any validation" which confirms the issue at the code level. This means an attacker can pass a `javascript:` URI as the returnTo value and when the user clicks the Continue button JavaScript executes directly in their browser instead of redirecting them anywhere.

---
## Vulnerable Endpoint
```
https://kzlabs.com/57.php?returnTo=
```
**Vulnerable Parameter:** `returnTo` parameter echoed raw into the `href` attribute of the Continue button without any validation

---
## Steps to Reproduce
1. Go to `https://kzlabs.com/57.php` which simulates Shopify's account confirmation endpoint.
2. First test with a unique term like `unique` and view the page source.
3. In the source code around line 461 you can see the returnTo value reflected raw inside the Continue button href:
```
<a href="unique"><" class="btn-continue">
```
4. The source comment confirms "$returnTo is echoed into the href WITHOUT any validation" and also notes that `javascript:alert(document.cookie)` as returnTo will execute JS when clicked.
5. Now craft the payload using a `javascript:` URI and inject a script tag to execute JavaScript:
```
unique<img width="1881" height="900" alt="Screenshot 2026-05-24 031552" src="https://github.com/user-attachments/assets/822c14f1-f784-4b21-87f4-e6ed86327a7d" />
'"><script>alert(1)</script>
```
6. Visit the following URL directly:
```
https://kzlabs.com/57.php?returnTo=unique%27"><script>alert(1)</script>
```
7. The page loads and a JavaScript alert box pops up immediately displaying `1` confirming the payload broke out of the href attribute and executed.

---
## Payload Used
```
unique'"><script>alert(1)</script>
```

---
## Proof of Concept

**Screenshot 1** — Page source showing the returnTo value `unique` reflected raw inside the Continue button href at line 461, with the source comment clearly stating "$returnTo is echoed into the href WITHOUT any validation" and noting that `javascript:alert(document.cookie)` will execute JS when clicked, confirming the vulnerable injection point.

<img width="1881" height="900" alt="Screenshot 2026-05-24 031552" src="https://github.com/user-attachments/assets/deb3bc8a-bcb3-4cac-991d-4c725cfb73b2" />


**Screenshot 2** — Alert box displaying `1` triggered on the account confirmation page immediately after loading the crafted URL, with the full payload visible in the URL bar confirming reflected XSS execution via the returnTo parameter.

<img width="1905" height="947" alt="Screenshot 2026-05-24 031503" src="https://github.com/user-attachments/assets/3e6d172b-cd53-4b66-aac7-ac7c34788799" />


---
## Impact
- An attacker can craft a malicious URL and send it to any user going through the account confirmation flow and the moment they click Continue the script executes in their browser
- Steal session cookies of anyone who clicks the Continue button giving the attacker full access to their account
- The source code also confirms this enables open redirect meaning an attacker can silently redirect victims to any external site like `?returnTo=https://evil.com` for phishing
- Since this simulates Shopify's account confirmation flow users are in a high trust moment confirming their details making them far less likely to suspect anything malicious
- Can be used to redirect victims to fake login pages to harvest credentials right after they just confirmed their account details

---
## Remediation
1. Validate the `returnTo` parameter against a strict allowlist of trusted URLs or paths before reflecting it anywhere on the page, reject anything that is not a relative path or a trusted domain
2. Block `javascript:` URIs entirely from being accepted as a redirect value as there is no legitimate reason for a redirect parameter to contain JavaScript
3. If you're using PHP then use `htmlspecialchars()` with `ENT_QUOTES` flag before echoing any user supplied value into an HTML attribute to prevent breaking out of the attribute context
4. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` and block script tags from being injected through URL parameters
5. Implement a strict Content Security Policy (CSP) header that blocks inline script execution so even if a payload gets injected the browser refuses to run it
6. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

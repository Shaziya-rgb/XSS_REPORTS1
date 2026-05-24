---
## Title
Reflected Cross-Site Scripting (XSS) via Username in URL Path Segment

---
## Vulnerability Type
Reflected XSS

---
## Summary
The username segment in the URL path of the Imgur-style messaging page is reflected raw into a double-quoted `href` attribute without any encoding or sanitization. The source code itself has a comment saying "VULNERABLE: $username is echoed raw into this double-quoted href" which confirms the issue at the code level. Since the username is placed directly inside an `href` attribute without escaping, an attacker can break out of the attribute by closing the quote and injecting an img tag with an onerror handler to execute JavaScript automatically the moment the page loads.

---
## Vulnerable Endpoint
```
https://kzlabs.com/58.php/account/{username}/messages
```
**Vulnerable Parameter:** Username segment in the URL path reflected raw inside the `href` attribute of the profile link

---
## Steps to Reproduce
1. Go to `https://kzlabs.com/58.php/account/unique/messages` and view the page source.
2. Around line 382 you can see the username `unique` reflected raw inside the profile link href:
```
href="/account/unique/profile">
```
3. The source comment confirms "VULNERABLE: $username is echoed raw into this double-quoted href" and even suggests the payload to use.
4. Since the value is injected into a double-quoted attribute with no encoding, craft the following payload to break out and inject an img tag:
```
unique"><img src=x onerror=alert(1)>
```
5. Visit the following URL directly:
```
https://kzlabs.com/58.php/account/unique"><img src=x onerror=alert(1)>/messages
```
6. The page loads and the alert box fires immediately displaying `1` confirming the payload broke out of the href attribute and executed via the onerror handler.

---
## Payload Used
```
unique"><img src=x onerror=alert(1)>
```

---
## Proof of Concept

**Screenshot 1** — Page source showing the username `unique` reflected raw inside the profile link `href` attribute at line 382, with the source comment clearly stating "VULNERABLE: $username is echoed raw into this double-quoted href" and the suggested payload `testcatplzignore"><img src=x onerror=prompt(document.domain)>` confirming the exact injection point.

<img width="1893" height="960" alt="image" src="https://github.com/user-attachments/assets/21af3f81-ab14-4b16-a12d-d552f8196a60" />


**Screenshot 2** — Alert box displaying `1` triggered on the Imgur-style messages page immediately after loading the crafted URL, with the full payload `unique"><img src=x onerror=alert(1)>` visible in the URL bar and the raw payload rendering on the profile card confirming reflected XSS execution via the URL path segment.

<img width="1885" height="919" alt="Screenshot 2026-05-24 032734" src="https://github.com/user-attachments/assets/cc6dbb90-60ed-4299-9b7a-ba8be71e3c22" />


---
## Impact
- An attacker can craft a malicious profile URL and send it to any Imgur user and the moment they open it the script executes automatically without any interaction needed beyond clicking the link
- Steal session cookies of anyone who visits the crafted URL giving the attacker full access to their Imgur account
- Since the payload fires via an img onerror handler the victim does not need to click or hover over anything making it a fully automatic execution
- Can be used to silently redirect victims to fake login pages to harvest credentials
- Since Imgur is a widely used platform and links are commonly shared in posts and messages the attack surface for distributing this URL is extremely large

---
## Remediation
1. Escape the username before reflecting it inside the `href` attribute by replacing `"` with `&quot;` so it cannot break out of the attribute context
2. Filter out HTML tags like `<script>`, `<img>`, `<svg>` from the URL path segment before reflecting anything back to the page
3. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` so even if a quote slips through the method won't execute
4. If you're using PHP then use `htmlspecialchars()` function with `ENT_QUOTES` flag before echoing any user supplied value into an HTML attribute as this encodes both single and double quotes
5. Implement a strict Content Security Policy (CSP) header that blocks inline script execution so even if a payload gets injected the browser refuses to run it
6. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

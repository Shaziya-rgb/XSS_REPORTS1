---
## Title
Reflected Cross-Site Scripting (XSS) via Search Parameter in JavaScript Analytics Context

---
## Vulnerability Type
Reflected XSS

---
## Summary
The search parameter on the Help Center page reflects user input directly inside a JavaScript analytics tracking script without any escaping or sanitization. When you search for something, the search term gets embedded raw into a JavaScript object as the value of `internalSearchTerm`. This means if you break out of the string context with a quote and inject JavaScript, it executes immediately. No storage involved, the payload fires the moment the page loads with the crafted URL.

---
## Vulnerable Endpoint
```
https://kzlabs.com/55.php?search=
```
**Vulnerable Parameter:** `search` parameter reflected inside the JavaScript analytics block without string escaping

---
## Steps to Reproduce
1. Go to `https://kzlabs.com/55.php` and open the Help Center search.
2. First search for a unique term like `unique1` and view the page source.
3. In the source code around line 425 you can see the search term reflected raw inside the JavaScript analytics block:
```
internalSearchTerm: "unique1",
```
4. This confirms the search term is injected directly into a JS string with no escaping at all.
5. Now craft the payload to break out of the string context and inject JavaScript:
```
unique"-alert(1)-"
```
6. Submit this as the search term or visit the following URL directly:
```
https://kzlabs.com/55.php?search=unique"-alert(1)-"
```
7. The page loads and a JavaScript alert box pops up displaying `1` confirming the payload broke out of the JS string and executed.

---
## Payload Used
```
unique"-alert(1)-"
```

---
## Proof of Concept

**Screenshot 1** — Page source showing the search term `unique` reflected raw inside the JavaScript analytics block at line 425 as the value of `internalSearchTerm`, with a comment in the source itself saying "The search term is reflected here WITHOUT JavaScript string escaping" confirming the vulnerability at the code level.

<img width="1909" height="891" alt="Screenshot 2026-05-24 025957" src="https://github.com/user-attachments/assets/5ce81427-7fad-4110-b2f0-4c9c6e248c1b" />


**Screenshot 2** — Alert box displaying `1` triggered on the Search Results page after submitting the payload `unique"-alert(1)-"`, with the payload visible in both the URL bar and the search input field confirming reflected XSS execution in the JavaScript context.

<img width="1890" height="943" alt="Screenshot 2026-05-24 025925" src="https://github.com/user-attachments/assets/730bab4d-a371-4325-98d7-89a8ec5e3789" />


---
## Impact
- An attacker can craft a malicious URL and send it to any Equifax user and the moment they click it the script executes in their browser
- Steal session cookies of anyone who clicks the link giving the attacker full access to their account
- Since this is on the Equifax Help Center it targets real users who trust the domain making phishing via this vector extremely effective
- Can be used to redirect victims to fake login pages that look identical to Equifax to harvest credentials
- The vulnerability exists inside an analytics script which shows the dev team never considered it as an injection point making it easy to miss during code review

---
## Remediation
1. Escape the search term before embedding it inside a JavaScript string by replacing `"` with `\"` so it cannot break out of the string context.
2. Filter out HTML tags like `<script>`, `<img>`, `<svg>` from the search field before reflecting anything back to the page
3. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` so even if a quote slips through the method won't execute
4. If you're using PHP then use `json_encode()` when injecting any user supplied value into a JavaScript context as it handles all necessary escaping automatically, and use `htmlspecialchars()` for any user input reflected in HTML context
5. Implement a strict Content Security Policy (CSP) header that blocks loading of external scripts as this would have directly prevented any external payload from loading even if the input slips through
6. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

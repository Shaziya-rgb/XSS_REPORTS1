---
## Title
Reflected Cross-Site Scripting (XSS) via Search Parameter in HTML Attribute Context

---
## Vulnerability Type
Reflected XSS

---
## Summary
The search parameter on the PUBG Community Hub page reflects user input directly inside an HTML attribute called `data-query` without using `htmlspecialchars` or any encoding. The source code even has a comment saying "The $p parameter is echoed directly into the data-query attribute WITHOUT htmlspecialchars" which confirms the vulnerability at the code level. Since the value is injected into an attribute without proper quoting or escaping, an attacker can break out of the attribute and inject event handlers like `onmouseover` to execute JavaScript without needing any HTML tags at all.

---
## Vulnerable Endpoint
```
https://kzlabs.com/56.php?p=
```
**Vulnerable Parameter:** `p` parameter reflected inside the `data-query` HTML attribute without encoding

---
## Steps to Reproduce
1. Go to `https://kzlabs.com/56.php` and open the Community Hub search.
2. First search for a unique term like `unique'">` and view the page source.
3. In the source code around line 435 you can see the search term reflected raw inside the `data-query` attribute:
```
data-query='unique'">'
```
4. The source code comment confirms "The $p parameter is echoed directly into the data-query attribute WITHOUT htmlspecialchars" confirming the injection point.
5. Since the attribute uses single quotes and no escaping is applied, craft the following payload to break out and inject an event handler:
```
unique' onmouseover='alert(1)
```
6. Submit this as the search term or visit the following URL directly:
```
https://kzlabs.com/56.php?p=unique'+onmouseover%3D'alert(1)
```
7. The page loads and hovering over the search results area triggers the JavaScript alert box displaying `1` confirming the payload broke out of the attribute and executed.

---
## Payload Used
```
unique' onmouseover='alert(1)
```

---
## Proof of Concept

**Screenshot 1** — Page source showing the search term `unique` reflected raw inside the `data-query` attribute at line 435 with the source comment clearly stating "The $p parameter is echoed directly into the data-query attribute WITHOUT htmlspecialchars" and "Payload to exploit: '><img src=a onerror=alert(document.cookie)>" confirming the vulnerable injection point.

<img width="1888" height="915" alt="Screenshot 2026-05-24 030325" src="https://github.com/user-attachments/assets/8ae42855-ca54-49cc-a9f1-e7da259316ed" />


**Screenshot 2** — Alert box displaying `1` triggered on the PUBG Community Hub search results page after submitting the payload, with the payload `unique' onmouseover='alert(1)` visible in the search filter field and the URL bar confirming reflected XSS execution in the HTML attribute context.

<img width="1893" height="923" alt="Screenshot 2026-05-24 030258" src="https://github.com/user-attachments/assets/bc07ccd9-30bc-4028-90f6-f339ad676c1d" />

---
## Impact
- An attacker can craft a malicious URL and send it to any PUBG Community Hub user and the moment they interact with the page the script executes in their browser
- Steal session cookies of anyone who visits the crafted link giving the attacker full access to their account
- Since this is on a trusted gaming platform domain users are far less likely to suspect the link is malicious making social engineering extremely effective
- Can be used to redirect victims to fake login pages to harvest credentials
- The vulnerability exists inside an HTML attribute which is a less obvious injection point and easy to miss during code review

---
## Remediation
1. Escape the `p` parameter before reflecting it inside the `data-query` attribute by replacing `'` with `&#39;` and `"` with `&quot;` so it cannot break out of the attribute context
2. Filter out JavaScript event handlers like `onmouseover`, `onerror`, `onclick` from user input before reflecting anything back to the page
3. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` so even if a quote slips through the method won't execute
4. If you're using PHP then use `htmlspecialchars()` function with `ENT_QUOTES` flag before rendering any user input inside HTML attributes as this encodes both single and double quotes
5. Implement a strict Content Security Policy (CSP) header that blocks inline script execution so even if a payload gets injected the browser refuses to run it
6. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

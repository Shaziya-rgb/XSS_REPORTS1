---
## Title
Reflected Cross-Site Scripting (XSS) via Unquoted Attribute Injection in URL Path Segment

---
## Vulnerability Type
Reflected XSS

---
## Summary
The `POST_ID` segment in the URL path of this Reddit-style Shreddit API comments page is reflected raw into an unquoted `id` attribute on the "See More Comments" button at the bottom of the page. Unlike typical attribute injection where you need to break out of quotes, this one has no quotes at all around the attribute value. That means a simple space character is enough to inject a brand new HTML attribute directly, so adding `onmouseover=alert(document.domain)` after a space in the POST_ID turns the button into an XSS trigger the moment anyone hovers over it.

---
## Vulnerable Endpoint
```
https://kzlabs.com/59.php/svc/shreddit/api/comments/askreddit/{POST_ID}/t1_COMMENT_ID
```
**Vulnerable Parameter:** `POST_ID` path segment reflected raw into an unquoted `id` attribute on the See More Comments button

---
## Steps to Reproduce
1. Go to `https://kzlabs.com/59.php/svc/shreddit/api/comments/askreddit/t3_u9po1l/t1_i5u8kpl` and view the page source.
2. Replace the POST_ID segment with `hello123` and view source again, around line 711 you can see it reflected raw inside an unquoted id attribute:
```
<button id=hello123 class="see-more-btn">
```
3. The source comment at line 705 confirms "$post_id is echoed raw into an UNQUOTED id attribute. A space in the post_id injects a new HTML attribute directly."
4. Since there are no quotes to break out of, a single space is all that is needed to inject a new attribute. Craft the following payload:
```
hello123 onmouseover=alert(1)
```
5. Visit the following URL directly:
```
https://kzlabs.com/59.php/svc/shreddit/api/comments/askreddit/hello123%20onmouseover=alert(1)/t1_i5u8kpl
```
6. The page loads and hovering over the See More Comments button triggers the alert box displaying `1` confirming the payload injected a new attribute and executed.

---
## Payload Used
```
hello123 onmouseover=alert(1)
```

---
## Proof of Concept

**Screenshot 1** — Page source showing the POST_ID value `hello123` reflected raw inside the unquoted `id` attribute of the See More Comments button at line 711, with the source comment clearly stating "$post_id is echoed raw into an UNQUOTED id attribute. A space in the post_id injects a new HTML attribute directly" confirming the exact injection point.

<img width="1901" height="915" alt="Screenshot 2026-05-24 033953" src="https://github.com/user-attachments/assets/1f471823-713f-48a3-8e43-4675bb861263" />


**Screenshot 2** — Alert box displaying `1` triggered on the Reddit-style comments page after hovering over the See More Comments button, with the crafted URL visible in the URL bar confirming reflected XSS execution via unquoted attribute injection in the URL path segment.

<img width="1894" height="932" alt="Screenshot 2026-05-24 033922" src="https://github.com/user-attachments/assets/41eb1856-ba39-4d0b-b233-d64a2564827a" />


---
## Impact
- An attacker can craft a malicious URL and share it anywhere on the platform and the moment a user hovers over the See More Comments button the script fires
- Steal session cookies of anyone who visits the crafted URL giving the attacker full access to their Reddit account
- What makes this interesting is that there are no quotes to break out of making it a less obvious injection point that is easy to miss during code review
- Can be used to silently redirect victims to fake login pages to harvest credentials
- Since Reddit is a massively visited platform and comment threads are shared widely the attack surface for distributing this crafted URL is extremely large

---
## Remediation
1. Wrap the `id` attribute value in double quotes so the POST_ID cannot inject new attributes using a space character
2. Filter out spaces and event handler keywords like `onmouseover`, `onerror`, `onclick` from the POST_ID segment before reflecting it back to the page
3. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` so even if an attribute gets injected the method won't execute
4. If you're using PHP then use `htmlspecialchars()` function with `ENT_QUOTES` flag before echoing any user supplied value into an HTML attribute
5. Implement a strict Content Security Policy (CSP) header that blocks inline script execution so even if a payload gets injected the browser refuses to run it
6. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

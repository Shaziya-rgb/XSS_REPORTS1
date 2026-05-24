---
## Title
Stored Cross-Site Scripting (XSS) via Rich Text Editor HTML Tab in Article Body

---
## Vulnerability Type
Stored XSS

---
## Summary
The Body field in the Write section has a rich text editor with two tabs, Write and HTML. The HTML tab accepts raw HTML directly and does not sanitize it before saving to the database. Whatever you put in there gets stored and then rendered as actual HTML when the article loads, which means the payload fires for anyone who visits that page.

---
## Vulnerable Endpoint
```
https://kzlabs.com/61.php
```
**Vulnerable Parameter:** Body field (HTML tab inside the Write article form)

---
## Steps to Reproduce
1. Log in to the application at `https://kzlabs.com/61.php` with a valid account.
2. Navigate to the **Write** tab.
3. Fill in the **Title** and **Tags** fields with any value.
4. In the **Body** field, click on the **HTML** tab and enter the following payload:
```
"><img src=x onerror=alert(1)>
```
5. Submit the article.
6. You are redirected to the My Articles page and the article shows as Published.
7. Observe that a JavaScript alert box pops up immediately displaying `3` — confirming the payload was stored and executed.
8. Anyone who opens that article will trigger the same alert.

---
## Payload Used
```
"><img src=x onerror=alert(1)>
```

---
## Proof of Concept

**Screenshot 1** — The Write form showing the payload entered inside the HTML tab of the Body field, confirming the editor accepts raw HTML with no filtering at all.

<img width="1915" height="941" alt="Screenshot 2026-05-24 010323" src="https://github.com/user-attachments/assets/0da9bc06-6aba-46bc-a6c3-d76fa1047151" />


**Screenshot 2** — Alert box displaying `3` triggered automatically after the article was published and the My Articles page loaded, with the app itself showing the toast message "Article published — XSS payload fires when your articles page loads" which confirms the stored execution.

<img width="1889" height="943" alt="Screenshot 2026-05-24 010335" src="https://github.com/user-attachments/assets/b014a9aa-27ee-4f55-92ec-3331dab2333f" />


---
## Impact
- Steal session cookies of every user who reads the article
- Take over accounts without needing passwords
- Since articles are publicly visible the attack surface goes beyond just authenticated users
- Can inject fake login forms to harvest credentials
- One published article is enough to silently hit every single visitor

---
## Remediation
1. Filter out HTML tags like `<script>`, `<img>`, `<svg>` from the Body field before saving anything to the database
2. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` so even if a tag slips through the method won't execute
3. If you're using PHP then use `htmlspecialchars()` function before rendering any user input back to the page
4. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

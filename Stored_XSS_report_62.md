## Title
Stored Cross-Site Scripting (XSS) via Profile Signature Field in Forum

---
## Vulnerability Type
Stored XSS

---
## Summary
The Signature field in the My Profile section does not sanitize or encode user-supplied input before storing it in the database. The page even shows a Live Preview section that renders the signature as actual HTML, which makes it obvious the input is being treated as markup. Once saved, the payload fires for anyone who visits a page where that signature appears — which on a forum means every post the user has ever made.

---
## Vulnerable Endpoint
```
https://kzlabs.com/62.php
```
**Vulnerable Parameter:** Signature field inside the My Profile page

---
## Steps to Reproduce
1. Log in to the application at `https://kzlabs.com/62.php` with a valid account.
2. Navigate to **My Profile** from the left sidebar.
3. Scroll down to the **Profile Signature** section.
4. In the **Signature** field enter the following payload:
```
"><img src=x onerror=alert(1)>
```
5. Click **Save Profile**.
6. Observe that a JavaScript alert box pops up immediately displaying `3` — confirming the payload was stored and executed.
7. The toast at the bottom confirms "Profile saved! Your signature is now active on all your posts" meaning the payload is now attached to every post this user has made across the entire forum.

---
## Payload Used
```
"><img src=x onerror=alert(1)>
```

---
## Proof of Concept

**Screenshot 1** — The Edit Profile page showing the payload entered inside the Signature field, with the Live Preview section at the bottom already rendering the HTML, confirming the app treats signature input as raw HTML with no filtering at all.

<img width="1890" height="947" alt="Screenshot 2026-05-24 012125" src="https://github.com/user-attachments/assets/92dc43a6-9743-4585-89a7-4762ad51483f" />


**Screenshot 2** — Alert box displaying `3` triggered automatically right after hitting Save Profile, with the toast message "Profile saved! Your signature is now active on all your posts" confirming the payload is now stored and will fire across all forum posts by this user.

<img width="1885" height="944" alt="Screenshot 2026-05-24 012135" src="https://github.com/user-attachments/assets/885f9ad1-40b2-4538-84af-bd9517d388f4" />


---
## Impact
- The signature appears on every post the affected user has made so the payload fires on multiple pages across the forum, not just one
- Steal session cookies of every user who visits any thread where this user has posted
- Take over accounts without needing passwords
- Since forum threads are publicly visible the attack surface goes beyond just authenticated users
- Can inject fake login forms to harvest credentials
- One profile save is enough to plant the payload across the entire post history of that account

---
## Remediation
1. Filter out HTML tags like `<script>`, `<img>`, `<svg>` from the Signature field before saving anything to the database
2. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` so even if a tag slips through the method won't execute
3. If you're using PHP then use `htmlspecialchars()` function before rendering any user input back to the page
4. Use a server side HTML sanitization library like HTMLPurifier for PHP or DOMPurify for JavaScript to strip dangerous tags and attributes before saving or rendering the signature since it is intentionally rendered as HTML
5. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

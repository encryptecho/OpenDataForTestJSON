# Web App Security Checklist for Future Apps

**For:** Django / Django REST Framework + React / Vite / TypeScript apps  
**Use case:** SaaS dashboards, education portals, non-profit apps, file upload systems, admin panels  
**Author:** Encrypt-Echo  

---

## 1. Main Security Mindset

Security is not one setting. It is a complete workflow.

A secure app should protect:

- user accounts
- uploaded files
- private documents
- API endpoints
- admin dashboard
- database records
- server configuration
- backups
- logs
- frontend rendering

The most important rule:

> Never trust user input, uploaded files, browser requests, or frontend-only permissions.

Anything coming from the user must be validated on the backend.

---

## 2. File Upload Security

File uploads are one of the most dangerous parts of a web application.

A bad uploaded file usually harms your system when the app:

- processes the file
- stores it incorrectly
- serves it publicly
- shows it to admins/users
- allows executable file types
- allows very large files

### 2.1 Safe File Upload Rules

Use this workflow for every upload:

```text
1. User must be authenticated.
2. Check user permission.
3. Check file size.
4. Check allowed extension.
5. Check real MIME/content type.
6. Rename the file.
7. Store it in a safe folder.
8. Do not execute uploaded files.
9. Scan or validate file when possible.
10. Serve private files only after permission check.
```

### 2.2 Allowed File Types

For profile images:

```text
Allow: JPG, JPEG, PNG, WebP
Reject: SVG, HTML, JS, PHP, EXE, SH, PY, ZIP
Max size: 2–5 MB
```

For student documents:

```text
Allow: PDF, JPG, JPEG, PNG
Reject: SVG, HTML, JS, PHP, EXE, SH, PY, ZIP, DOCM
Max size: 5–10 MB depending on your app
```

### 2.3 Why SVG Should Usually Be Rejected

SVG looks like an image, but it is XML-based and can contain scripts.

For public uploads, avoid SVG unless you sanitize it very carefully.

### 2.4 Rename Uploaded Files

Do not keep the original user filename as the final storage filename.

Bad example:

```text
/media/passport_ahmed_original.jpg
```

Better example:

```text
/media/student-documents/8f3d2a9e-2341-4c91-bf21.webp
```

Benefits:

- avoids privacy leaks
- avoids dangerous filenames
- avoids path traversal tricks
- avoids overwriting files

### 2.5 Keep Uploads Separate from Code

Good structure:

```text
project/
├── backend/
├── frontend/
├── media/
├── staticfiles/
└── docker-compose.yml
```

Avoid storing uploads inside:

```text
backend/apps/
backend/config/
backend/templates/
backend/static/
```

Uploaded files should be treated as data, not code.

---

## 3. Django Media Security

### 3.1 Django Settings

Example:

```python
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

Django does not execute uploaded `.py` files from `MEDIA_ROOT`. They are just files.

The danger usually comes from bad web server configuration or unsafe file processing.

### 3.2 Public vs Private Media

Public media:

```text
profile pictures
public blog images
public university logos
```

Private media:

```text
student passports
transcripts
contracts
invoices
payment documents
medical or legal documents
```

Private files should not be directly accessible by public URL.

Bad:

```text
https://example.com/media/passports/student1.pdf
```

Better:

```text
User requests file → Django checks permission → file is served only if allowed
```

### 3.3 Nginx Media Configuration

Safe Nginx example:

```nginx
location /media/ {
    alias /var/www/app/media/;
}
```

This serves media as files only.

Dangerous example:

```nginx
location /media/ {
    include fastcgi_params;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
}
```

Do not allow uploaded files to be executed as PHP, Python, CGI, or scripts.

### 3.4 Caddy Media Concept

With Caddy, the idea is the same:

```text
/media/ should serve files only.
/media/ should not execute code.
private files should go through backend permission checks.
```

---

## 4. Image Processing Security

If your app resizes images or generates thumbnails, it must open and process uploaded files.

That creates risk.

### 4.1 Risks

A malicious image can cause:

- high memory usage
- high CPU usage
- backend crash
- thumbnail generation failure
- possible exploitation of vulnerable image libraries

### 4.2 Safe Rules

Use these protections:

```text
limit file size
limit image dimensions
use updated image libraries
convert uploaded images to safe formats
strip metadata when needed
process images in background jobs if heavy
reject corrupted images
```

Example limits:

```text
Max file size: 5 MB
Max width: 4000 px
Max height: 4000 px
Allowed formats: JPEG, PNG, WebP
```

---

## 5. XSS Protection

XSS means Cross-Site Scripting.

It happens when user input is displayed as executable HTML or JavaScript.

### 5.1 What XSS Can Do

XSS may allow attacker JavaScript to:

- read visible page data
- steal non-HttpOnly cookies
- abuse the logged-in user session
- submit forms as the user
- change account details
- create fake login screens
- attack admins who open infected content

### 5.2 Django Template Safety

Safe:

```django
{{ user_comment }}
```

Dangerous for user content:

```django
{{ user_comment|safe }}
```

Avoid using `|safe` with anything submitted by users.

### 5.3 React Safety

Safe:

```tsx
<p>{userInput}</p>
```

Dangerous:

```tsx
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

Avoid `dangerouslySetInnerHTML` unless the HTML is sanitized.

### 5.4 Rich Text Content

If your app allows rich text, use a sanitizer.

Example rule:

```text
Allow simple formatting only: bold, italic, lists, links.
Reject scripts, iframes, forms, event handlers, and unknown tags.
```

### 5.5 Content Security Policy

Use a Content Security Policy to reduce XSS impact.

Example header:

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self';
```

---

## 6. Cookie and Session Security

### 6.1 Secure Cookie Settings

Use:

```python
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_SAMESITE = "Lax"

CSRF_COOKIE_HTTPONLY = False
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_SAMESITE = "Lax"
```

### 6.2 Meaning

```text
HttpOnly: JavaScript cannot read the cookie.
Secure: cookie is sent only over HTTPS.
SameSite: helps reduce CSRF risk.
```

### 6.3 Important Note

`HttpOnly` protects cookies from direct JavaScript theft.

But XSS can still perform actions using the victim’s active browser session.

So you still need strong XSS protection.

---

## 7. CSRF Protection

CSRF means Cross-Site Request Forgery.

It tricks a logged-in user’s browser into sending unwanted requests.

### 7.1 Django Protection

Django has CSRF protection for normal forms.

For DRF APIs, your setup depends on your auth method:

```text
Session authentication: CSRF required.
JWT in Authorization header: CSRF risk is usually lower, but XSS risk remains.
```

### 7.2 Safe Rules

Use:

```text
CSRF tokens for session-based auth
SameSite cookies
safe CORS configuration
no state-changing GET requests
POST/PUT/PATCH/DELETE for changes
```

Bad:

```text
GET /delete-account
```

Good:

```text
DELETE /api/account/
```

---

## 8. Authentication Security

### 8.1 Strong Login Protection

Use:

```text
rate limiting
login throttling
MFA/2FA for admins and agents
strong password rules
password reset protection
new-device notification
session expiration
audit logs
```

### 8.2 Password Storage

Never store plain text passwords.

Use Django’s built-in password hashing.

Do not create your own password hashing system.

### 8.3 Multi-Factor Authentication

For a SaaS app, MFA should be required for:

```text
admins
agents
staff users
finance users
users who access private documents
```

MFA methods ranked from stronger to weaker:

```text
security key / passkey
app-based authenticator code
email code
SMS code
```

SMS is better than no MFA, but it is weaker than authenticator apps or security keys.

---

## 9. Authorization and Permissions

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What are you allowed to access?
```

### 9.1 Common Broken Access Control Problem

Bad example:

```text
/api/applications/101/
```

A student changes the URL to:

```text
/api/applications/102/
```

If they can see another student’s application, this is a serious security issue.

### 9.2 Backend Must Enforce Permissions

Never rely only on frontend route protection.

Bad mindset:

```text
The button is hidden, so the user cannot access it.
```

Correct mindset:

```text
The backend checks every request and blocks unauthorized access.
```

### 9.3 DRF Permission Examples

Use permission classes such as:

```text
IsAuthenticated
IsAdminUser
custom IsOwner
custom IsAssignedAgent
custom IsApplicationParticipant
```

Example logic:

```text
Student can see only their own applications.
Agent can see only assigned students.
Admin can see all records.
Hospital/university partner can see only related records.
```

---

## 10. API Security

### 10.1 Use HTTPS Only

Production APIs must use HTTPS.

Use:

```text
https://api.example.com
```

Avoid production APIs over plain HTTP.

### 10.2 Rate Limiting

Protect endpoints from abuse.

Apply rate limits to:

```text
login
password reset
registration
email verification
file upload
search endpoints
payment endpoints
contact forms
```

### 10.3 Pagination

Do not return unlimited records.

Bad:

```text
GET /api/students/ returns 100,000 students
```

Good:

```text
GET /api/students/?page=1&page_size=20
```

### 10.4 Filtering and Search

Validate filters.

Avoid letting users search fields they should not access.

Example:

```text
Students should not search all applications in the system.
Agents should search only assigned records.
Admins may search globally.
```

### 10.5 Error Messages

Do not leak sensitive details.

Bad:

```json
{
  "error": "Postgres error in table users: password_hash column failed"
}
```

Good:

```json
{
  "error": "Invalid request."
}
```

Log technical details internally, but do not expose them to users.

---

## 11. SQL Injection Protection

Django ORM protects against most SQL injection when used correctly.

Safe:

```python
User.objects.filter(email=email)
```

Dangerous:

```python
User.objects.raw(f"SELECT * FROM users WHERE email = '{email}'")
```

Avoid building SQL with string formatting.

If raw SQL is necessary, use parameterized queries.

---

## 12. CORS Security

CORS controls which frontend domains can call your backend from the browser.

### 12.1 Development

Development may allow:

```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:5173",
]
```

### 12.2 Production

Production should allow only real domains:

```python
CORS_ALLOWED_ORIGINS = [
    "https://education-sng.com",
    "https://www.education-sng.com",
]
```

Avoid:

```python
CORS_ALLOW_ALL_ORIGINS = True
```

Do not use wildcard CORS in production.

---

## 13. Admin Dashboard Security

Django admin is powerful and must be protected.

Use:

```text
strong admin passwords
MFA for admins
non-obvious admin URL
IP restriction if possible
login rate limiting
staff permissions carefully
audit logs
no shared admin accounts
```

Example:

```text
Avoid: /admin/
Better: /secure-control-panel-9382/
```

Changing the URL is not full security, but it reduces automated noise.

---

## 14. Environment Variables and Secrets

Never commit secrets to GitHub.

Keep these in `.env` or secret manager:

```text
SECRET_KEY
DATABASE_URL
EMAIL_PASSWORD
AWS_ACCESS_KEY
JWT_SIGNING_KEY
STRIPE_SECRET_KEY
```

Bad:

```python
SECRET_KEY = "hardcoded-secret"
```

Good:

```python
SECRET_KEY = os.environ.get("SECRET_KEY")
```

Add `.env` to `.gitignore`.

---

## 15. Django Production Settings

Production must use:

```python
DEBUG = False
ALLOWED_HOSTS = ["example.com", "www.example.com"]
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
X_FRAME_OPTIONS = "DENY"
SECURE_CONTENT_TYPE_NOSNIFF = True
```

Only enable HSTS after HTTPS is fully working.

---

## 16. Frontend Security

### 16.1 Do Not Store Sensitive Tokens in LocalStorage If Possible

LocalStorage is accessible by JavaScript.

If XSS happens, localStorage tokens can be stolen.

Safer options:

```text
HttpOnly cookies for session/auth tokens
short-lived access tokens
refresh token rotation
strict CSP
```

### 16.2 Avoid Dangerous HTML Rendering

Avoid:

```tsx
dangerouslySetInnerHTML
```

Avoid rendering unsanitized HTML from users.

### 16.3 Protect Frontend Routes, But Do Not Trust Them

Frontend protection improves UX.

Backend permission checks provide real security.

---

## 17. Docker and Deployment Security

### 17.1 Docker Rules

Use:

```text
small official base images
non-root container user when possible
.env files outside Git
separate dev and prod compose files
limited exposed ports
named volumes for persistent data
regular image updates
```

### 17.2 Expose Only Needed Ports

Usually public ports:

```text
80
443
```

Avoid exposing database ports publicly.

Bad:

```text
Postgres exposed to internet on 5432
```

Good:

```text
Postgres accessible only inside Docker network
```

### 17.3 Reverse Proxy

Use Nginx or Caddy in front of your app.

The reverse proxy should handle:

```text
HTTPS
static files
media routing
request size limits
security headers
compression
proxy to backend/frontend
```

---

## 18. Server Security

For Ubuntu VPS:

```text
create non-root user
use SSH key login
disable root SSH login
disable password SSH login
set up UFW firewall
allow only 22/80/443 or your custom SSH port
install Fail2ban
keep system updated
use automatic security updates if possible
monitor logs
```

Example firewall rules:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

---

## 19. Backup Security

Backups are part of security.

Use:

```text
database backups
media file backups
encrypted backups
off-server backups
backup rotation
restore testing
```

A backup is not trustworthy until you test restoring it.

### 19.1 What to Backup

For Django apps:

```text
database
media uploads
.env file or secret documentation
Docker compose files
Caddy/Nginx configs
```

Do not rely only on the VPS provider snapshot.

---

## 20. Logging and Monitoring

Log important security events:

```text
failed login attempts
password reset requests
admin login
file uploads
permission denied events
new device login
role changes
deleted records
payment events
```

Do not log sensitive data:

```text
passwords
full tokens
secret keys
full credit card data
private document contents
```

---

## 21. Payment Security

If using Stripe or another payment provider:

```text
never store card numbers yourself
use hosted checkout when possible
verify webhook signatures
log payment status changes
protect subscription endpoints
keep billing actions admin-only or owner-only
```

Your app should store:

```text
customer ID
subscription ID
payment status
plan name
billing history metadata
```

Your app should not store:

```text
full card number
CVV
raw banking secrets
```

---

## 22. Email Security

Protect email features from abuse.

Use:

```text
rate limit contact forms
verify email ownership
avoid leaking whether an account exists
use secure password reset tokens
short expiration for reset links
```

Bad password reset message:

```text
No user exists with this email.
```

Better:

```text
If an account exists, we sent password reset instructions.
```

---

## 23. Real Security Checklist Before Launch

Before deploying, check this list:

```text
[ ] DEBUG=False
[ ] HTTPS working
[ ] HSTS enabled after HTTPS is stable
[ ] SECRET_KEY is not in GitHub
[ ] .env is ignored by Git
[ ] database is not public
[ ] media folder does not execute code
[ ] upload size limits exist
[ ] dangerous file types rejected
[ ] private files require permission checks
[ ] CORS allows only real frontend domains
[ ] CSRF configured correctly
[ ] cookies use Secure and SameSite
[ ] admin panel protected
[ ] rate limiting enabled
[ ] API permissions tested
[ ] users cannot access other users’ data
[ ] logs do not expose secrets
[ ] backups configured
[ ] restore process tested
[ ] dependencies updated
[ ] server firewall enabled
[ ] Fail2ban enabled
```

---

## 24. Education SNG Example Security Model

For an education agency SaaS app:

### Student

Can:

```text
view own profile
upload own documents
view own applications
send messages to assigned agent
```

Cannot:

```text
view other students
view all agents
change application status directly
access admin files
```

### Agent

Can:

```text
view assigned students
review assigned documents
update assigned application progress
message assigned students
```

Cannot:

```text
view all students unless permitted
access admin-only settings
change payment records unless permitted
```

### Admin

Can:

```text
manage users
assign agents
view applications
manage universities
review audit logs
manage documents
```

Must have:

```text
MFA
strong password
audit logging
limited staff permissions
```

---

## 25. Best Security Rule for Every Feature

Before building any feature, ask:

```text
Who can create it?
Who can read it?
Who can update it?
Who can delete it?
Can another user guess the ID and access it?
Can this input become HTML, SQL, file path, or command?
Can this file harm the server or another user?
What happens if this request is repeated 1,000 times?
What should be logged?
What should never be logged?
```

---

## 26. Final Simple Summary

To protect your future apps:

```text
Validate every input.
Protect every API with permissions.
Do not trust frontend-only security.
Do not execute uploaded files.
Reject dangerous file types.
Keep private files private.
Use HTTPS.
Use secure cookies.
Prevent XSS and CSRF.
Rate limit sensitive endpoints.
Protect admin access.
Keep secrets out of GitHub.
Backup and test restore.
Monitor logs.
```

Security is not something added at the end. It must be designed from the first day of the project.

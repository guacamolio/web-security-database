# web-security-database
school project web page login security testing and making it secure enough


Here's a full plan. I assumed Python + Flask + SQLite. It's simple, secure by default, and easy to explain in a defense. If your school requires another stack (PHP/MySQL, Node, etc.), tell me and I'll adapt the steps.

1. Simplify the design

Your Donewell dashboard has a lot that the assignment doesn't ask for (focus blocks, weekly stats, "Plus" upsell, calendar, insights). Cut it down to 4 pages:

Page	Contents
Register	Name, email, password, confirm password
Login	Email, password, link to register
Dashboard	Top bar (logo, projects link, logout), task list, filter/sort, "Add task"
Projects	Project list, project page with members, invitations, shared tasks

The dashboard layout is just: sidebar or tabs (Active / Completed) → filter by priority → sort by due date → task rows (checkbox, title, priority badge, due date, edit/delete). Use one CSS file and a single-column layout that collapses on mobile; that covers the responsive requirement.

2. Database design (5 tables)
users: id, name, email (UNIQUE), password_hash, created_at
tasks: id, title, description, due_date, priority (Low/Medium/High), is_done, owner_id, project_id (nullable), assigned_to (nullable)
projects: id, name, owner_id
project_members: project_id, user_id, role (owner/member)
invitations: id, project_id, invited_user_id, status (pending/accepted/declined)

Personal tasks have project_id = NULL. Shared tasks have a project.

3. Step-by-step build order

Step 1: Setup. Create a folder, a virtual environment, and install flask, flask-login, flask-wtf (forms + CSRF), flask-sqlalchemy, argon2-cffi (or use Werkzeug's hashing). Put the secret key in an environment variable, not in code.

Step 2: Database models. Write the 5 tables with SQLAlchemy. Use foreign keys and UNIQUE on email.

Step 3: Register and login.

Validate input (valid email, name length, password at least 8-10 characters).
Hash the password with Argon2 or generate_password_hash; never store the plain one.
Login: look up the user by email, verify the hash, start a session with Flask-Login.
Use the same error message for wrong email or wrong password ("Invalid email or password").
Logout clears the session.

Step 4: Protect pages. Put @login_required on every route except register and login.

Step 5: Personal tasks (CRUD). Create, edit, delete, toggle done. Every query must include owner_id == current_user.id.

Step 6: Dashboard. Show active and completed tasks separately, with a filter by priority and sort by due date.

Step 7: Projects. Create a project (the creator becomes owner and a member).

Step 8: Invitations. The owner enters a registered user's email, which creates a pending invitation. The invited user sees it and clicks Accept (added to members) or Decline.

Step 9: Shared tasks. Members can create tasks in the project and assign them to another member. Check membership on every project route.

Step 10: Test the full scenario from section 4 of your spec with two accounts. Then try to attack your own app (section 5 below).

4. Security checklist (the main focus)

Passwords

Argon2/bcrypt hashing with salt (built in), never plain text or MD5/SHA1.
Never log passwords or put them in error messages.

Database

Use parameterized queries / ORM only, never string-built SQL. This stops SQL injection.
Keep the database file outside the web-accessible folder, with restricted file permissions.
Don't commit the .db file or secrets to Git (.gitignore).
Make regular backups.
If you want data-at-rest protection, encrypt the disk, or use SQLCipher for an encrypted SQLite file. For sensitive text fields, you can encrypt with the cryptography library (Fernet), with the key in an environment variable.

Access control (the most commonly failed part)

Never trust IDs from the URL. For /task/5/edit, load the task and check: is it mine, or am I a member of its project? If not, return 403/404.
Check roles: only the owner can invite people or delete the project.
Only the invited user can accept or decline their own invitation.

Input and output

Validate on the server (client-side validation is just for convenience).
Jinja2 auto-escapes output, which prevents XSS. Don't use |safe on user content.
CSRF tokens on every form (Flask-WTF does this).

Sessions and transport

Cookies: HttpOnly, SameSite=Lax, Secure (when on HTTPS).
Session timeout, and use HTTPS in production.
Add rate limiting on login (Flask-Limiter), e.g. 5 attempts per minute, to slow brute force.

General

debug=False in production.
Keep dependencies updated.
5. Security tests to run (and mention in your report)
Visit /dashboard while logged out → redirected to login.
Log in as user A, change the task ID in the URL to one of user B's → blocked.
Type ' OR 1=1 -- into the login form → fails normally.
Put <script>alert(1)</script> in a task title → displayed as text.
Open the database file → the password column shows only hashes.
Register the same email twice → rejected.
Non-member opens a project URL → blocked.
6. Suggested folder structure
todo-app/
  app.py
  models.py
  forms.py
  routes/ (auth.py, tasks.py, projects.py)
  templates/ (base.html, login.html, register.html, dashboard.html, project.html)
  static/style.css
  .env          (secrets, not in Git)
  .gitignore
7. Suggested schedule
Day 1: setup, models, register/login/logout
Day 2: personal tasks and dashboard
Day 3: projects, invitations, shared tasks
Day 4: security testing, responsive CSS, cleanup
Day 5: documentation and demo prep

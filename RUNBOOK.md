# RUNBOOK: Onboard an application to SSO with OIDC (Grafana)

A runbook is a short checklist you can follow to do a task the same way every time. This one is based on what I did in Lab 05. Full write-up with screenshots: [README.md](README.md)

---

**What it does:**
It connects Grafana to Authentik so people can log in with their Authentik account and MFA, and get the right Grafana role based on their group.

**When to run it:**
- **In this lab:** once, by hand.
- **At a real company (from what I've learned):** it usually starts when the team that owns an app sends in a request (like a help desk ticket) asking for SSO. The **app owner** decides who should get which role, and the **security / identity team** approves the setup before it goes live.

**Before you run it, get these from the app team:**
- [ ] The app's exact **redirect URI** (the address to send users back to after login). Every character has to match.
- [ ] Whether the app can safely keep a secret (if yes, use a **Confidential** client)
- [ ] What user information the app needs (here: `openid`, `email`, `profile`)
- [ ] The app's **roles** and which group should get each one (here: Admin, Editor, Viewer)
- [ ] What role people should get if they aren't in any group (here: Viewer)
- [ ] A **backup admin login** on the app that doesn't use SSO, in case something breaks
- [ ] Some test users: one for each role, plus one with no group

**Run order:**
1. **Provider (in Authentik):** create an OAuth2/OpenID provider, set it to Confidential, add the redirect URI, and pick the scopes. Copy the Client ID and Secret straight into Grafana. Don't save them in notes.
2. **Application (in Authentik):** create the application and connect it to the provider.
3. **Grafana settings:** turn on Generic OAuth and enter the Client ID, Secret, scopes, and the three Authentik addresses.
4. **Role mapping:** create the groups in Authentik, add the users, and put the role rule in Grafana:
   ```
   contains(groups[*], 'Grafana Admins') && 'Admin' || contains(groups[*], 'Grafana Editors') && 'Editor' || 'Viewer'
   ```

*Why does the provider come first?* Everything else depends on it. You can't create the application without a provider to connect it to, and you can't set up Grafana until the provider has given you the Client ID, Secret, and addresses. Role mapping goes last because it only matters once login is working.

**How you know it worked:**

| Test user | Group | Should land as | What to check |
|---|---|---|---|
| Marc Spector | Grafana Admins | **Admin** | Can create dashboards and open Administration |
| Illyana Rasputin | Grafana Editors | **Editor** | Can create and edit dashboards |
| WP Tester | *(none)* | **Viewer** | Can only look, can't create or edit |

To double-check, log in with Grafana's built-in admin and go to **Administration → Users → Organization users**. All three should show the right role.

**If it goes wrong:**

**Problem 1: The redirect URI was wrong**
- **What happened:** the redirect URI in my Authentik provider was `http://10005/login/generic_oauth`. It had the port number but was missing the server name in front of it, so Authentik couldn't send users back to the right Grafana address.
- **Where to look:** Authentik → Providers → Grafana OIDC → Redirect URIs (see `screenshots/04-provider-redirect-uri.png`)
- **What fixes it:** using the full address, `<grafana-url>/login/generic_oauth`, so it matches exactly what Grafana sends.
- **What to check first next time:** look at the browser address bar during the failed login, find the `redirect_uri=` part, and compare it letter by letter with the redirect URI in Authentik.

**Problem 2: A user logged in but got the wrong role (happened in my testing)**
- **What I saw:** Illyana, who is in Grafana Editors, logged in fine but was only a **Viewer** and couldn't create or edit dashboards.
- **Where:** Grafana → Authentication → Generic OAuth → User mapping → *Role attribute path*
- **Why:** I typed `'Grafana Edit ors'` with an extra space, so it didn't match the group name and fell back to Viewer.
- **What fixed it:** correcting it to `'Grafana Editors'` and logging in again.
- **What to check first next time:** make sure the group names in the rule match Authentik **exactly**, including capital letters and spaces.

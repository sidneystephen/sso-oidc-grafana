# Lab 05: Setting Up Single Sign-On (SSO) for Grafana with Authentik

This is my fifth security home lab. The goal was to let people sign in to **Grafana** (a dashboard app) using their **Authentik** account, so they don't need a separate Grafana password. I also wanted each person's Grafana permissions to be set automatically based on the group they're in.

I'm still learning this, so I explain things the way I came to understand them while doing the lab.

**Short step-by-step version:** [RUNBOOK.md](RUNBOOK.md)

---

## Some terms I had to learn first

| Term | What it means (in my words) |
|---|---|
| **SSO (Single Sign-On)** | You log in once with one account and it gets you into multiple apps. |
| **OIDC (OpenID Connect)** | The "language" the login system and the app use to talk to each other about who you are. |
| **Identity Provider (IdP)** | The system that holds the user accounts and checks passwords. Here, that's **Authentik**. |
| **Client ID / Client Secret** | Like a username and password, but for the *app* instead of a person. Grafana uses them to prove to Authentik that it's really Grafana. The secret must be kept private. |
| **Redirect URI** | The address Authentik sends you back to after you log in. It has to match Grafana's address exactly. |
| **Scopes** | What information the app is asking for about you (here: `openid`, `email`, `profile`). |

### How the login works

1. I click **"Sign in with Authentik"** on Grafana.
2. Grafana sends me to Authentik.
3. I enter my password and my MFA code (the MFA was set up in Lab 2).
4. Authentik asks if I'm OK sharing my email and profile with Grafana.
5. Authentik sends me back to Grafana, and Grafana knows who I am and which group I'm in.

---

## Part 1: Create the provider in Authentik

The **provider** holds the technical login settings (secrets, addresses, scopes). I found it helpful to think of the provider as the "wiring" and the application (Part 2) as the "front door."

1. In Authentik I went to **Applications → Providers → New Provider** and picked **OAuth2/OpenID Provider**.

   ![Choosing the provider type](screenshots/01-authentik-new-provider-type.png)

2. I named it `Grafana OIDC` and chose the authorization flow that asks the user for consent.

   ![Provider name and flow](screenshots/02-provider-name-and-flow.png)

3. I set the **Client Type** to **Confidential**, because Grafana runs on a server and can keep a secret safe. Authentik created a Client ID and Client Secret for me (blurred here).

   ![Client type (Client ID blurred)](screenshots/03-provider-confidential-client.png)

4. I added the **Redirect URI**. This is where Authentik sends users back to Grafana: `<grafana-url>/login/generic_oauth`

   ![Redirect URI](screenshots/04-provider-redirect-uri.png)

5. I made sure the scopes `openid`, `email`, and `profile` were selected.

   ![Scopes](screenshots/05-provider-scopes.png)

6. After saving, Authentik showed **"Provider not assigned to any application."** That's normal. I fix it in the next part.

   ![Provider not assigned yet](screenshots/06-provider-not-yet-assigned.png)

## Part 2: Create the application in Authentik

The **application** is what users actually see in Authentik, and it's where you control who is allowed in.

I went to **Applications → New Application**, named it `Grafana`, and connected it to the `Grafana OIDC` provider I just made.

![New application](screenshots/07-new-application-form.png)
![Application connected to the provider](screenshots/08-application-linked-to-provider.png)

Authentik then gave me three addresses that Grafana needs:

| What Grafana calls it | Address |
|---|---|
| Auth URL | `http://server/application/o/authorize/` |
| Token URL | `http://server/application/o/token/` |
| API URL | `http://server/application/o/userinfo/` |

## Part 3: Set up Grafana

In Grafana I went to **Administration → Authentication → Generic OAuth**.

![Grafana authentication options](screenshots/09-grafana-auth-providers.png)

I filled in:
- **Display name:** `Authentik` (this is the text on the login button)
- **Client ID and Client Secret:** copied from Authentik
- **Scopes:** `openid`, `email`, `profile`
- **Auth URL, Token URL, API URL:** from the table above

![Generic OAuth settings](screenshots/10-grafana-generic-oauth-settings.png)

After saving, it showed **Enabled**, and a **"Sign in with Authentik"** button appeared on the Grafana login page.

![Enabled](screenshots/11-grafana-generic-oauth-enabled.png)
![New login button](screenshots/12-grafana-login-sign-in-with-authentik.png)

## Part 4: Give people different access with groups

At this point everyone who logged in would get the same access. I wanted access to depend on each person's job instead, so I created two groups in Authentik and put one test user in each:

| Test user | Group | Grafana role I expected |
|---|---|---|
| Marc Spector | Grafana Admins | **Admin** (full control) |
| Illyana Rasputin | Grafana Editors | **Editor** (can make and change dashboards) |
| WP Tester | *(no group)* | **Viewer** (look only) |

![Creating a group](screenshots/13-authentik-new-group.png)
![The two groups](screenshots/14-authentik-groups.png)
![Admin group member](screenshots/15-grafana-admins-member.png)
![Editor group member](screenshots/16-grafana-editors-member.png)

Next I told Grafana how to turn a group into a role. In Grafana's **User mapping** section there's a box called **Role attribute path**, and I put this rule in it:

```
contains(groups[*], 'Grafana Admins') && 'Admin' || contains(groups[*], 'Grafana Editors') && 'Editor' || 'Viewer'
```

In plain English:
- If you're in **Grafana Admins**, you're an **Admin**.
- Otherwise, if you're in **Grafana Editors**, you're an **Editor**.
- Otherwise, you're a **Viewer**.

![Role attribute path](screenshots/17-grafana-role-attribute-path.png)

## Part 5: Testing

### Test 1: Admin (Marc Spector) ✅
I clicked "Sign in with Authentik," logged in, set up MFA, and approved the consent screen. I landed in Grafana as an **Admin** and was able to create a dashboard.

![Authentik login](screenshots/18-test-admin-authentik-login.png)
![MFA setup (QR code blurred)](screenshots/19-test-admin-mfa-enrollment.png)
![Consent screen](screenshots/20-test-admin-consent.png)
![Creating a dashboard](screenshots/21-test-admin-create-dashboard.png)

### Test 2: Editor (Illyana Rasputin) ⚠️ then ✅
She logged in fine, but she showed up as a **Viewer** and couldn't create or edit dashboards. That wasn't what I expected.

![MFA setup (QR code blurred)](screenshots/22-test-editor-mfa-enrollment.png)
![She could only view](screenshots/23-issue-editor-landed-as-viewer.png)

**What went wrong:** I had a typo in my rule. It said `'Grafana Edit ors'` with an extra space. Since that didn't match the real group name, the rule skipped to the last option, `'Viewer'`.

![The typo](screenshots/24-issue-role-path-typo.png)

**How I fixed it:** I changed it to `'Grafana Editors'`, saved, and had her log in again. She was now an **Editor** and could create and edit dashboards.

![Fixed](screenshots/25-fix-editor-can-create-dashboards.png)

> 💡 **Lesson learned:** The login worked, so at first it looked like everything was fine. The problem was with *permissions*, not with logging in. Group names in the rule have to match **exactly**, spaces included.

### Test 3: User with no group (WP Tester) ✅
This user wasn't in either group, so they should get the lowest access. They landed as a **Viewer** with no create or edit options, which is what I wanted.

![Login](screenshots/26-test-viewer-authentik-login.png)
![No edit options](screenshots/27-test-viewer-no-edit-rights.png)
![Role shows Viewer](screenshots/28-test-viewer-org-role.png)

### Double-checking in Grafana
Finally, I logged in with Grafana's own built-in admin account (not through Authentik) and opened **Administration → Users → Organization users**. All three test users were there with the right roles: Admin, Editor, and Viewer. (Logins and emails are blurred.)

![Grafana local admin login](screenshots/29-verify-local-admin-login.png)
![All three roles correct](screenshots/30-verify-org-users-roles.png)

---

## What I learned

- **Provider vs. application:** The provider is the technical login setup. The application is the thing users see and the place where access gets controlled.
- **Logging in and permissions are two different things.** My typo didn't stop anyone from logging in. It quietly gave someone less access than they should have had.
- **A safe default is important.** Anyone not in a group only gets Viewer access.
- **Check the actual app.** Looking at the users list in Grafana showed what access each person really got.
- **Keep a backup admin login.** If Authentik ever breaks, I can still get into Grafana with its local admin account.
- **Protect secrets.** Client secrets and MFA QR codes should never end up in notes, screenshots, or GitHub.

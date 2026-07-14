# Deploying a Private GitHub Pages Site with Cloudflare Access

Reusable knowledge base documenting how the `BAKEET` repository was published to GitHub Pages and then restricted to a specific list of email addresses using Cloudflare Access (Zero Trust). Follow these steps to repeat the setup for another repo/domain.

---

## 1. Push a local folder to a new GitHub repo

```bash
cd "/path/to/project"
git init
git branch -M main
git remote add origin https://github.com/<user>/<repo>.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

## 2. Add a README

Create `README.md` describing the project structure, then commit and push:

```bash
git add README.md
git commit -m "Add project README"
git push origin main
```

## 3. Enable GitHub Pages (default domain)

1. GitHub repo → **Settings** → **Pages**
2. **Source**: `Branch: main`, **Folder**: `/ (root)`
3. Save. Site becomes available at:
   ```text
   https://<user>.github.io/<repo>/
   ```

### Make the page load automatically (avoid landing on README)

Add an `index.html` that redirects to the actual report/page:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="refresh" content="0; url=bakeet_results.html" />
    <title>BAKEET</title>
  </head>
  <body>
    <p>Redirecting to <a href="bakeet_results.html">bakeet_results.html</a>...</p>
  </body>
</html>
```

Commit and push it — GitHub Pages picks it up automatically as the entry point.

> Note: the GitHub **repository** page (`github.com/<user>/<repo>`) will always show the README — that is normal and separate from the Pages site.

## 4. Removing an inherited custom domain (if the account-level Pages site has one)

Symptom: the project Pages URL (`https://<user>.github.io/<repo>/`) 301-redirects to an old custom domain (e.g. `http://olddomain.com/<repo>/`).

Cause: the personal `https://github.com/<user>/<user>.github.io` repository has a `CNAME` file setting an account-wide custom domain that GitHub Pages applies to *all* project sites.

Fix: delete the `CNAME` file from the `<user>.github.io` repo (via GitHub UI, git, or the Contents API). After deletion:

```bash
curl -H "Authorization: Bearer <token>" -H 'Accept: application/vnd.github+json' \
  https://api.github.com/repos/<user>/<user>.github.io/pages
```

should show `"cname": null` and `"html_url": "https://<user>.github.io/"`.

## 5. GitHub Pages cannot be made "private" on a personal account

- Restricting Pages visibility to specific GitHub accounts requires **GitHub Enterprise**.
- For a personal/free account, the practical alternative is putting the custom domain behind **Cloudflare Access** with an email allowlist (steps below).

## 6. Point a domain to Cloudflare

1. Create a free Cloudflare account, add the domain (e.g. `example.com`).
2. Cloudflare scans existing DNS records (keep MX/TXT email records untouched).
3. At the domain registrar (e.g. Namecheap):
   - Domain List → **Manage** → **Nameservers** → switch to **Custom DNS**
   - Enter the two nameservers Cloudflare provides (e.g. `rocco.ns.cloudflare.com`, `violet.ns.cloudflare.com`)
   - Remove the old registrar nameservers, save
   - Ensure **DNSSEC is off** at the registrar before/while switching
4. Wait for Cloudflare to mark the domain **Active** (minutes to ~24h). Verify propagation:
   ```bash
   dig +short NS example.com
   ```

## 7. Create a subdomain CNAME pointing to GitHub Pages

Cloudflare → **DNS** → **Records** → **Add record**:

- Type: `CNAME`
- Name: `report` (or any subdomain)
- Target: `<user>.github.io`
- Proxy status: **DNS only** at first (needed for GitHub's DNS check)

Verify:
```bash
dig +short CNAME report.example.com
```

## 8. Add the custom domain in GitHub Pages

1. Repo → **Settings** → **Pages** → **Custom domain**
2. Enter `report.example.com`, Save
3. Wait for **DNS check successful**
4. Enable **Enforce HTTPS**

## 9. Re-enable Cloudflare proxy

Go back to the Cloudflare DNS record for `report` and toggle **Proxy status** to **Proxied** (orange cloud). This is required for Cloudflare Access to intercept requests.

## 10. Add the One-Time PIN identity provider

New Cloudflare Zero Trust orgs default to the **Cloudflare** login method (requires a Cloudflare account to sign in — not suitable for external guests). Add OTP instead:

1. Zero Trust dashboard → **Integrations** → **Identity providers**
2. **Add an identity provider** → **One-time PIN** → Save
3. Optionally delete the default **Cloudflare** identity provider (⋯ menu → Delete) if no other app needs it — this does not affect your own Cloudflare account login, only Access app login options.

## 11. Create the Access application

Zero Trust → **Access controls** → **Applications** → **Add an application** → **Self-hosted**:

- **Public hostname**
  - Subdomain: `report`
  - Domain: `example.com`
- **Access policy**
  - Name: `Allowed emails`
  - Action: `Allow`
  - Include rule: `Emails` → add each permitted address
- **Authentication** tab: leave "Accept all available identity providers" on (will include One-time PIN once added), or explicitly select **One-time PIN**
- **Session Duration**: how long a verified user stays logged in before re-verifying (e.g. `24 hours`, or shorter like `1 hour`/`6 hours` for tighter security)
- Click **Create**

## 12. Test

Open an incognito window and visit `https://report.example.com`:

- Login page should show an **email box** + **Send login code** (not just a "Cloudflare" button — that would mean OTP isn't enabled/selected yet).
- Enter an allowed email → receive a one-time PIN by email → enter it → redirected to the site.
- Emails not on the allow list are told "no access" and never receive a code.
- No Cloudflare account/signup is required for allowed users — they only need access to their email inbox.

---

## Key facts / gotchas

- GitHub Pages project sites inherit the custom domain of the account's `<user>.github.io` repo unless that repo's `CNAME` file is removed.
- A repository's Pages config (`cname` field via API) is separate from the account-level Pages config — check both when debugging redirects.
- Personal GitHub accounts cannot make a Pages site private; use Cloudflare Access (or similar auth proxy) in front of a custom domain instead.
- Cloudflare Access "One-time PIN" requires no signup for end users — just email access.
- Keep the DNS record **proxied** (orange cloud) for Cloudflare Access to work; **DNS only** bypasses Cloudflare's edge (used temporarily only for GitHub's domain verification step).

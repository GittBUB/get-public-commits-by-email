# Identify Public Activities by Email

This repo contains two python scripts that require a PAT or GitHub App token:
`get-commits-by-email`
`get-public-repos-by-email` 


## Setting Up a Personal Access Token (PAT)

Both scripts authenticate to the GitHub API using a **classic Personal Access Token (PAT)**. Fine-grained tokens are not recommended here because these scripts search across all of public GitHub rather than a specific set of repositories.

### Steps to create a classic PAT

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**.
   Direct link: https://github.com/settings/tokens
2. Click **Generate new token (classic)**.
3. Give it a descriptive **Note** (e.g. `public-commit-search`).
4. Set an **Expiration** appropriate for your use case.
5. Select the **scopes** listed below for the script you are running.
6. Click **Generate token** and copy the value — it will not be shown again.
7. Paste the token into the `TOKEN = "ghp_xxx"` line at the top of the script.

### Required scopes by script

| Script | Required scopes | Why |
|---|---|---|
| `get-commits-by-email` | `public_repo` | Authenticates search-commits API requests and raises the rate limit to ~30 req/min |
| `get-public-repos-by-email` | `public_repo`, `read:user` | `public_repo` for commit/repo search; `read:user` for the `/users/{login}` profile endpoint |

> **Note:** The search endpoints work without any scopes, but an unauthenticated request is rate-limited to 10 requests/minute and results may be degraded. Always use a PAT.

### Security reminders

* **Never commit your PAT to source control.** The placeholder `ghp_xxx` must be replaced at runtime only.
* Store the token in an environment variable or a secrets manager and read it in the script (e.g. `TOKEN = os.environ["GITHUB_TOKEN"]`) rather than hard-coding it.
* Revoke tokens you no longer need at https://github.com/settings/tokens.

---

## Setting Up a GitHub App (Alternative to PAT)

GitHub Apps provide better security and higher rate limits than PATs. Use this approach for production or automated workflows.

### Why use a GitHub App?

| Feature | PAT | GitHub App |
|---------|-----|------------|
| Rate limit | 5,000 req/hr | 15,000 req/hr (with installation token) |
| Expiration | Configurable | Installation tokens auto-expire (1 hr) |
| Scope | User-wide | Granular per-installation |
| Audit trail | Limited | Full audit log in org settings |

### Step 1: Create the GitHub App

1. Go to **GitHub → Settings → Developer settings → GitHub Apps**
   Direct link: https://github.com/settings/apps
2. Click **New GitHub App**
3. Fill in the required fields:
   - **GitHub App name**: e.g. `commit-search-app` (must be unique across GitHub)
   - **Homepage URL**: Can be your repo URL or any valid URL
   - **Webhook**: Uncheck "Active" (not needed for these scripts)
4. Set **Permissions**:
   - Under **Repository permissions**:
     - `Contents`: Read-only (for commit search)
     - `Metadata`: Read-only (automatically selected)
   - Under **Account permissions**:
     - `Email addresses`: Read-only (for user search)
5. Under **Where can this GitHub App be installed?**, select:
   - "Only on this account" for personal use
   - "Any account" if you need to install it on organizations
6. Click **Create GitHub App**

### Step 2: Generate a Private Key

1. After creating the app, scroll down to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file will download — **store this securely**
4. Note your **App ID** (shown at the top of the app settings page)

### Step 3: Install the App

1. In your GitHub App settings, click **Install App** in the left sidebar
2. Select the account or organization where you want to install it
3. Choose repository access:
   - "All repositories" — for searching across all repos
   - "Only select repositories" — for limited scope
4. Click **Install**
5. Note the **Installation ID** from the URL after installation:
   `https://github.com/settings/installations/INSTALLATION_ID`

### Step 4: Generate Installation Access Tokens

GitHub Apps authenticate in two steps:
1. Create a JWT signed with your private key
2. Exchange the JWT for an installation access token

#### Python example:

```python
import jwt
import time
import requests

# Your GitHub App credentials
APP_ID = "123456"
PRIVATE_KEY_PATH = "path/to/your-app.pem"
INSTALLATION_ID = "12345678"

def get_installation_token():
    # Read private key
    with open(PRIVATE_KEY_PATH, "r") as f:
        private_key = f.read()
    
    # Create JWT (valid for 10 minutes max)
    now = int(time.time())
    payload = {
        "iat": now - 60,        # Issued at (60s ago to handle clock drift)
        "exp": now + 600,       # Expires in 10 minutes
        "iss": APP_ID           # Issuer = App ID
    }
    jwt_token = jwt.encode(payload, private_key, algorithm="RS256")
    
    # Exchange JWT for installation token
    headers = {
        "Authorization": f"Bearer {jwt_token}",
        "Accept": "application/vnd.github+json"
    }
    url = f"https://api.github.com/app/installations/{INSTALLATION_ID}/access_tokens"
    response = requests.post(url, headers=headers)
    response.raise_for_status()
    
    return response.json()["token"]

# Use the token in your scripts
TOKEN = get_installation_token()
```

#### Required dependency:

```bash
pip install PyJWT cryptography
```

### Step 5: Use the Token in Scripts

Replace the `TOKEN = "ghp_xxx"` line in the scripts with:

```python
import os
TOKEN = os.environ.get("GITHUB_TOKEN") or get_installation_token()
```

Or generate the token once and set it as an environment variable:

```bash
export GITHUB_TOKEN=$(python3 -c "from your_auth_module import get_installation_token; print(get_installation_token())")
python3 get-public-repos-by-email
```

### Security reminders for GitHub Apps

* **Never commit your private key (`.pem` file) to source control**
* Store the private key in a secrets manager (AWS Secrets Manager, Azure Key Vault, etc.)
* Installation tokens expire after 1 hour — regenerate as needed
* Regularly rotate your private key via the GitHub App settings

### Documentation

- Creating a GitHub App: https://docs.github.com/en/apps/creating-github-apps
- Authenticating as a GitHub App: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app
- Installation access tokens: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app

---

## Running the Scripts

### Prerequisites

- Python 3.x installed
- A GitHub Personal Access Token (see above)

---

### get-commits-by-email

Search GitHub commits by a list of author/committer emails, optionally within a date range.

#### Steps

1. **Create a virtual environment and install dependencies:**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install requests
   ```

2. **Configure the script** — edit `get-commits-by-email` and set:
   - `TOKEN` — your GitHub PAT
   - `EMAILS` — list of email addresses to search
   - `SCOPE` — (optional) limit to a repo or org, e.g. `"repo:owner/name"` or `"org:myorg"`
   - `DATE_RANGE` — (optional) date filter, e.g. `"2025-01-01..2025-06-30"`

3. **Run the script:**
   ```bash
   source .venv/bin/activate
   python3 get-commits-by-email
   ```

4. **Output:** Results are saved to `commits_by_email.csv`

#### Documentation
- Search commits endpoint: https://docs.github.com/en/rest/search/search#search-commits
- Commit search qualifiers: https://docs.github.com/en/search-github/searching-on-github/searching-commits

> **Note:** When a user enables "Keep my email address private," GitHub assigns them a noreply address like `12345678+username@users.noreply.github.com`. If they also enable "Block command line pushes that expose my email," their commits use that noreply address — so searching their real email will return nothing for those commits.

---

### get-public-repos-by-email

Discover GitHub accounts linked to email addresses, then list their public repos and gists outside your company orgs.

#### Steps

1. **Create a virtual environment and install dependencies:**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install requests
   ```

2. **Configure the script** — edit `get-public-repos-by-email` and set:
   - `TOKEN` — your GitHub PAT
   - `EMAILS` — list of email addresses to search
   - `COMPANY_ORGS` — list of org names to treat as "internal" (repos in these orgs are excluded)
   - `INCLUDE_FORKS` — set to `True` to include forked repos

3. **Run the script:**
   ```bash
   source .venv/bin/activate
   python3 get-public-repos-by-email
   ```

4. **Output:** Results are saved to `personal_accounts.csv` with columns:
   - `company_email` — the searched email
   - `github_login` — discovered GitHub username
   - `profile_name` — user's display name
   - `profile_company` — user's company field
   - `kind` — `repo` or `gist`
   - `item` — repo/gist name or description
   - `url` — link to the repo/gist

Documentation for the API used:
- Search commits:  https://docs.github.com/en/rest/search/search#search-commits
- Commit syntax:   https://docs.github.com/en/search-github/searching-on-github/searching-commits
- Search users:    https://docs.github.com/en/rest/search/search#search-users
- List user repos: https://docs.github.com/en/rest/repos/repos#list-repositories-for-a-user
- Get a user:      https://docs.github.com/en/rest/users/users#get-a-user

#### Note on Gist Visibility

GitHub has two types of gists:

| Type | Discoverable | API Accessible | Description |
|------|--------------|----------------|-------------|
| **Public** | Yes | Yes | Listed on user's profile, searchable, returned by `GET /users/{login}/gists` |
| **Secret** | No | No* | Not listed anywhere, only accessible via direct URL |

\* Secret gists are not truly private — anyone with the URL can view them. However, the API endpoint `GET /users/{login}/gists` only returns **public gists**. Secret gists cannot be discovered through the API without knowing the exact gist ID.

This means if a user has secret gists, they will **not** appear in the script's output.

# Identify Public Activities by Email

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

### get-public-commits-by-email
Leverage GitHub APIs to identify public commits by email address.

Just fill in TOKEN, EMAILS, and optionally SCOPE, then run python search_commits_by_email.py.

Documentation for the API used:
- Search commits endpoint: https://docs.github.com/en/rest/search/search#search-commits
- Commit search qualifiers (including author-date / committer-date ranges): https://docs.github.com/en/search-github/searching-on-github/searching-commits

NOTE: When a user enables "Keep my email address private," GitHub gives them a noreply address like 12345678+username@users.noreply.github.com (or the older username@users.noreply.github.com). If they also enable "Block command line pushes that expose my email," their commits get authored under that noreply address, not their real one. So author-email:their.real@email.com will return nothing for those commits — the real email was never written into the commit object.

### get-public-repos-by-email
Leverage GitHub APIS to identify public repos and gists by email address.

Just fill in TOKEN, EMAILS, and COMPANY_ORG, then run python get-public-repos-by-email.py.

Documentation for the API used:
- Search commits:  https://docs.github.com/en/rest/search/search#search-commits
- Commit syntax:   https://docs.github.com/en/search-github/searching-on-github/searching-commits
- Search users:    https://docs.github.com/en/rest/search/search#search-users
- List user repos: https://docs.github.com/en/rest/repos/repos#list-repositories-for-a-user
- Get a user:      https://docs.github.com/en/rest/users/users#get-a-user

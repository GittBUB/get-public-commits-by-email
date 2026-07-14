# Identify Public Activities by Email

This repo contains the following scripts:

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

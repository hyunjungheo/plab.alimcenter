# Public Repo Security Remediation Plan

Date: 2026-04-24
Repository: `plab.alimcenter`
Status: Action required

## Purpose

This document records the items that must be addressed because the repository is public and currently contains operational details that should not remain publicly exposed.

Sensitive values are intentionally omitted or redacted in this document. Refer to the file paths below, not to copied secret values.

## Confirmed Findings

| Severity | Location | Issue | Required action |
| --- | --- | --- | --- |
| Critical | `api/push.js` | Hardcoded external API key and direct proxy to internal query endpoint | Revoke key immediately, remove from code, replace with server-side env var, add authentication and origin restrictions |
| Critical | `index.html`, `api/push.js` | Public client can send arbitrary SQL-shaped requests through `/api/push` | Remove arbitrary query passthrough, replace with fixed server-side queries or protected internal-only API |
| High | `index.html` | Hardcoded employee email addresses, internal domain, and direct contact details | Remove or replace with non-personal aliases/configuration |
| Medium | `index.html` | Firebase project identifiers and direct Firestore read/write usage from browser | Review Firestore Security Rules and ensure public clients cannot read/write operational data beyond intended scope |
| Medium | Git history | Exposed blobs remain in commit history after file changes unless history is rewritten | Rewrite history, force-push, and invalidate old clones/forks where possible |
| Medium | Git commit metadata | Commit author email is publicly visible in history | Decide whether author email rewrite is required for this repository |

## Immediate Actions

### 1. Revoke exposed credentials

- Revoke the key currently referenced in `api/push.js`.
- Issue a new credential only after the endpoint design is changed.
- Assume the exposed key is compromised because the repository is public.

### 2. Remove direct SQL proxy behavior

Current behavior allows the public frontend to send query text to `/api/push`, which then forwards it to an internal query service.

Required changes:

- Stop accepting arbitrary query text from the browser.
- Replace with fixed, server-defined queries for each dashboard/report use case.
- Require authenticated access on the server side.
- Restrict allowed origins instead of `Access-Control-Allow-Origin: *`.
- Add request logging, rate limiting, and abuse monitoring.

### 3. Remove public exposure of employee identifiers

Required changes:

- Remove hardcoded employee email addresses from `index.html`.
- Remove direct personal contact text from the login/help UI.
- Replace personal addresses with a team alias or a neutral support channel if disclosure is necessary.
- Avoid embedding internal domain-based authorization rules directly in public client code when possible.

### 4. Review Firebase/Firestore exposure

Firebase config values are not secrets by themselves, but the current browser code directly reads and writes Firestore documents.

Required changes:

- Audit Firestore Security Rules immediately.
- Confirm that unauthenticated or overly broad authenticated access is not possible.
- Limit browser-accessible collections to the minimum necessary.
- Move sensitive writes behind a trusted backend if the data is operational or internal.

### 5. Rewrite public Git history

Removing values from the current files is not enough. The exposed content remains reachable in existing commits.

Required actions:

- Rewrite Git history to remove exposed secrets and internal identifiers from blobs.
- Force-push rewritten history to `main`.
- Invalidate old local clones and ask collaborators to re-clone.
- Review forks, cached archives, deployment logs, and external indexing where feasible.

## Suggested History Rewrite Workflow

Use `git filter-repo` or BFG. `git filter-repo` is preferred for precise text replacement and optional author metadata rewrite.

High-level process:

1. Create a backup mirror of the repository.
2. Prepare a replacement file for secret and identifier patterns.
3. Rewrite history locally.
4. Verify with secret scanning against `HEAD` and `git rev-list --all`.
5. Force-push rewritten history.
6. Rotate credentials again if there is any doubt about reuse or additional exposure.

Example replacement categories:

- Exposed API key values
- Personal email addresses embedded in source
- Internal endpoint hostnames if they should not be public

If commit author email must also be rewritten, include commit metadata rewrite as part of the same history rewrite window.

## Verification Checklist

- `api/push.js` no longer contains hardcoded credentials
- Public client no longer sends arbitrary query text to a privileged backend
- CORS is restricted to intended origins
- Firestore rules are reviewed and documented
- Employee personal addresses are removed from public source
- Secret scan passes on working tree
- Secret scan passes across Git history
- `main` is force-pushed after history rewrite
- Collaborators are notified to refresh local clones

## Prevention Controls

- Store secrets only in deployment environment variables or a managed secret store
- Add automated secret scanning in CI with a tool such as `gitleaks`
- Add a pre-commit or pre-push secret scan locally
- Add a public-repo review checklist before pushing operational code
- Do not expose internal data access patterns directly from browser code
- Prefer backend authorization over client-side identity gating for privileged operations

## Ownership Recommendation

- Backend/API changes: owner of `api/push.js` and deployment configuration
- Frontend cleanup: owner of `index.html`
- History rewrite and credential rotation: repository administrator
- Firestore rules review: Firebase project owner

## Notes

- This document is intentionally limited to remediation guidance.
- Do not paste the exposed secret values into follow-up tickets, PR descriptions, or new documentation.

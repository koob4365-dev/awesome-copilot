# Traffic Reporting operations

The workflow `.github/workflows/traffic-reporting.yml` fetches the repository's
page views for the last 14 days, prints the statistics, then sends the same JSON
to the configured HTTPS reporting endpoint. It runs daily and supports manual
`workflow_dispatch` runs.

## Required configuration

| Name | GitHub configuration | Requirement |
| --- | --- | --- |
| `REPORTING_TOKEN` | Repository Actions secret | A valid fine-grained PAT for this repository with `Administration: read`; `Metadata: read` is included automatically. The token owner must have write access to the repository. |
| `REPORTING_POST_URL` | Repository Actions variable | The intended reporting service's full HTTPS URL. |
| `REPORTING_API_KEY` | Repository Actions secret | The API key accepted by that reporting service through the `X-API-KEY` header. |

The job does not select a GitHub environment. An environment-only secret or
variable is therefore not available to this job. Forks need their own
configuration; upstream repository secrets are not copied into a fork.

## Authentication choice and safe setup

Keep the existing `REPORTING_TOKEN` contract for this single personal repository.
Use a fine-grained PAT restricted to `koob4365-dev/awesome-copilot` with only
`Administration: read` and the automatically included `Metadata: read`. Do not
request repository write permissions or a broad classic `repo` scope.

The traffic API requires `Administration: read`. This permission is not available
in the workflow `permissions` map for the built-in `GITHUB_TOKEN`, so falling back
to `${{ github.token }}` or adding `contents: write` does not solve traffic access.
A GitHub App installation token with the same repository restriction and
`Administration: read` is a supported alternative for an established, long-lived
integration. Reuse an appropriately restricted existing App when available;
creating an App, maintaining its private key, and adding token generation is a
separate migration, not required for this credential repair.

1. In the repository's **Settings > Secrets and variables > Actions**, check for
   the exact names above. Check the repository Secrets tab and Variables tab,
   rather than Codespaces, Dependabot, or an unreferenced environment.
2. If `REPORTING_TOKEN` already exists, check the token's expiry/revocation,
   resource owner, selected repository, and permissions in the owner's token
   settings. GitHub does not reveal stored secret values. A configured secret
   name alone does not prove a valid token.
3. If a replacement is needed, create the restricted fine-grained PAT in the
   owner's GitHub token settings, choose an expiration under the owner's policy,
   and record its renewal date privately. Paste it directly into the repository
   Actions secret form and submit there. Never paste it into chat, code, issues,
   command arguments, or logs. New credentials and their submission should be
   completed by the owner in GitHub.
4. A locally authenticated GitHub CLI can alternatively prompt for secret values
   without putting them in a shell command or shell history:

   ```sh
   gh secret set REPORTING_TOKEN --repo koob4365-dev/awesome-copilot
   gh secret set REPORTING_API_KEY --repo koob4365-dev/awesome-copilot
   ```

5. Configure `REPORTING_POST_URL` as a repository Actions **variable**, and the
   receiving service's key as `REPORTING_API_KEY`. Confirm the destination belongs
   to the intended service before sending traffic data. Do not print the key or
   include it in the URL.

## Verification and failure diagnosis

After fixing secrets, use **Actions > Traffic Reporting > Run workflow** and
select the branch containing the intended workflow version. Re-running an old
run keeps its original commit and does not test later workflow edits. A rerun of
run `40643570829` can verify a credentials-only repair against the old source.

Verify all three steps, not just the final job conclusion:

1. **Fetch GitHub traffic statistics** succeeds and returns valid traffic JSON.
2. **Output traffic data to console** succeeds with the same JSON.
3. **Send traffic data to reporting endpoint** succeeds with an HTTP 2xx status.
   If access to receiver logs is available, confirm receipt there as well; an HTTP
   success establishes endpoint acceptance, not downstream processing.

The workflow reports missing secrets without logging their values. GitHub CLI
also emits its API error when a nonempty token is rejected. A 401 generally means
invalid/expired/revoked credentials; a 403 or 404 requires checking repository
selection and permissions as well as GitHub's error message. Do not assume an
expired token from an empty `GH_TOKEN`: the empty value fails before an API call.
Report POST failures include the HTTP status without printing the response body,
API key, or destination. HTTP 000 indicates no HTTP response was obtained; inspect
connectivity, DNS, TLS, and timeout behavior.

## Incident record: 2026-10-28

- Run 6: `40643570829`, created `2026-10-28T06:23:37Z`, failed at fetch; the log
  shows empty `GH_TOKEN` and GitHub CLI exit code 4. Output and report steps were
  skipped.
- Run 5: `40457060293`, created `2026-10-27T05:44:48Z`, also failed. Both runs used
  commit `2d4b998d2df28917074ee33db45c9601484c72ad`.
- Confirmed cause at the first failing boundary: `secrets.REPORTING_TOKEN` did not
  supply a nonempty value to the job. Missing configuration or inaccessible
  secret scope remains to be distinguished. Token expiration was not tested.
- Source changes add explicit missing-token and missing-report-key checks, an
  actionable traffic API error, and report HTTP failure diagnostics. Scheduling,
  manual triggering, traffic JSON, console output, HTTPS enforcement, POST
  headers, timeout, and retry behavior are retained.
- Offline validation passed: YAML parsing, Bash syntax for all three steps,
  reproduction of the original exit 4, rejection before network invocation for
  empty token/URL/key or non-HTTPS URL, rejected-token diagnostics, successful
  fetch/output with unchanged JSON, and reporting responses 201, 401, 302 and
  transport failure 000. CLI/HTTP boundaries used test doubles; no real service
  was called. The repository's line-ending script also passed. The full resource
  generation build was not run because only the affected files were materialized.
- Live configuration remains unverified: the GitHub connector does not expose
  Secrets/Variables APIs, and this environment blocks the GitHub settings page.
  No token or reporting key has been created, read, replaced, or exposed.
- End-to-end verification is pending safe configuration by the repository owner.
  No redundant rerun was started while the known credential gap remains.

## GitHub references

- [Repository traffic API and required permissions](https://docs.github.com/en/rest/metrics/traffic#get-page-views)
- [Workflow token permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions)
- [Using Actions secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)
- [Managing fine-grained personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

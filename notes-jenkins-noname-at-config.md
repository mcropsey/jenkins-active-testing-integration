# VNotes → Jenkins → Akamai API Security Active Testing (mikec-lab)

Step-by-step record of how the VNotes CI/CD integration was moved from the old
`michaelc-lab` tenant to **`mikec-lab`**, and how to rebuild it from scratch.

_Last verified: 2026-11-18. Build #51 detected BOLA (HIGH) on the vulnerable app._

---

## 1. Topology

| Piece | Value |
|---|---|
| App (public) | https://notes.cropseyit.com (Swagger: `/docs`, spec: `/openapi.json`) |
| Deploy host | `192.168.1.98`, `/home/mcropsey/vnotes`, `docker compose`, port 8000 |
| Git repo | https://github.com/mcropsey/vnotes (branch `main`, `Jenkinsfile` at root) |
| Jenkins | https://jenkins.cropseyit.com = `http://192.168.1.100:8080` (Docker: `jenkins` + `jenkins-docker` dind) |
| Jenkins job | `vnotes`: Pipeline from SCM, `*/main`, GitHub push trigger (webhook) |
| AT tenant | https://mikec-lab.nonamesec.com (backend version 3.73.6 at setup time) |
| AT app | **VNotes**: `ab930b27-1a22-4acf-b5bf-0add38e87a62` |
| AT env (CI) | **Jenkins CI**: `2f05f900-45ce-4f65-b213-a0838416d548` (type **cicd**) |
| Test profile | Default API Security Tests: `892b51de-67cf-4328-beea-3ade61bcdeb2` |
| Scanner image | `us-central1-docker.pkg.dev/noname-artifacts/nns-docker/active-cli:<backend version>` |

Pipeline flow on every push to `main`:

```
GitHub push ─► Jenkins vnotes job
   ├─ Checkout
   ├─ Ship code to 192.168.1.98 (tar over ssh)
   ├─ Build & Deploy (docker compose build && up -d)
   ├─ Wait for app (GET https://notes.cropseyit.com/health)
   └─ Active Scan
        1. SA client id/secret ─► POST /auth/token ─► accessToken (ACTIVE_API_TOKEN)
        2. akamai/refresh-auths.sh: log in alice/bob/charlie/admin, PUT fresh
           "Authorization: Bearer <jwt>" into the Jenkins CI env's 4 authentications
        3. docker login GAR + pull active-cli:<GET /active/backend/version>
        4. active-cli scan --env-id <Jenkins CI> --test-group-id ... --severity-threshold
   Result: SUCCESS = no findings ≥ threshold · UNSTABLE = findings ≥ threshold · FAILURE = pipeline/scan error
```

---

## 2. Test users (VNotes demo accounts)

| User | Password | AT role |
|---|---|---|
| alice | alice123 | user |
| bob | bob12345 | user |
| charlie | charlie1 | user |
| admin | admin123 | admin |

Login is OAuth2 password **form** style (not JSON):

```bash
curl -X POST https://notes.cropseyit.com/api/auth/login -d 'username=alice&password=alice123'
# {"access_token":"eyJ...","token_type":"bearer"}   <- JWT valid 24h
```

The JWTs expire after 24h, so static tokens saved in AT go stale. The
pipeline re-mints them every build (step 6).

---

## 3. Create a service account in mikec-lab (UI, one time)

The CI/CD scan token **and** the scanner registry pull key both come from a
platform service account.

1. https://mikec-lab.nonamesec.com → **Settings → User Management → Service Accounts → Create**
2. Name `jenkins-vnotes`, Role **Admin** (or any role with Active Testing access), long Token TTL (e.g. 365 days).
3. Copy the **Client ID** and **Client Secret** right away. The secret is shown only once; if you lose it, regenerate it.

Verify:

```bash
curl -sS -X POST https://mikec-lab.nonamesec.com/auth/token -H 'Content-Type: application/json' \
  -d '{"grant_type":"client_credentials","client_id":"<ID>","client_secret":"<SECRET>"}'
# HTTP 201 {"accessToken":"..."}
```

With that token (`$T`), get the scanner registry credentials. This is the same call
the UI's "CI/CD Parameters" panel makes:

```bash
curl -sS -H "Authorization: Bearer $T" https://mikec-lab.nonamesec.com/api/active/v2/cicd/parameters
# {"data":{"registry_user":"_json_key_base64","registry_password":"<base64 GCP key>",
#          "registry_url":"us-central1-docker.pkg.dev/noname-artifacts/nns-docker"}}
```

---

## 4. Active Testing: application and environments

### 4a. Application (done through the Active Testing MCP server, or the UI)
- **Applications → New** · Name `VNotes` · Base URL `https://notes.cropseyit.com`
- Source: OpenAPI by URL `https://notes.cropseyit.com/openapi.json`
- This also auto-creates a **Default** (deployment) environment. It was **deleted** here because all scans go through Jenkins CI. Manual UI scans can target Jenkins CI too.

### 4b. CI/CD environment "Jenkins CI"
It must be **type `cicd`**. MCP `create_environment` always creates `deployment`, and the
CI/CD scan-configuration endpoints reject that type ("is not a CI/CD environment").
In the UI, use **VNotes → Settings → CI/CD → Jenkins → New environment**. By API
(the same call the UI wizard makes), using the service-account token:

```bash
curl -sS -X POST -H "Authorization: Bearer $T" -H 'Content-Type: application/json' \
  https://mikec-lab.nonamesec.com/api/active/v2/frontend/application/ab930b27-1a22-4acf-b5bf-0add38e87a62/environment \
  -d '{
    "app_id":"ab930b27-1a22-4acf-b5bf-0add38e87a62",
    "name":"Jenkins CI", "type":"cicd",
    "base_url":"https://notes.cropseyit.com",
    "sources":[{"name":"VNotes OpenAPI","type":"openapi_sdk","import_type":"url",
                "import_data":"https://notes.cropseyit.com/openapi.json","auth_mappings":[]}],
    "auths":[
      {"name":"alice","role":"user","mappings":[{"type":"header","key":"Authorization","value":"Bearer <jwt>"}]},
      {"name":"bob","role":"user","mappings":[{"type":"header","key":"Authorization","value":"Bearer <jwt>"}]},
      {"name":"charlie","role":"user","mappings":[{"type":"header","key":"Authorization","value":"Bearer <jwt>"}]},
      {"name":"admin","role":"admin","mappings":[{"type":"header","key":"Authorization","value":"Bearer <jwt>"}]}
    ],
    "test_group_id":"892b51de-67cf-4328-beea-3ade61bcdeb2",
    "worker_pool":"default_pool", "timeout":60
  }'
```

**The auth names must stay exactly `alice`, `bob`, `charlie`, `admin`.** The refresh script matches on them.

### 4c. CI/CD scan configuration (Jenkins platform)
UI: **Settings → CI/CD → Jenkins → existing env "Jenkins CI"**. Test profile is
*Default API Security Tests*, severity threshold is `high`, timeout is 30 min, verbose is on.
API equivalent:

```bash
curl -sS -X PUT -H "Authorization: Bearer $T" -H 'Content-Type: application/json' \
  https://mikec-lab.nonamesec.com/api/active/v2/frontend/cicd/environments/2f05f900-45ce-4f65-b213-a0838416d548/scan-configuration \
  -d '{"platform":"jenkins","test_profile_id":"892b51de-67cf-4328-beea-3ade61bcdeb2",
       "scan_control_params":{"severityThreshold":"high"},
       "network_performance":{"timeout":30},"logging_config":{"verbose":true,"analyze":false}}'
```

The wizard's generated Jenkins snippet (`POST /api/active/v2/frontend/cicd/jenkins`)
puts the registry key **in plain text** in the Jenkinsfile and uses `CORE_CLI_CLIENT_ID/SECRET`.
We do **not** use it as-is. The key goes into Jenkins credentials, and the scanner
authenticates with the service-account token (`ACTIVE_API_TOKEN`) instead.

---

## 5. Jenkins credentials

**Manage Jenkins → Credentials → System → Global → Add Credentials**, all of kind **Secret text**:

| ID | Value |
|---|---|
| `mikec-at-registry-key` | `registry_password` from `/api/active/v2/cicd/parameters` |
| `mikec-at-sa-client-id` | service account Client ID |
| `mikec-at-sa-client-secret` | service account Client Secret |
| `vnotes-at-users` | `alice:alice123 bob:bob12345 charlie:charlie1 admin:admin123` |

These already existed and are unchanged: `vnotes-deploy-ssh` (deploy key, also used for git checkout) and `github-token`.

The old michaelc-lab credentials (`active-registry-key`, `active-cli-client-id`,
`active-cli-secret`) are no longer referenced by the Jenkinsfile.

Script equivalent (REST, with crumb):

```bash
J=http://192.168.1.100:8080; A='admin:<password>'; CJ=$(mktemp)
CR=$(curl -sS -c $CJ -u $A "$J/crumbIssuer/api/json" | python3 -c 'import sys,json;d=json.load(sys.stdin);print(d["crumbRequestField"]+":"+d["crumb"])')
curl -sS -b $CJ -u $A -H "$CR" -X POST "$J/credentials/store/system/domain/_/createCredentials" \
  --data-urlencode 'json={"":"0","credentials":{"scope":"GLOBAL","id":"mikec-at-sa-client-id","secret":"<value>","description":"...","$class":"org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl"}}'
```

---

## 6. Repository changes (`mcropsey/vnotes`, commit `d6bfa1f` on `main`)

### `akamai/refresh-auths.sh` (new)
Before each scan, it logs in every user in `$AT_USERS` against `$TARGET_URL/api/auth/login`.
It then `PUT`s the fresh JWT into the AT authentication of the same name:

```
GET  /api/active/v2/applications/{APP_ID}/environments/{ENV_ID}/authentications
PUT  /api/active/v2/applications/{APP_ID}/environments/{ENV_ID}/authentications/{authId}
     {"authenticationName":"alice","role":"user",
      "mappings":[{"type":"header","key":"Authorization","value":"Bearer <jwt>"}]}
```

It uses only bash, curl, sed and grep, because the Jenkins container has no python or jq. Tokens are never printed.

### `Jenkinsfile` (rewritten)
- Points at mikec-lab: `ACTIVE_API_URL`, `ACTIVE_BACKEND_URI`, `ACTIVE_TOKEN_URL`, and `APP_ID`/`ENV_ID`/`TEST_GROUP_ID` from the table above.
- Uses the new `mikec-at-*` and `vnotes-at-users` credentials.
- Pulls `active-cli:<backend /version>`. The old cached `3.71.0` fallback and the dead michaelc registry key are gone.
- Logs in to the registry **host** (`us-central1-docker.pkg.dev`), not the repo path. Otherwise `docker pull` can't find the stored credential.
- New **Wait for app** stage that polls `/health` before scanning.
- Build parameters: `SEVERITY_THRESHOLD` (high | medium | low | info | none, default `high`) and `SKIP_DEPLOY` (scan only).
- Exit handling: a threshold breach marks the build **UNSTABLE** (the scan worked and found issues). Any other non-zero exit marks it **FAILURE**.
- Archives `akamai/results/scan.log`.
- The deploy stages (ship to .98, compose build/up) are unchanged.

---

## 7. Test / verification

1. Push to `main`. The GitHub webhook starts `vnotes` (`Started by GitHub push by mcropsey`).
2. Console shows, in order:
   - `Obtained Active Testing API token.`
   - `Refreshed AT auth: alice|bob|charlie|admin` → `Refreshed 4 authentication(s)`
   - `Using active-cli:3.73.6`, `Login Succeeded`
   - scan progress and the findings list
3. In AT: **VNotes → Scans → Environment dropdown = "Jenkins CI"** (the only environment).
4. Manual re-run with no redeploy: **Build with Parameters → SKIP_DEPLOY ✓**.

### Verification log (2026-11-18)

| Build | Trigger | Result | Notes |
|---|---|---|---|
| #48 | GitHub push | FAILURE | Token, auth refresh, registry login and image pull were all OK. Then `SEVERITY_THRESHOLD: unbound variable`, because Jenkins doesn't define `params` on the first run of a newly parameterized job. Fixed by defaulting it (`SEVERITY_GATE = params.SEVERITY_THRESHOLD ?: 'high'`). |
| #49 | GitHub push | **UNSTABLE** (as designed) | Full authenticated scan, about 3 min. Threshold `high` exceeded (2 HIGH). BOLA test **passed** only because `app/` was the *fixed* variant at the time (see below). |
| #50 | GitHub push | **UNSTABLE** (as designed) | First run with the 10 req/s rate limit: zero `rate limit exceeded`, about 9.7 min, and it also caught *Weak Password Policy* (MEDIUM), which #49 missed. Still the fixed app, so no BOLA. |
| #51 | GitHub push | **UNSTABLE** (as designed) | First run with the **vulnerable** app restored. **BOLA detected:** *Resource is Accessible Across Users (Fuzz-Based)* HIGH on `GET` and `PUT /api/notes/{note_id}`. 4 HIGH total, 0 rate-limit hits, about 8.8 min. |

Build #49 findings: 2 HIGH ("Sensitive API accessed over HTTP" on `/api/auth/login` and `/api/auth/register`) and 2 MEDIUM (Weak JWT Algorithm, Excessive JWT Token Expiration). LOW: Server Information Disclosure, Username Enumeration, Publicly Exposed OpenAPI Specification. INFO: security headers, OpenAPI field-type mismatches.

### Gotcha 1: the app variant decides whether BOLA exists
VNotes is meant to be BOLA-vulnerable (README). On 2026-10-06, commits `bfffe7e` and `8d9cee7` left the **fixed**
variant in `app/`, which checks `owner_id` and returns 403. The scan correctly reported no BOLA. Commit `9fed8fc`
restored `variants/vulnerable/app` (the UI is unchanged). To switch:

```bash
./switch-variant.sh vulnerable   # or: fixed  (then: git checkout static/index.html to keep the current UI)
git commit -am "switch variant" && git push   # pipeline redeploys and rescans
```

To check BOLA by hand (direct to the app, bypassing Kong):
```bash
TB=$(curl -s -X POST http://192.168.1.98:8000/api/auth/login -d 'username=bob&password=bob12345' | sed -n 's/.*"access_token":"\([^"]*\)".*/\1/p')
curl -s -H "Authorization: Bearer $TB" http://192.168.1.98:8000/api/notes/3   # alice's note: 200 = vulnerable, 403 = fixed
```

AT only needs **two or more auths with the same role** (alice, bob and charlie are all `user`) for the cross-user
BOLA test. It creates and discovers object IDs itself, so you don't supply note IDs.

### Gotcha 2: Kong rate limit in front of the app
`notes.cropseyit.com` goes through Cloudflare, then Kong (`x-kong-request-id`), which allows **1000 req/min**. An unthrottled scan
starts getting `API rate limit exceeded`, and 429s can turn real failures into false "PASSED" results. The AT env is
therefore capped at **10 req/s** (`rate_limit_requests=10, rate_limit_seconds=1`). Set this in the UI under Environment Settings → Rate limit, or:
```bash
curl -X POST -H "Authorization: Bearer $T" -H 'Content-Type: application/json' \
  https://mikec-lab.nonamesec.com/api/active/v2/frontend/environment/<ENV_ID> \
  -d '{"rate_limit_requests":10,"rate_limit_seconds":1}'
```
The CI/CD scan configuration also has `network_performance.rateLimit = 10`.

---

## 8. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Could not obtain accessToken` | SA secret wrong or regenerated. Update `mikec-at-sa-client-secret`. |
| `is not a CI/CD environment` | The env was created as `deployment` type. Recreate it as `cicd` (step 4b). |
| `WARN: no AT authentication named 'x'` | An AT auth was renamed. Names must match the users in `vnotes-at-users`. |
| `login failed for 'x'` | The VNotes DB was reset or the password changed. `POST /api/seed` reseeds the demo users. |
| `docker pull` denied | The registry key was rotated. Re-fetch `/api/active/v2/cicd/parameters` and update `mikec-at-registry-key`. |
| Scan shows only unauthenticated results | Auth tokens are stale. Check that the refresh step ran, or run `akamai/refresh-auths.sh` by hand. |
| Build UNSTABLE every time | Expected: VNotes is deliberately vulnerable (BOLA = HIGH). Use `SEVERITY_THRESHOLD=none` to make it informational. |
| Webhook doesn't fire | GitHub repo → Settings → Webhooks → `https://jenkins.cropseyit.com/github-webhook/`. Check *Recent Deliveries*. |

---

## 9. Claude Code prompt to repeat this

Paste this into Claude Code with the `mikec-active-testing` MCP server
connected. First create the service account in step 3 and fill in the values.

```text
Set up an Akamai API Security Active Testing CI/CD integration for my app with Jenkins, end to end, and test it.

Inputs:
- App: https://notes.cropseyit.com  (Swagger UI /docs, OpenAPI spec /openapi.json)
- Test users (OAuth2 password FORM login at POST /api/auth/login, JWT valid 24h):
  alice/alice123 (user), bob/bob12345 (user), charlie/charlie1 (user), admin/admin123 (admin)
- Git repo: https://github.com/mcropsey/vnotes (branch main, Jenkinsfile at root; push with my
  personal gh account `mcropsey`, then switch back to `mcropsey-akamai`)
- Jenkins: http://192.168.1.100:8080 (= https://jenkins.cropseyit.com), user admin / <JENKINS_PASSWORD>,
  job `vnotes` (Pipeline from SCM, */main, GitHub push trigger). Jenkins runs in Docker with a
  docker:dind agent; controller has bash/curl/docker but no python/jq.
- Deploy: existing stages ship the repo to 192.168.1.98 and run docker compose; keep them.
- Active Testing tenant: https://mikec-lab.nonamesec.com
- Service account (Settings > User Management > Service Accounts, Admin role):
  client_id=<CLIENT_ID> client_secret=<CLIENT_SECRET>

Do the following:
1. Mint a platform token: POST /auth/token {"grant_type":"client_credentials",...} -> accessToken.
   Get the scanner registry creds from GET /api/active/v2/cicd/parameters with that token.
2. Create AT app "VNotes" (base URL + OpenAPI URL) via the MCP server, or reuse it if it already exists.
3. Create a CI/CD-TYPE environment "Jenkins CI" via
   POST /api/active/v2/frontend/application/{appId}/environment with "type":"cicd", the OpenAPI
   source, and four header auths named alice/bob/charlie/admin ("Authorization: Bearer <jwt>").
   (MCP create_environment only makes deployment-type envs.)
4. PUT /api/active/v2/frontend/cicd/environments/{envId}/scan-configuration with platform=jenkins,
   Default API Security Tests profile, severityThreshold=high, verbose.
5. Add Jenkins Secret-text credentials: mikec-at-registry-key, mikec-at-sa-client-id,
   mikec-at-sa-client-secret, vnotes-at-users ("user:pass ..."). Never commit secrets to git.
6. In the repo, add akamai/refresh-auths.sh (re-login each user, then PUT fresh Bearer JWTs into
   /api/active/v2/applications/{app}/environments/{env}/authentications/{id}, matched by name), and
   rewrite the Jenkinsfile Active Scan stage: mint SA token -> ACTIVE_API_TOKEN, run the refresh
   script, docker login to the registry HOST, pull active-cli:<GET /active/backend/version>, run
   `scan --api-url --env-id --test-group-id --app-version --severity-threshold --verbose`.
   A threshold breach makes the build UNSTABLE; other errors make it FAIL. Add a /health wait stage
   and SEVERITY_THRESHOLD / SKIP_DEPLOY parameters.
7. Commit and push to main. Confirm the webhook triggers the build, watch it finish, and confirm the
   scan shows up in AT under VNotes > Jenkins CI with authenticated findings (e.g. BOLA).
8. Update ~/Downloads/notes-jenkins-noname-at-config.md with the final IDs and results.
```

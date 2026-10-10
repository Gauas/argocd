# Production identity cutover

- `api.gauas.com/v1/auth`, `/v1/users`, `/.well-known/jwks.json` and
  `/v1/identity/health` route to identity-service.
- Vault path: `secret/identity-service/production`.
- HunterJob image is pinned to the local-JWT build from commit `ddec2b1`.
  It verifies EdDSA JWTs using cached public JWKS from the identity-service
  internal DNS address; no private key is copied to HunterJob.
- Account-service, authorization-service, their HPAs and ForwardAuth middleware
  are removed. The HunterJob ingress remains enabled and uses only CORS.
- Normal API requests no longer depend on central token validation or revoked
  SID Redis lookups. Logout revokes refresh sessions; access JWTs last until exp.
- Frontend uses server-only `API_BASE_URL=https://api.gauas.com` and forwards the
  HttpOnly access cookie as an Authorization Bearer header. No X-Gauas trust.
- Health endpoints are public. Protected endpoints must reject missing/bad
  tokens and forged identity headers with 401.
- Vault Agent log level is `warn`, preserving warnings/errors without routine
  token-renewal chatter. No database migration or Vault value is changed here.

## Rollout constraints

HunterJob is temporarily scheduled on `peguin`: `whale` currently fails host
DNS resolution for `registry-1.docker.io`. Remove the temporary node selector
after node DNS is repaired. The existing crawler is not restarted for this
logging-only change; its Vault agent already runs init-only.

Shared email-service and socket-hub manifests live on ArgoCD's `master` branch
under `clusters/shared-service`, not on `production`. Email-service uses the
same temporary `peguin` placement. Their manifests were applied selectively
because the existing full shared-service sync is blocked on the unrelated
`minio-public-bucket` hook. This change does not alter that hook or MinIO data.

Keep `email.send` for verification commands on the asynchronous outbox/queue
path. Message names follow `domain.action`; `email.sent` describes completion,
not a request to send. Email-service does not participate in authentication.

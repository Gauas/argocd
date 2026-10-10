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

HunterJob and email-service have no node selector. The temporary `peguin`
placement has been removed; repair node DNS rather than constraining application
scheduling. On `whale`, the Tailscale DNS resolver returns SERVFAIL for Docker
Hub while querying 8.8.8.8 succeeds. Its host uses ifupdown/DHCP on `ens18`.
A persistent per-node configuration is `supersede domain-name-servers 8.8.8.8,
1.1.1.1;` in `/etc/dhcp/dhclient.conf`, paired with
`tailscale set --accept-dns=false`. Apply the network restart from a VM console
or a maintenance window: restarting networking may interrupt SSH/Tailscale.
This opts the node out of Tailscale-managed DNS/MagicDNS, not the VPN itself.
These DNS changes are instructions, not changes applied by this cutover.

Shared email-service and socket-hub manifests live on ArgoCD's `master` branch
under `clusters/shared-service`, not on `production`. Their manifests were applied selectively
because the existing full shared-service sync is blocked on the unrelated
`minio-public-bucket` hook. This change does not alter that hook or MinIO data.

Keep `email.send` for verification commands on the asynchronous outbox/queue
path. Message names follow `domain.action`; `email.sent` describes completion,
not a request to send. Email-service does not participate in authentication.

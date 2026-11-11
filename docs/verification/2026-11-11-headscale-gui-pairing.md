# HA Headscale GUI pairing verification (2026-11-11)

## Candidate

- Woow Tailscale 0.1.5: masked one-use `auth_key` in HA Configuration, temporary file for `tailscale up`, option removal after Running.
- Woow Headscale VPN 0.1.1: Headscale v0.29.4, Headplane 0.7.1, HA Ingress retained.

## Completed checks

- `tailscale/tests/*.sh` passed with `TMPDIR=/tmp`; Bash syntax, YAML parsing, and `git diff --check` passed. The default Omnigent temp directory forces setgid (mode 2700), which makes the pre-existing migration test's exact mode assertion fail; `/tmp` provides normal mode 700.
- `test-headscale-pairing.sh` runs the actual s6 `post-tailscaled/run` against stubbed bashio and Tailscale. It covers fresh enrollment, healthy existing node, rejected key, stalled login, no key, missing option returned as `null`, clear failure, key file mode/cleanup, and identity preservation.
- The pinned Tailscale 1.102.3 source accepts `--auth-key=file:<path>`; the CLI trims the file's contents.
- A local isolated Headscale v0.29.4 plus Tailscale v1.102.3 userspace client rejected an invalid key, accepted a fresh one-use key on the same client state, reached `Running`, appeared in Headscale under the chosen owner with `100.64.0.1`, and kept that address after daemon restart. Repeated with the HA add-on's default `advertise_exit_node=true` and `advertise_connector=true` flags. Temporary keys, identity files, and logs were removed afterward.
- The official `ghcr.io/tale/headplane:0.7.1` manifest has amd64 and arm64 variants. Its amd64 `/app/build/server/index.js` contains the exact `"mode": "lazy"` route-discovery pattern once, so the add-on's guarded ingress patch can apply.
- Headplane v0.7.1 source confirms the GUI path **Settings → Manage Auth Keys → Create pre-auth key**. The dialog selects **User**, defaults **Reusable** to off, and displays the full key once after creation. The add-on guides were corrected to match that UI.
- The field is masked in HA, but a user-selected Bashio `trace` log level may print raw options before the key is cleared. Documentation instructs administrators to use `info`/`debug` during pairing. Normal script-level tests do not validate Bashio's trace redaction.

## Release gates still needed

- GitHub Actions builds on amd64/aarch64 and Store Upgrade comparison/data-preservation checks.
- A real HAOS Supervisor upgrade and browser clicks through Headplane Users, pre-auth key creation, Woow Tailscale Configuration, Web UI status, Headplane Machines, and peer service access. No reachable HAOS Supervisor or HA browser session is connected to this workspace yet; local CLI checks do not replace this acceptance.


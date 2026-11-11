# HA Headscale GUI Pairing Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Let a Home Assistant administrator pair the Woow Tailscale add-on to Headscale using the existing Headplane GUI and HA configuration form, then see a truthful connected state, without copying a Headscale administrator API key between add-ons.

**Architecture:** Headplane selects the node owner and creates a single-use pre-auth key. The Woow Tailscale add-on accepts the Headscale URL and a masked, optional key in HA's native Configuration UI. Its existing `tailscale up` path reads the key from a temporary 0600 file, preserves its node state, and removes the HA option after a confirmed Running state. A key failure leaves the option editable and the existing identity intact. The existing Tailscale Web UI remains the status display. The Headscale add-on is updated to the PaaS-tested Headscale/Headplane versions only after its ingress and data-upgrade checks pass.

**Tech Stack:** Home Assistant Supervisor add-on schema, bashio, s6, Tailscale CLI, Headplane, Bash tests, Docker and HAOS upgrade checks.

---

### Task 1: Add a masked one-use pairing key to Woow Tailscale

**Files:**
- Modify: `tailscale/config.yaml`
- Modify: `tailscale/translations/en.yaml`
- Modify: `tailscale/rootfs/etc/s6-overlay/s6-rc.d/post-tailscaled/run`
- Create: `tailscale/tests/test-headscale-pairing.sh`

1. Add optional `auth_key: ""` to `options` and `auth_key: password?` to `schema`. Keep `login_server` as the existing URL field. The field is an enrollment key, never a saved Headscale administrator key.
2. After tailscaled reaches `Running`, `NeedsLogin`, or `Stopped`, read `auth_key` once. If it is nonempty and the node is not already Running, create a 0600 file under `/run` and append `--auth-key=file:<path>` to the *existing complete* `tailscale up` flag array. Do not pass the key in argv, logs, or a persistent file. Clean the file on every exit path.
3. If the node is already Running, keep its identity and ignore the supplied key; still clear the option after the normal `tailscale up` and Running checks. Do not force reauthentication on a healthy node.
4. When `tailscale up` fails, retain the HA option for correction, remove the temporary file, and report a concise error without key material. When it reaches Running, call `bashio::addon.option 'auth_key' ''` and verify the operation succeeded. Keep the original no-key web-auth path unchanged.
5. The test must run the actual s6 `post-tailscaled/run` script against stubs for bashio, tailscale, and status. Cover fresh success, already Running, bad key, no key, and cleanup. It must assert no secret appears in captured argv/logs and no current identity file is removed. Run the test before implementation to confirm failure and after implementation to confirm success.

### Task 2: Make pairing steps and status understandable in HA

**Files:**
- Modify: `tailscale/DOCS.md`
- Modify: `tailscale/README.md`
- Modify: `tailscale/CHANGELOG.md`
- Modify: `tailscale/translations/en.yaml`
- Modify: `tailscale/config.yaml`

1. Document the GUI flow: Headscale add-on Web UI → Headplane Users → select owner → create one-use pre-auth key → Woow Tailscale Configuration → paste the public `login_server` URL and key → Save/Restart → open Woow Tailscale Web UI and confirm Running/IP; verify the node in Headplane. Include the web-auth alternative using Add Device for users who do not want a pre-auth key.
2. Explain same-server recovery, invalid/expired key replacement, control-server switching, and the fact that a one-use key disappears from HA options only after success. The Headscale public control URL and Headplane ingress URL are distinct.
3. Bump Woow Tailscale from `0.1.4` to `0.1.5`; add the change to its changelog. Run `tailscale/tests/*.sh` and the repository's CI checks.

### Task 3: Align Woow Headscale runtime with PaaS and document owner selection

**Files in `source/ha-vpn-headscale`:**
- Modify: `headscale/config.yaml`
- Modify: `headscale/Dockerfile`
- Modify: `headscale/addon_info.yaml`
- Modify: `headscale/DOCS.md`
- Modify: `headscale/README.md`
- Modify: `headscale/CHANGELOG.md`
- Create: focused validation tests under `headscale/tests/` if current checks do not cover the changed behavior.

1. Update the underlying Headscale from v0.29.3 to v0.29.4 and Headplane from 0.7.0 to 0.7.1, matching the PaaS-tested pair. Bump the add-on to `0.1.1`. Keep existing data paths, ingress-only management, and the public control-plane port.
2. Verify the version-sensitive Headplane route-discovery patch still applies; test Machines, Users, DNS, ACL, Settings, Add Device, owner selection, login/logout, and all static assets through HA Ingress. Preserve the old image and version if this exact version pair fails.
3. Document the one-use Headplane key flow for the Woow Tailscale add-on, with no Headscale admin API key sharing. Verify the existing data-preservation `upgrade-check/seed` and `verify` hooks against 0.1.0 → 0.1.1.

### Task 4: Release and acceptance

**Files:** each repository's PR and Store synchronization records; keep receipts under `docs/verification/`.

1. Run focused tests, full repository CI, image builds for amd64/aarch64, and an isolated HAOS upgrade test for both add-ons. On HAOS, create a Headplane owner/key, pair Woow Tailscale through HA Configuration, confirm the same node/IP in Tailscale and Headplane, prove peer HTTP traffic, test bad-key retry and same-server reauthentication, and verify backups and rollback.
2. Push feature branches to the two canonical GitHub repositories, open PRs against `main`, and wait for all required checks including Store upgrade comparison. Merge only validated versions, then verify Store source synchronization and installed add-on versions. Do not describe a branch push, candidate image, or local HAOS run as a Store release.


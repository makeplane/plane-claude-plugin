# Licensing (Commercial)

Without a key, the instance runs on the **Free plan**. Installing without a key is fine, and the
user can activate later. The user enters the key in the UI. Never ask for it in the chat.

## Where to activate

| Plan | Scope | Where |
| --- | --- | --- |
| Pro, Business | One workspace (other workspaces stay Free) | **Workspace settings → Billing and plans** |
| Enterprise Grid | The whole instance, including workspaces created later | **God mode → Billing** |
| Enterprise Grid, air-gapped | The whole instance, from a license **file** | **God mode → Billing**, upload the file |

Details: <https://developers.plane.so/self-hosting/manage/manage-licenses/overview>.

## How the binding works (set expectations)

- A key binds to the workspace (Pro / Business) and to the **instance**. The instance is
  identified by the machine signature generated at setup. A key works in one place at a time.
  Activating it elsewhere fails with `License already active for another workspace` or
  `… another instance`. **Delink** it first, from Billing → … → Delink license key, or from the
  Prime portal.
- **Never change the machine signature** of a licensed install:
  - Compose, Podman and air-gapped: `MACHINE_SIGNATURE` in the env file.
  - AIO: `/app/data`.
  - prime-cli: the host machine-id.
  - Kubernetes: derived from the license domain, namespace and release name, so never rename
    them.
- Seats come from roles. Owners, Admins and Members each take a paid seat. Guests are free, up to
  five per paid seat. Enterprise Grid counts every user account on the instance.
- After a renewal, a plan change or a seat purchase, use **Sync plan** in the same Billing
  screen if the change hasn't shown up.
- Air-gapped: there are no online check-ins. The file carries the entitlements and is pinned to
  the Plane version, so after an upgrade, delink and upload the new file.

## Network

Online activation and sync need outbound HTTPS to `https://prime.plane.so` **from the Plane
host or cluster**. Test from there, not from the user's laptop. With a corporate proxy or
firewall, allowlist that host. On Kubernetes, leave `license.licenseServer` at its default.

## When activation fails

Start with <https://developers.plane.so/self-hosting/troubleshoot/license-errors>. The common
causes: the key is active elsewhere (delink it), more paid roles than purchased seats, an
Enterprise Grid key entered in workspace settings or a Pro / Business key entered in God mode,
or no route to `prime.plane.so`.

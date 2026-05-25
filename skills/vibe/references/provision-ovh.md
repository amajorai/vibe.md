# OVH Provisioning

OVH does not support fully scripted VPS provisioning via a simple CLI flow. The CLI is useful for listing and inspecting, but **you must order/create the VPS in the OVH Control Panel** (or via the OVH API).

```bash
curl -fsSL https://raw.githubusercontent.com/ovh/ovhcloud-cli/main/install.sh | sh
ovhcloud login

# Inspect existing VPS instances:
ovhcloud vps list
```

To provision: go to the [OVH Control Panel](https://www.ovh.com/manager/) → order a VPS → choose Ubuntu, add your SSH key during checkout, and note the IP address it assigns. Because there is no provisioning command, this is a **manual step** — wait until the Control Panel shows the VPS as delivered/running and gives you an IP before continuing.

Then connect (note the SSH user OVH assigns differs by image — it is often `ubuntu` or `debian`, shown in the Control Panel; the root account is usually disabled for direct login):

```bash
SERVER_IP=<your-ovh-vps-ip>
ssh ubuntu@"$SERVER_IP"   # adjust the user to match what the Control Panel shows
```

**If SSH fails**, do not continue to Phase 3B yet. Common causes and fixes:

- *Connection refused / timed out:* the server may still be booting. Wait 30–60s and retry.
- *Permission denied (publickey):* you are using the wrong key or the wrong user. Try `ssh -i ~/.ssh/<your-key> <user>@"$SERVER_IP"`, and verify the user matches the provider (OVH usually `ubuntu`/`debian`).
- *No public key registered:* re-check that the key you uploaded during checkout matches your local `~/.ssh/*.pub`.
- *Host key changed warning:* if you reused an IP, run `ssh-keygen -R "$SERVER_IP"` then retry.

Resolve SSH before proceeding — all remaining setup happens inside that session.

**Transition to the server:** Once that SSH connection succeeds you are on the server — proceed to Phase 3B. The `$SERVER_IP` variable lives only in your local shell and is not carried over — that's fine, Phase 3B does not need it.

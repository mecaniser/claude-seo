# Persistent local CI

All existing jobs use disposable Ubuntu ARM64 VMs on owned hardware, labelled
`owned-linux`. Each repository has a separate runner registration and supervisor.
Tests, workflow triggers, permissions and failure semantics are unchanged.

A pristine stopped `ci-tools-base` is provisioned with `base.yaml`, `provision.sh`
and `network.nft`. No repository jobs ever run in the base. Workers have no host
mounts, forwarded agents, host Docker socket or sudo. Rootless Docker and the
root-owned firewall isolate job execution from the host and private network.
Host authentication obtains only short-lived registration tokens for the guest.
Each ephemeral job is followed by VM deletion and a fresh clone.

Install the reviewed supervisor outside any checkout as a macOS per-login
LaunchAgent with KeepAlive and RunAtLoad. Start it with `--slots 1 --state-dir`
pointing to its private state directory. The host needs `gh` and Lima. External
contributor approval is required at the repository setting; labels alone are
not a security boundary. Never expose host credentials to a guest.

The Mac must be powered, connected, and logged in after FileVault unlock.
An offline host queues jobs; it does not bypass tests or increase paid limits.
Rollback is a focused PR restoring hosted routing; retain every check.

## Base firewall provisioning

Before stopping the pristine base, install and enable its root-owned firewall:

```sh
limactl shell BASE sudo bash < provision.sh
limactl shell BASE sudo tee /etc/nftables.conf < network.nft >/dev/null
limactl shell BASE sudo systemctl enable --now nftables
limactl shell BASE sudo nft list table inet ci_isolation
limactl stop BASE
```

Replace `BASE` with the base named in `supervisor.py`. Registration fails closed
without the firewall. Never run the provisioning script in a worker or host.

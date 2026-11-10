# Restoring etcd from a snapshot

> **WARNING:** practise this on a throwaway VM or spare cluster first. Do **not**
> test it on the live cluster. A restore rolls the cluster back to the moment
> of the snapshot. Everything created after it is lost.

## What you need

| Item | Where it is |
|---|---|
| A snapshot file | omarchy: `~/Backups/pi-cluster/<node>/snapshots/` |
| The server token (must match the snapshot's cluster) | omarchy: `~/Backups/pi-cluster/<node>/token` |

Snapshots from any node are interchangeable (it is one etcd cluster). Pick the
newest one from before the problem started.

## How backups flow

1. k3s takes an etcd snapshot on every server every 12 hours and keeps 5
   (k3s defaults) in `/var/lib/rancher/k3s/server/db/snapshots/`.
2. Every hour, `k3s-backup-stage.timer` on each Pi copies the snapshots and the
   token into `/home/k3sbackup/outbox/`.
3. Every 6 hours, `pi-cluster-backup.timer` on omarchy (a user timer: it runs
   while `pts` is logged in there, and catches up on login) pulls each node's
   outbox into `~/Backups/pi-cluster/<node>/` and deletes snapshots older than 30 days.
   It uses a dedicated key that can only read the outbox (`rrsync -ro`).

Take a manual snapshot before risky changes:
```sh
ssh pts@10.0.0.104 sudo k3s etcd-snapshot save --name before-<change>
```

## Case A: cluster still installed, data is bad

Run on the nodes as root. `SNAP` is the snapshot file name.

1. Copy the snapshot to pi1 (from omarchy):
   ```sh
   scp ~/Backups/pi-cluster/pi1/snapshots/$SNAP pts@10.0.0.104:/tmp/
   ssh pts@10.0.0.104 sudo mv /tmp/$SNAP /var/lib/rancher/k3s/server/db/snapshots/
   ```
2. Stop k3s on **all three** servers:
   ```sh
   sudo systemctl stop k3s        # pi1, pi2, pi3
   ```
3. On **pi1**, reset etcd from the snapshot:
   ```sh
   sudo k3s server --cluster-reset \
     --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/$SNAP
   ```
   Wait for: `Managed etcd cluster membership has been reset, restart without --cluster-reset flag now`.
4. Start k3s on **pi1**: `sudo systemctl start k3s`, and wait for `kubectl get nodes` to work.
5. On **pi2**, then **pi3** (one at a time), move the old etcd data aside and start k3s so it rejoins:
   ```sh
   sudo mv /var/lib/rancher/k3s/server/db /var/lib/rancher/k3s/server/db.old-$(date +%F)
   sudo systemctl start k3s
   ```
6. Verify: 3 nodes `Ready`, 3 etcd members, `kubectl get pods -A` healthy.

## Case B: rebuilding from bare nodes

1. Image the cards, run `10-base.yml` (see README).
2. Put the **old** token in pi1's k3s config before installing, so it can decrypt
   the snapshot: add `token: "<contents of the backed-up token file>"` to
   `/etc/rancher/k3s/config.yaml` on pi1 (or install pi1 normally, then
   stop k3s and add it).
3. Run `20-k3s.yml --limit pi1`, then follow Case A steps 1-4 on pi1.
4. Copy that token to `~/.config/pi-cluster/k3s-token` on the workstation,
   then join pi2 and pi3 with `20-k3s.yml --limit pi2` / `pi3`.

## Reference

- k3s docs: Backup and Restore with Embedded etcd: https://docs.k3s.io/datastore/backup-restore

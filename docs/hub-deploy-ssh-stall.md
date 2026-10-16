# Hub deploy stall after a successful install

Job: [rehearse-86960-periodic-ci-openshift-kni-eco-ci-cd-main-cnf-ran-ptp-sno-5.0-cnf-ran-ptp-tests/2110746860237885440](https://prow.ci.openshift.org/view/gs/test-platform-results-public/pr-logs/pull/openshift_release/86960/rehearse-86960-periodic-ci-openshift-kni-eco-ci-cd-main-cnf-ran-ptp-sno-5.0-cnf-ran-ptp-tests/2110746860237885440)

Step `telcov10n-functional-cnf-ran-hub-deploy` ran `playbooks/deploy-ocp-sno.yml` against hub `kni-qe-110` (OpenShift `5.0.0-rc.6`). The step ran from 15:17Z to 18:08Z (2h 51m). The cluster was installed by 16:29Z. About 1h 39m of the step was one SSH session that stayed open after `openshift-install` had already printed `Install complete!`.

## Timeline (EDT, UTC-4)

| Time | What happened |
| --- | --- |
| 11:17 | Step starts. Registry and HTTP store come up; the release mirror is about 23G and finishes well before the VM boot. |
| 11:37–11:52 | Agent ISO is built from a cached RHCOS image. The hub disk image writes in about 12 seconds. |
| 11:54:05 | Bootstrap is complete. |
| 11:54:13 | `openshift-install --log-level=debug agent wait-for install-complete` starts over SSH from the CI pod (`10.29.6.124`). Bastion `sshd` pid `395052`. |
| 12:07–12:19 | API flaps (`server is down or not responding`, eight times). Install progress resets. This is real cluster bring-up, not the long stall. |
| 12:29:26–12:29:30 | ClusterVersion `Completed` (`5.0.0-rc.6`). Operators are stable, the console route is admitted, the installer logs `Install complete!`. All cluster operators are Available. |
| 12:45:39 | Bastion closes the SSH session. Secure log has no `disconnected by user` / `Received disconnect` for pid `395052`. |
| 12:45–14:08 | No new CI SSH to the bastion. |
| 14:08:34 | Ansible opens a new session and runs `wait-for install-complete` again. It returns in about 2 seconds and appends a second `Install complete!` to the same `.openshift_install.log`. |
| 14:08:39 | Post-playbook `oc get clusterversion` succeeds. The step is marked successful. |

Marketplace `ImagePullBackOff` for `registry.redhat.io` is a disconnected-cluster side effect. It did not gate this step.

## Why the SSH session stuck

The last play in `playbooks/deploy-ocp-sno.yml` imported `redhatci.ocp.monitor_agent_based_installer` (`requirements.yml` pins `redhatci.ocp` `3.5.1781925805`). That role ran:

```yaml
cmd: "{{ agent_based_installer_path }} --log-level=debug agent wait-for install-complete"
```

`deploy-ocp-sno.yml` set `mabi_retry_install_complete_check: true`, so the role retried that command (10 times, 60s apart) until stderr matched a known installer failure string. That string is not in `.openshift_install.log`. The second run happened because the SSH session died uncleanly, so Ansible treated the task as failed.

What the process did after success:

1. `openshift-install` wrote `Install complete!` to both `.openshift_install.log` and stderr. With `--log-level=debug`, stderr was a long stream carried over the SSH channel.
2. The CI-side `ssh` stopped reading that stream and left its packet loop. `ansible.cfg` already sets `ServerAliveInterval=30` and `ServerAliveCountMax=10` (a 5 minute dead-peer timeout). Those probes only run while `ssh` is in that loop, so they never fired.
3. The installer blocked on the stderr write. TCP gave up on the full window about 16 minutes later (session close at 12:45, with no client disconnect).
4. The CI socket stayed until the kernel TCP keepalive expired (default `tcp_keepalive_time` 7200s plus probes, about 2h 11m). The session opened at 11:54 and the next SSH was at 14:08. Ansible could not start the retry until that socket died.
5. The retry itself was cheap. The hub was already up.

`ControlMaster` is not involved. `ansible.cfg` `ssh_args` replaces Ansible's default and omits `ControlMaster` / `ControlPersist`. Ansible only adds `ControlPath` when `ControlPersist` is present, so the `control_path` settings in that file are unused.

The role was removed from `redhatci.ocp` in collection v5 (2026-08). Bumping the collection is not a drop-in replacement for this play.

## Fix options

### Option 1 — own the wait in eco-ci-cd

Implemented as `playbooks/roles/monitor_agent_based_installer`. `deploy-ocp-sno.yml` and `deploy-ocp-hybrid-multinode.yml` import that role. The `redhatci.ocp` pin is unchanged.

The role keeps the API `/readyz` wait and the install-complete retry. `openshift-install` stdout and stderr go to `.openshift_install.log` on the bastion, appended after the process exits so the write does not interleave with the installer's own log. Ansible sees the exit code, the few stderr lines the retry condition uses, and the last 40 lines of that log. Each wait is wrapped in GNU `timeout 90m` (`mabi_wait_timeout`) and an Ansible task timeout of 5700 seconds (`mabi_task_timeout`). That Ansible timeout fails the attempt before `until` runs, so the rescue runs one 3 minute `wait-for install-complete`. A cluster that is already up passes that re-check and the play continues. Otherwise the rescue gathers logs and fails. `ControlMaster` stays off.

### Option 2 — `IPQoS=none` in eco-ci-cd `ansible.cfg`

Also entirely in this repo, and smaller. Append `-o IPQoS=none` to `ssh_args` in `ansible.cfg`. Keep the current `ServerAlive*` values. Do not add `ControlMaster`.

This targets a known failure mode on some CI-to-lab paths: SSH DSCP marking is dropped and the session blackholes. It does not fix the case where `ssh` has already left its packet loop, which is what the bastion log shows here (no client disconnect, then a 16 minute TCP retry, then a silent socket until 14:08). Use it together with option 1, not instead of it.

### Option 3 — change `redhatci.ocp`

Patch `roles/monitor_agent_based_installer` in the collection this repo pins (`3.5.1781925805`): same redirect and task timeout as option 1. That change is not in eco-ci-cd. The role no longer exists on current `redhatci.ocp` main, so a collection bump does not deliver it. Anyone shipping the fix has to fork or patch the pinned 3.5 collection and then move the pin in `requirements.yml`.

## What not to change

Do not put `ControlMaster=auto` and `ControlPersist` back into `ssh_args`. With a master connection, EOF on stderr can wait until the persisted connection closes, which is the same class of hang.

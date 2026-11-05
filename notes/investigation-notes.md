# Investigation Notes – Linux Systemd Service Persistence

## 1. Investigation Summary

This investigation evaluated whether Wazuh could detect systemd service persistence-related changes on a Rocky Linux 9.8 virtual machine.

Wazuh FIM was configured to monitor `/etc/systemd/system` in real time. Three custom detection rules were used to identify creation, modification, and deletion events involving a controlled systemd service and its enablement link.

A harmless one-shot service using `/usr/bin/logger` was created and tested.

The investigation successfully validated all three detection categories.

## 2. Environment and Baseline

| Attribute | Observed Value |
|---|---|
| Operating System | Rocky Linux 9.8 |
| Kernel | 5.14.0-687.31.1.el9_8.x86_64 |
| Virtualization | VMware |
| Architecture | x86-64 |
| systemd Version | 252 |
| Wazuh Version | 4.13.1 |
| Wazuh Revision | rc1 |
| Manager IP | 192.168.203.129 |
| Manager Status | Active |
| Syscheck | Running |
| Analysisd | Running |

The initial systemd inventory showed 50 enabled service unit files.

The controlled service did not exist during the initial baseline.

The vendor unit directory was:

`/usr/lib/systemd/system`

The Wazuh Manager, Indexer, and Dashboard start commands were issued. The Manager was subsequently verified as active. Independent active-state verification for the Indexer and Dashboard was not captured in the supplied evidence.

## 3. Wazuh FIM Configuration

Real-time monitoring was configured for:

`/etc/systemd/system`

The configuration was validated using:

- `xmllint --noout /var/ossec/etc/ossec.conf`
- `wazuh-syscheckd -t`
- `wazuh-analysisd -t`

Wazuh Manager was restarted successfully.

The Wazuh log confirmed:

`2026/11/05 07:49:14`

Monitoring path:

`/etc/systemd/system`

The log also confirmed that the directory was configured for real-time monitoring.

## 4. Custom Detection Rules

Three custom rules were configured in:

`/var/ossec/etc/rules/local_rules.xml`

| Rule ID | Level | Parent Rule | Purpose |
|---|---:|---:|---|
| 100640 | 12 | 554 | Service or enablement-link creation |
| 100641 | 12 | 550 | Service or enablement-link modification |
| 100642 | 10 | 553 | Service or enablement-link deletion |

The path filter covered:

- `/etc/systemd/system/lab128-persistence-demo.service`
- `/etc/systemd/system/multi-user.target.wants/lab128-persistence-demo.service`

All rules were mapped to MITRE ATT&CK T1543.002.

The rules file passed XML validation and Wazuh Analysisd configuration testing.

## 5. Controlled Service Creation

The following service was created:

`/etc/systemd/system/lab128-persistence-demo.service`

The service configuration used:

- `Type=oneshot`
- `/usr/bin/logger` as the executable.
- `RemainAfterExit=yes`
- `WantedBy=multi-user.target`

The file was configured with:

- Owner: root
- Group: root
- Permissions: 0644

`systemd-analyze verify` completed without reporting an error.

### Creation Detection

Wazuh generated rule `100640` at:

`2026-11-05 08:08:49 IST`

Detected path:

`/etc/systemd/system/lab128-persistence-demo.service`

The alert recorded:

- FIM event: added
- Mode: realtime
- File permissions: rw-r--r--
- Owner: root
- Group: root
- File hashes: MD5, SHA1, and SHA256

This confirmed service unit creation detection.

## 6. Enablement-Link Creation

The service was enabled using:

`systemctl enable lab128-persistence-demo.service`

Systemd created:

`/etc/systemd/system/multi-user.target.wants/lab128-persistence-demo.service`

The link pointed to:

`/etc/systemd/system/lab128-persistence-demo.service`

Wazuh generated rule `100640` at:

`2026-11-05 08:11:16 IST`

The alert identified the enablement link as a newly created file-system object.

This confirmed that the FIM configuration and custom rule also detected the controlled enablement-link creation.

## 7. Initial Service Execution

The service was started successfully.

Systemd reported:

`Active: active (exited)`

Execution result:

`status=0/SUCCESS`

The journal recorded:

`Nov 05 08:11:50 localhost.localdomain lab128-systemd[17818]: Lab 128 initial service execution`

The `active (exited)` state was expected for this successful one-shot service with `RemainAfterExit=yes`.

## 8. Service Modification

The service message was changed from:

`Lab 128 initial service execution`

To:

`Lab 128 modified service execution`

The unit configuration was verified again, and systemd was instructed to reload the unit definition.

The service was restarted.

### Modification Detection

Wazuh rule `100641` detected the service unit modification at:

`2026-11-05 08:13:48 IST`

The alert recorded changes to:

- File size.
- Inode.
- Modification time.
- MD5 hash.
- SHA1 hash.
- SHA256 hash.

Additional modification events were recorded at:

- `2026-11-05 08:13:57 IST`
- `2026-11-05 08:14:08 IST`

The same `sed -i` command had been issued repeatedly. These repeated file rewrites generated additional inode and modification-time changes, even though the intended configuration content was already updated.

These events should not be interpreted as three distinct service configuration changes.

### Modified Service Execution

The journal recorded:

`Nov 05 08:14:37 localhost.localdomain lab128-systemd[17920]: Lab 128 modified service execution`

This confirmed that the modified service configuration was loaded and executed successfully.

## 9. Cleanup and Deletion Detection

The service was disabled using:

`systemctl disable --now lab128-persistence-demo.service`

Systemd removed the enablement link.

The service was then removed from:

`/etc/systemd/system/lab128-persistence-demo.service`

Systemd's manager configuration was reloaded.

### Enablement-Link Deletion

Wazuh rule `100642` detected deletion of the enablement link at:

`2026-11-05 08:16:56 IST`

Detected path:

`/etc/systemd/system/multi-user.target.wants/lab128-persistence-demo.service`

### Service Unit Deletion

Wazuh rule `100642` detected deletion of the service unit file at:

`2026-11-05 08:17:48 IST`

Detected path:

`/etc/systemd/system/lab128-persistence-demo.service`

Both deletion events were recorded in `alerts.json`.

## 10. systemctl reset-failed Observation

After the service file was removed and systemd configuration was reloaded, the following command was attempted:

`systemctl reset-failed lab128-persistence-demo.service`

Systemd returned:

`Unit lab128-persistence-demo.service not loaded.`

This was expected in the observed cleanup state.

The service had completed successfully, was disabled, and its unit file had been removed. There was no loaded failed-unit state to reset.

This message does not indicate a Wazuh failure or a failed service execution.

## 11. Final Wazuh Health

The final validation confirmed:

| Component | Result |
|---|---|
| Wazuh Manager | Active |
| Wazuh Syscheck | Running |
| Wazuh Analysisd | Running |

The controlled service unit and enablement link had been removed.

## 12. Findings

### Confirmed

- Wazuh FIM monitored `/etc/systemd/system` in real time.
- Rule 100640 detected service unit creation.
- Rule 100640 detected enablement-link creation.
- Rule 100641 detected service unit modification.
- Rule 100642 detected enablement-link deletion.
- Rule 100642 detected service unit deletion.
- The controlled service executed successfully.
- Both initial and modified execution messages were present in the system journal.
- Wazuh Manager remained active after the investigation.

### Not Established

- No malicious systemd persistence was demonstrated.
- No unauthorized service execution was demonstrated.
- No privilege escalation was demonstrated.
- No unauthorized user activity was established.
- The investigation did not establish coverage of every possible systemd persistence location or service configuration mechanism.

## 13. Conclusion

The investigation successfully validated Wazuh's ability to detect controlled systemd service unit creation, modification, and deletion, as well as enablement-link creation and deletion.

The FIM alerts were correlated with systemd status and journal records to verify the service's actual execution.

The results demonstrate useful visibility into systemd persistence-related changes. However, FIM alerts should be treated as investigation leads rather than proof of malicious activity.

Further investigation should establish whether a change was authorized, identify the process and user responsible, inspect the service configuration, and correlate the activity with relevant execution and authentication telemetry.

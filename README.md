# Lab 128 – Linux Systemd Service Persistence Investigation

## Overview

This lab investigates how Linux systemd services can be used as a persistence mechanism and how changes to service unit files and systemd enablement links can be detected using Wazuh.

The investigation was performed on a Rocky Linux 9.8 virtual machine running Wazuh 4.13.1. File Integrity Monitoring (FIM) was configured to monitor `/etc/systemd/system` in real time.

A controlled one-shot systemd service was created, enabled, executed, modified, and removed to validate detection coverage.

Custom Wazuh rules were used to identify service configuration creation, modification, and deletion events.

## Objectives

- Understand systemd service persistence on Linux.
- Identify important systemd unit and enablement locations.
- Configure Wazuh FIM to monitor `/etc/systemd/system`.
- Create custom Wazuh rules for systemd service changes.
- Detect service unit creation, modification, and deletion.
- Monitor the creation and removal of systemd enablement links.
- Validate controlled service execution using systemd and journal logs.
- Correlate FIM alerts with systemd activity.
- Verify cleanup and confirm that Wazuh remains operational.

## Environment

| Component | Details |
|---|---|
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel | 5.14.0-687.31.1.el9_8.x86_64 |
| Virtualization | VMware |
| Architecture | x86-64 |
| systemd | 252 |
| Wazuh Manager | 4.13.1 |
| Wazuh Revision | rc1 |
| Wazuh Agent | Local manager monitoring (agent ID 000) |
| Monitored Directory | `/etc/systemd/system` |
| FIM Mode | Real-time |

## Investigation Scope

The investigation focused on the following systemd locations:

- `/etc/systemd/system/` – Locally configured systemd unit files.
- `/etc/systemd/system/multi-user.target.wants/` – Enablement links associated with the multi-user target.
- `/usr/lib/systemd/system/` – Vendor-provided systemd unit files.

The controlled test service was:

`lab128-persistence-demo.service`

The service used `/usr/bin/logger` to write a harmless message to the system journal.

No malicious payloads, privilege escalation, or unauthorized persistence were introduced.

## Detection Rules

Three custom Wazuh rules were configured in:

`/var/ossec/etc/rules/local_rules.xml`

| Rule ID | Level | Detection |
|---|---:|---|
| 100640 | 12 | Systemd service or enablement-link creation |
| 100641 | 12 | Systemd service or enablement-link modification |
| 100642 | 10 | Systemd service or enablement-link deletion |

All three rules were mapped to MITRE ATT&CK technique **T1543.002 – Systemd Service**.

## Investigation Results

| Activity | Result |
|---|---|
| Service unit creation | Detected |
| Enablement-link creation | Detected |
| Service enablement | Successful |
| Controlled service execution | Successful |
| Initial journal message | Verified |
| Service unit modification | Detected |
| Modified journal message | Verified |
| Enablement-link deletion | Detected |
| Service unit deletion | Detected |
| Cleanup | Completed |
| Wazuh Manager | Active |
| Wazuh Syscheck | Running |
| Wazuh Analysisd | Running |

### Key Evidence

**1. Service creation**

Wazuh rule `100640` detected creation of:

`/etc/systemd/system/lab128-persistence-demo.service`

Timestamp: `2026-11-05 08:08:49 IST`

**2. Enablement-link creation**

Wazuh rule `100640` detected creation of:

`/etc/systemd/system/multi-user.target.wants/lab128-persistence-demo.service`

Timestamp: `2026-11-05 08:11:16 IST`

**3. Controlled service execution**

The service completed successfully with:

`status=0/SUCCESS`

The journal recorded:

`Lab 128 initial service execution`

**4. Service modification**

Wazuh rule `100641` detected changes to the service unit file.

The modified service subsequently executed successfully and logged:

`Lab 128 modified service execution`

**5. Cleanup detection**

Wazuh rule `100642` detected deletion of both:

- The systemd enablement link.
- The controlled service unit file.

## Important Observations

- Systemd persistence can involve both a service unit file and an enablement link.
- Monitoring only service execution would not provide complete visibility into service configuration changes.
- Wazuh FIM successfully detected the controlled file-system changes.
- A successful service execution does not establish malicious activity.
- Repeated execution of the same `sed -i` modification command generated additional FIM modification events because the file was rewritten.
- The `systemctl reset-failed` command returned a unit-not-loaded message after the test service had been removed. This was expected because there was no loaded failed unit state to reset.

## MITRE ATT&CK Mapping

| Technique ID | Technique | Relevance |
|---|---|---|
| T1543.002 | Create or Modify System Process: Systemd Service | Creation or modification of systemd service configurations can establish persistence. |

## Conclusion

This investigation successfully demonstrated how Wazuh can monitor Linux systemd service persistence activity through real-time File Integrity Monitoring and custom detection rules.

The controlled test validated detection of service unit creation, modification, and deletion, along with the creation and removal of its systemd enablement link.

Systemd service execution was independently verified through systemd status and journal records.

The investigation confirms that Wazuh can provide useful visibility into systemd persistence-related changes. However, a file change alone does not establish malicious intent. Further investigation of the service definition, execution context, user, parent process, and surrounding system activity would be required to determine whether an observed change is malicious.

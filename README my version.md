# Wazuh-Lab-128-Linux-Systemd-Service-Persistence-Investigation
## Overview
### 1. What Is systemd?

systemd is the system and service manager used by Rocky Linux and many other modern Linux distributions.

It is responsible for initializing the system, managing services, handling dependencies, and controlling when system components start or stop.

Systemd manages different types of units, including:

- `.service` – Services and background processes.
- `.target` – Groups of units used to represent system states.
- `.timer` – Scheduled execution of units.
- `.socket` – Socket-based activation.
- `.mount` – File-system mount points.
- `.path` – Path-based activation.

This investigation focuses specifically on `.service` units.

### 2. What Is a systemd Service Unit?

A systemd service unit is a configuration file that defines how a service should be managed.

A typical service unit contains sections such as:

#### [Unit]

Defines service metadata and dependencies.

Common directives include:

- `Description`
- `After`
- `Requires`
- `Wants`

#### [Service]

Defines how the service executes.

Common directives include:

- `Type`
- `ExecStart`
- `ExecStop`
- `User`
- `Group`
- `Restart`

#### [Install]

Defines how the service can be enabled.

A common directive is:

- `WantedBy`

For example:

`WantedBy=multi-user.target`

This allows the service to be associated with the multi-user target when enabled.

### 3. Important systemd Locations

#### /etc/systemd/system/

Contains locally configured or administrator-created systemd unit files.

Unexpected service files in this directory should be investigated.

#### /etc/systemd/system/*.wants/

Contains symbolic links used to associate units with systemd targets.

For example:

`/etc/systemd/system/multi-user.target.wants/`

When a service is enabled for a target, systemd commonly creates an enablement link in an appropriate `.wants` directory.

#### /etc/systemd/system/*.d/

Contains service-specific drop-in configuration directories.

An attacker with sufficient privileges may modify an existing service through a drop-in configuration instead of replacing the original unit file.

#### /usr/lib/systemd/system/

Contains vendor-provided systemd unit files installed by packages.

Changes here can also be security-relevant, although this lab focused on `/etc/systemd/system`.

#### /run/systemd/system/

Contains runtime systemd unit configuration.

Changes in this location may be temporary and can disappear after a reboot.

### 4. How systemd Persistence Works

An attacker who has obtained sufficient privileges may create or modify a systemd service to execute an unauthorized command.

A persistence mechanism may involve:

1. Creating a new service unit file.
2. Configuring an executable or script through `ExecStart`.
3. Enabling the service for a target.
4. Creating an enablement link.
5. Allowing the service to execute when the associated target is activated.

A service can also be modified to change its execution command, user context, dependencies, or restart behavior.

However, not every newly created or modified service is malicious. Administrators and software packages routinely make legitimate systemd changes.

### 5. Why Monitor systemd Changes?

Monitoring service configuration changes helps security teams identify possible persistence mechanisms.

Important events include:

- New service unit files.
- Modified service configurations.
- Deleted service files.
- New enablement links.
- Removed enablement links.
- Unexpected changes to service drop-ins.

A useful investigation should correlate file changes with service execution and system activity.

### 6. Wazuh File Integrity Monitoring

Wazuh FIM monitors files and directories for changes.

Depending on configuration, it can identify:

- File creation.
- File modification.
- File deletion.
- Permission changes.
- Ownership changes.
- Hash changes.
- Other file metadata changes.

For this lab, real-time monitoring was configured for:

`/etc/systemd/system`

This allowed Wazuh to detect changes to the controlled service unit and its enablement link.

### 7. Custom Detection Rules

Three custom rules were created to identify relevant systemd persistence-related file changes.

| Rule ID | Parent Rule | Purpose |
|---|---|---|
| 100640 | 554 | Detect file creation |
| 100641 | 550 | Detect file modification |
| 100642 | 553 | Detect file deletion |

The rules filtered events for the controlled service file and its enablement link.

The detections were mapped to:

**MITRE ATT&CK T1543.002 – Create or Modify System Process: Systemd Service**

### 8. Why Correlation Matters

A FIM alert identifies a change to a monitored file, but it does not independently establish that the change resulted in malicious persistence.

An analyst should examine:

- Who created or modified the service?
- Was the change authorized?
- What does `ExecStart` execute?
- Which user runs the service?
- Is the service enabled?
- When was the service executed?
- What did the service do?
- Are there related authentication or privilege-escalation events?

Useful evidence sources include:

- Wazuh FIM alerts.
- `systemctl status`.
- `systemctl cat`.
- `systemctl is-enabled`.
- `journalctl`.
- Linux authentication logs.
- Process execution telemetry.
- Audit logs, when available.

### 9. Lab Validation Approach

A harmless one-shot service was created for this investigation.

It used `/usr/bin/logger` to write a message to the system journal.

The service was:

1. Created.
2. Enabled.
3. Executed.
4. Modified.
5. Executed again.
6. Disabled.
7. Removed.

Wazuh detected the associated file creation, modification, and deletion events.

The service execution was independently verified through systemd status and journal records.

### 10. Key Takeaway

Systemd is a legitimate Linux service-management mechanism, but it can also be abused for persistence.

Monitoring service unit files and enablement links provides useful visibility into changes that may establish persistence.

The strongest investigation combines FIM alerts with service configuration, execution logs, user context, and surrounding system activity.
This lab investigates how Linux systemd services can be used as a persistence mechanism and how changes to service unit files and systemd enablement links can be detected using Wazuh.

The investigation was performed on a Rocky Linux 9.8 virtual machine running Wazuh 4.13.1. File Integrity Monitoring (FIM) was configured to monitor `/etc/systemd/system` in real time.

A controlled one-shot systemd service was created, enabled, executed, modified, and removed to validate detection coverage.

Custom Wazuh rules were used to identify service configuration creation, modification, and deletion events.
## Lab Objectives

The primary objective of this lab is to investigate Linux systemd service persistence and validate whether Wazuh can detect changes to systemd service configurations and enablement links.

### 1. Understand systemd Persistence

- Understand the role of systemd in Linux service management.
- Identify how service unit files can be used for persistence.
- Understand the purpose of service enablement links.
- Identify important systemd configuration directories.

### 2. Establish the System Baseline

- Identify the Rocky Linux version and systemd version.
- Verify Wazuh Manager health.
- Confirm that Wazuh Syscheck and Analysisd are running.
- Review enabled system services.
- Identify existing local systemd unit files and symbolic links.
- Confirm that the controlled test service does not already exist.

### 3. Configure Wazuh FIM

- Back up the existing Wazuh configuration.
- Configure real-time monitoring for `/etc/systemd/system`.
- Validate the Wazuh XML configuration.
- Validate the Syscheck and Analysisd configurations.
- Restart Wazuh Manager and verify monitoring initialization.

### 4. Implement Custom Detection Rules

- Create a rule for systemd service creation.
- Create a rule for systemd service modification.
- Create a rule for systemd service deletion.
- Include service enablement-link paths in the detection scope.
- Map detections to MITRE ATT&CK T1543.002.
- Validate the custom rule configuration.

### 5. Conduct Controlled Testing

- Create a harmless systemd one-shot service.
- Validate its ownership and permissions.
- Verify the service unit syntax.
- Enable the service and inspect its enablement link.
- Execute the service.
- Verify execution through systemd status and journal records.
- Modify the service configuration.
- Execute the modified service.
- Verify the updated journal message.

### 6. Validate Detection Coverage

- Confirm Wazuh detection of service unit creation.
- Confirm detection of enablement-link creation.
- Confirm detection of service unit modification.
- Confirm detection of enablement-link deletion.
- Confirm detection of service unit deletion.
- Correlate Wazuh alerts with the controlled systemd activity.

### 7. Cleanup and Final Validation

- Disable the controlled service.
- Remove the enablement link.
- Remove the controlled service unit file.
- Reload the systemd manager configuration.
- Verify that the test artifacts are absent.
- Confirm Wazuh Manager remains active.
- Confirm Syscheck and Analysisd remain operational.

### Expected Outcome

Wazuh should detect the creation, modification, and deletion of the controlled systemd service configuration and its enablement link through real-time FIM.

The service should execute successfully, with its activity independently confirmed through systemd and journal logs.

The lab should finish with the controlled service and enablement link removed and Wazuh operational.

## Lab Scenario

Systemd is the service and system management framework used by many modern Linux distributions, including Rocky Linux. It controls how services and other system components are started, stopped, and managed. Because systemd services can be configured to run automatically during system startup, attackers may abuse service unit files and their associated enablement links to maintain persistence on a compromised Linux host.

In this investigation, a controlled systemd persistence scenario was created on a Rocky Linux 9.8 virtual machine monitored by Wazuh. The objective was to understand how changes to systemd service configurations could be detected and investigated through File Integrity Monitoring (FIM).

A harmless test service named `lab128-persistence-demo.service` was introduced under `/etc/systemd/system/`. Its configuration used `/usr/bin/logger` to write a predefined message to the system journal. The service was then enabled, creating a symbolic link under `multi-user.target.wants`, and started to verify that the configured action executed successfully.

The investigation covered the following stages:

- **Service creation:** Introduced a new systemd unit file and examined whether Wazuh detected its creation.
- **Persistence configuration:** Enabled the service and monitored the symbolic link created under `multi-user.target.wants`.
- **Service execution:** Started the service and verified its execution through `systemctl status` and the system journal.
- **Configuration modification:** Changed the service's logging message and checked whether Wazuh detected the modification.
- **Persistence removal:** Disabled the service and removed both the enablement link and unit file, validating whether the deletion events were captured.

Wazuh FIM monitoring was configured for `/etc/systemd/system/`, with custom detection rules for file creation, modification, and deletion. The rules were mapped to MITRE ATT&CK technique **T1543.002 – Systemd Service** to provide relevant detection context.

The investigation demonstrated why monitoring only service execution is insufficient. A service unit file defines the service configuration, while its enablement link determines whether it is configured to start automatically through a systemd target. Monitoring both provides useful visibility into changes associated with potential persistence.

This was a controlled detection-validation exercise, not an investigation of an actual compromise. No malicious payload, unauthorized access, or privilege escalation was involved. The test confirmed that Wazuh could generate alerts for the creation, modification, and deletion of the selected service artifacts, providing a practical foundation for investigating suspicious systemd persistence activity.

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


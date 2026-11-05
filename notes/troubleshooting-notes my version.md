# Troubleshooting Notes 

## 1. Systemd Unit Not Loaded During Cleanup

### Observation

The following command was executed:

`systemctl reset-failed lab128-persistence-demo.service`

Systemd returned:

`Failed to reset failed state of unit lab128-persistence-demo.service: Unit lab128-persistence-demo.service not loaded.`

### Context

The controlled service had already been disabled and its unit file removed.

The systemd manager configuration had also been reloaded.

The service had previously completed successfully with `status=0/SUCCESS`.

### Explanation

`systemctl reset-failed` is used to clear failed state and related counters for a unit.

In this case, systemd reported that the unit was not loaded. There was no loaded failed-unit state to reset.

### Resolution

No corrective action was required.

The service had already been removed, and the cleanup objective was achieved.

---

## 2. Repeated Modification Events

### Observation

Wazuh rule `100641` generated three modification events:

- 08:13:48 IST
- 08:13:57 IST
- 08:14:08 IST

### Context

The following `sed -i` command was issued repeatedly:

`sed -i 's/Lab 128 initial service execution/Lab 128 modified service execution/' /etc/systemd/system/lab128-persistence-demo.service`

### Explanation

The command rewrites the file.

Repeated execution can create additional file-system changes, including inode and modification-time changes, even when the intended text replacement has already occurred.

Wazuh FIM detected these file-system changes.

### Resolution

The service configuration was verified using:

`systemd-analyze verify`

The service was reloaded and restarted.

The journal confirmed successful execution of the modified service.

### Lesson Learned

Before repeating a file modification command, inspect the current content.

For example:

`grep 'ExecStart' /etc/systemd/system/lab128-persistence-demo.service`

Avoid unnecessary rewrites because they generate additional FIM events and make investigation timelines noisier.

---

## 3. Verifying Service Execution

### Observation

The service appeared as:

`Active: active (exited)`

### Explanation

The service was configured with:

- `Type=oneshot`
- `RemainAfterExit=yes`

A one-shot service executes its configured command and exits.

With `RemainAfterExit=yes`, systemd keeps the unit in an active state after successful execution.

### Validation

The service status reported:

`status=0/SUCCESS`

The journal contained:

`Lab 128 initial service execution`

The modified execution was also recorded.

### Resolution

No corrective action was required. The observed state was expected for this test service.

---

## 4. Verifying FIM Monitoring

### Observation

The investigation required confirmation that Wazuh was monitoring `/etc/systemd/system`.

### Validation

The Wazuh log was checked for:

`Monitoring path: '/etc/systemd/system'`

The log confirmed real-time monitoring at:

`2026/11/05 07:49:14`

### Resolution

No additional change was required after monitoring initialization was confirmed.

---

## 5. Validating Custom Rules

### Observation

Custom rules were added to:

`/var/ossec/etc/rules/local_rules.xml`

### Validation

The following checks were performed:

- `xmllint --noout /var/ossec/etc/rules/local_rules.xml`
- `wazuh-analysisd -t`
- Grep verification of rule IDs 100640, 100641, and 100642.

The commands returned no reported configuration errors, and all three rules were present.

### Resolution

Wazuh Manager was restarted successfully.

The Manager was confirmed active, with Syscheck and Analysisd running.

---

## 6. Preserving Existing Custom Rules

### Observation

During the investigation, there was concern that content in `local_rules.xml` might have been accidentally removed while editing the file.

### Validation

The current file was compared against:

`local_rules.xml.pre-lab128`

The difference showed the three Lab 128 rules added to the earlier configuration.

The current file still contained:

- Lab 126 rule 100620.
- Lab 127 rules 100630–100632.
- Lab 128 rules 100640–100642.
- Earlier custom rules 100001, 100501, and 100502.

### Resolution

No restoration was necessary.

The current file was retained to preserve all existing custom rules.

### Lesson Learned

Always compare the current configuration against a known backup before restoring an entire file.

A backup created before a lab may not contain rules added by later investigations.

---


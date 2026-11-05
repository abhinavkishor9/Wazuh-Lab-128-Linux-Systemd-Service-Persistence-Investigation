# Timeline – Linux Systemd Service Persistence Investigation

**Date:** 5 November 2026  
**Timezone:** IST (UTC+05:30)  
**Environment:** Rocky Linux 9.8 / Wazuh 4.13.1

| Time (IST) | Activity | Evidence / Result |
|---|---|---|
| Before 07:49 | Wazuh services started | Start commands issued for Manager, Indexer, and Dashboard. |
| 07:49:14 | FIM monitoring initialized | Wazuh log confirmed real-time monitoring of `/etc/systemd/system`. |
| Before 08:08 | Custom rules validated | XML and Analysisd checks completed without reported errors. |
| 08:08:49 | Controlled service created | Rule 100640 detected creation of `lab128-persistence-demo.service`. |
| 08:11:16 | Service enabled | Rule 100640 detected creation of the multi-user target enablement link. |
| 08:11:50 | Initial service execution | Systemd reported `status=0/SUCCESS`; initial logger message appeared in the journal. |
| 08:13:48 | Service configuration modified | Rule 100641 detected the service unit modification. |
| 08:13:57 | Additional modification event | Rule 100641 detected another file-system modification. |
| 08:14:08 | Additional modification event | Rule 100641 detected another file-system modification. |
| 08:14:37 | Modified service execution | Journal recorded `Lab 128 modified service execution`. |
| 08:16:56 | Enablement link removed | Rule 100642 detected deletion of the multi-user target link. |
| 08:17:48 | Service unit removed | Rule 100642 detected deletion of the service unit file. |
| After 08:17:48 | Systemd cleanup validation | `reset-failed` reported that the unit was not loaded; no reset was required. |
| Final validation | Wazuh health checked | Manager active; Syscheck and Analysisd running. |

## Key Detection Events

| Rule ID | Event | Timestamp (IST) |
|---|---|---|
| 100640 | Service unit created | 08:08:49 |
| 100640 | Enablement link created | 08:11:16 |
| 100641 | Service unit modified | 08:13:48 |
| 100641 | Additional file-system modification | 08:13:57 |
| 100641 | Additional file-system modification | 08:14:08 |
| 100642 | Enablement link deleted | 08:16:56 |
| 100642 | Service unit deleted | 08:17:48 |

## Timeline Notes

- The repeated modification events were associated with repeated `sed -i` commands, not three separately intended configuration changes.
- Initial and modified service execution were independently confirmed through journal records.
- The enablement link and service unit file were deleted separately and generated separate Wazuh alerts.
- The investigation did not require a system reboot.
- The `reset-failed` message occurred after cleanup and did not indicate a failed service execution.

# Wazuh SOC Home Lab
I built a wazuh soc home-lab with windows server AD-DC,Ubuntu-server,pfsense and a domain joined windows client.i also deployed wazuh-agent,sysmon to collect logs from endpoints and simulated and failed logon and validate the account lockout policy.

<img width="1206" height="1306" alt="Wazuh-Lab-Architechture(1)" src="https://github.com/user-attachments/assets/fbb9de2e-981d-4607-a28e-646c53a6abd2" />


| Machine | OS | Role | IP |
| Wazuh server | Ubuntu | SIEM manager + dashboard | 10.10.10.x |
| Domain controller | Windows Server | Active Directory | 10.10.10.x |
| Windows client | Windows 10/11 | Endpoint with Sysmon | 10.10.10.x |
| Ubuntu server | Ubuntu | Linux endpoint | 10.10.10.x | Kali attacker machine | Linux | 10.10.10.70

# Setup Summary
1. Deployed pfSense as the gateway for the lab network.
2. Built the Active Directory domain (Windows Server domain controller, Windows client joined to the domain).
3. Installed the Wazuh server (manager and dashboard) on Ubuntu.
4. Installed Wazuh agents on the Windows endpoints and the Ubuntu server.
5. Installed Sysmon on the Windows machines and configured the agent to collect Sysmon events.
6. Verified that events reach the Wazuh dashboard.
7. Wrote custom detection rules and triggered test scenarios (failed logons, account lockouts).

 <img width="314" height="280" alt="sysmon agent conf" src="https://github.com/user-attachments/assets/e26be8bc-1388-40c6-8656-e7dc641f0d0b" />


 <img width="308" height="248" alt="sysmon status" src="https://github.com/user-attachments/assets/a5708764-b9dd-4d82-802c-7d0b758f8cc7" />


# Investigation 01: Multiple Failed Logons Followed by account lockout

| Field | Detail |
|---|---|
| **Date** | 2026-09-29 |
| **Analyst** | Michael Ayodele |
| **Alert** | Wazuh rule 60204: Multiple Windows logon failures |
| **Severity** |medium |
| **Verdict** | True Positive (simulated brute-force in lab) using Kali attacker machine |
| **MITRE ATT&CK** | T1110 Brute Force |

# Summary
Wazuh raised an alert for repeated failed logons against the account `testuser` on a joined  Windows client. The lockout threshold was reached and the account was automatically locked out, which is the intended result of the domain's account lockout policy. I triaged the alert and documented the finding below..

# Alert Details

<img width="1920" height="1011" alt="dashboard view" src="https://github.com/user-attachments/assets/e2408984-bfeb-46dd-9db6-3f83c0c92c15" />

*Wazuh dashboard showing the multiple-failures alert on the Windows client.*

# Investigation 

- **Who:** Account `testuser`
- **What:** 3 failed logons (Event ID 4625) followed by 1 account lockout (Event ID 4740)
- **Where:** Windows client `MO1`, source IP `10.10.10.XXX`
- **When:** 18:29 to 18:59 on 2026-09-29 (failures within about 4 minutes)
- **Why:** simulated brute-force attack using kali attacker machine. (using Hydra)

# Evidence


<img width="1920" height="1011" alt="failed login eventID 4625" src="https://github.com/user-attachments/assets/f14cec9a-fb74-43db-9406-8ce3a1a08c31" />

*Event ID 4625 showing logon type 3 (network) and failure reason "unknown user name or bad password".*


<img width="1920" height="1011" alt="windows failure eventID 4776" src="https://github.com/user-attachments/assets/f4815190-1783-4a23-9d71-112882f40a26" />

<img width="1920" height="1011" alt="eventID4776 details" src="https://github.com/user-attachments/assets/f80d1ae3-2b76-4e10-ae93-5d49b5926d1c" />



*Event ID 4776 showing logon type 3 (network) and failure reason "unknown user name or bad password".*


# Correlation
- Reviewed activity around the incident for `testuser`: no new processes or group changes.
- Checked for account lockout (Event ID 4740): triggered,lockout threshold was reached.
- searched across all agents, no other hosts targeted.



# Recommended Response
1. Reset the password for "testuser" and force re-authentication.
2. Block or investigate the source IP at pfSense.
3. Review other logons from that source over the past 24 hours.
4. Tune the lockout policy so this pattern locks the account earlier.


# Lessons Learned
Wazuh's Windows failed-logon rule (60204) maps to Event ID 4625, while Microsoft Sentinel surfaced the related Event ID 4771 (Kerberos pre-auth failure, code 0x18) with additional detail. Comparing both tools showed how the same incident looks different depending on the log source, and confirmed the value of correlating the failure events with the resulting lockout event rather than treating them as separate alerts.




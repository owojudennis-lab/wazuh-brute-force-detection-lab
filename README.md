# Wazuh Brute Force Detection Lab

## 📌 Overview

This lab demonstrates how to detect brute-force login attempts using custom correlation rules in Wazuh. The objective was to move beyond default alerting and implement detection logic capable of identifying repeated failed authentication attempts originating from an external attacker system.

The lab simulated a realistic blue-team workflow involving:

* Windows authentication logging
* Custom Wazuh rule creation
* Attack simulation from Kali Linux
* Alert validation and tuning

---

# 🎯 Objectives

* Understand how Wazuh processes Windows authentication logs
* Identify failed login events (Windows Event ID 4625)
* Create a custom correlation rule for brute-force detection
* Simulate attacks from an external attacker machine
* Validate alert generation and severity escalation
* Analyze attacker source IP visibility within alerts

---


# 🧰 Lab Environment

| Component               | Purpose          |
| ----------------------- | ---------------- |
| Ubuntu Server           | Wazuh Manager    |
| Windows VM              | Target endpoint  |
| Kali Linux              | Attacker machine |
| Active Directory Domain | `lab.local`      |

---

# 🌐 Network Architecture

```text id="0d83zr"
Kali Linux (Attacker)
        ↓
Windows Endpoint (Target)
        ↓
Wazuh Manager (Detection & Monitoring)
```

All systems were connected using a Host-Only Adapter network within VirtualBox.

---

# 🔍 Base Detection Analysis

During failed login attempts, Windows generated:

| Field         | Value                                        |
| ------------- | -------------------------------------------- |
| Event ID      | 4625                                         |
| Wazuh Rule ID | 60122                                        |
| Description   | Logon failure – Unknown user or bad password |

---

# 🧠 Detection Engineering Process

Initial analysis identified that:

* Rule `60122` was responsible for detecting failed Windows logins
* Multiple failed logins should be correlated into a higher-severity alert
* Default alerts alone generated excessive noise

A custom Wazuh rule was therefore created to detect repeated authentication failures within a short timeframe.

---

# ⚙️ Custom Rule Configuration

File modified:

```bash id="6rtyz5"
/var/ossec/etc/rules/local_rules.xml
```

Custom rule:

```xml id="6cij0f"
<group name="custom_rules,authentication">

  <rule id="100001" level="10" frequency="5" timeframe="60">
    <if_matched_sid>60122</if_matched_sid>
    <description>Multiple failed Windows login attempts detected (possible brute force)</description>
    <group>authentication_failed,pci_dss_10.2.4,pci_dss_10.2.5</group>
  </rule>

</group>
```

---

# 🔎 Rule Logic Breakdown

| Parameter        | Purpose                                |
| ---------------- | -------------------------------------- |
| `if_matched_sid` | Correlates Windows failed login events |
| `frequency="5"`  | Triggers after 5 failed attempts       |
| `timeframe="60"` | Detection window of 60 seconds         |
| `level="10"`     | Escalates severity to high priority    |

---

# 🧪 Attack Simulation

Brute-force login attempts were simulated from Kali Linux using RDP authentication attempts against the Windows endpoint.

Command used:

```bash id="8slx1m"
for i in {1..10}; do xfreerdp /v:192.168.56.107 /u:Administrator /p:wrongpass +auth-only /cert:ignore; done
```

---
---
<img scr="Attack code for sismulation of login attempts.png">
---

# 📊 Detection Results

## Default Detection

Wazuh generated:

* Rule ID: `60122`
* Severity Level: `5`

These alerts represented individual failed login attempts.

---

## Custom Correlation Detection

After repeated login failures:

* Rule ID: `100001`
* Severity Level: `10`

The custom rule successfully escalated repeated failures into a high-severity brute-force alert.

---

# 🌐 Source IP Visibility

The alerts included the attacker IP within:

```json id="c7jl5y"
win.eventdata.ipAddress
```

This confirmed that:

* The attack originated from the Kali attacker machine
* Wazuh successfully captured external attacker metadata

---

# ⚠️ Challenges Encountered

## 1. Incorrect Rule SID

Initial testing used an incorrect SID (`5716`) which prevented rule correlation.

Resolution:

* Identified correct Windows authentication rule (`60122`)

---

## 2. XML Configuration Errors

Improper rule formatting caused:

* Wazuh manager startup failures
* Rule parsing errors

Resolution:

* Corrected XML structure and validation process

---

## 3. RDP Connectivity Issues

Kali initially failed to connect to the Windows endpoint due to:

* Disabled RDP
* Firewall restrictions

Resolution:

* Enabled Remote Desktop
* Allowed RDP through Windows Firewall

---

## 4. Active Response Mapping Limitation

Attempts to implement automatic IP blocking revealed that Windows authentication logs expose attacker IPs as:

```text id="v1w7rt"
win.eventdata.ipAddress
```

rather than the default `srcip` field expected by Wazuh active response.

This prevented straightforward automated firewall blocking without advanced decoder customization.

---

# 🧠 Lessons Learned

* Effective detection engineering requires accurate identification of base events
* Event correlation significantly improves alert quality
* Not all alerts represent attacks; correlation reduces noise
* Windows log field mapping can impact automation workflows
* Small XML errors can break Wazuh rule processing
* External attacker simulation provides more realistic security testing than localhost testing

---

# 🚀 Future Improvements

Planned enhancements include:

* Implementing custom decoder mapping:

  ```text id="b8l6k6"
  win.eventdata.ipAddress → srcip
  ```

* Enabling Wazuh active response for automated IP blocking

* Detecting attacks targeting privileged accounts

* Integrating endpoint hardening controls:

  * Account lockout policies
  * Audit policy tuning
  * RDP hardening

---

# 📌 Conclusion

This lab successfully demonstrated how to build a custom brute-force detection workflow using Wazuh. By correlating repeated Windows authentication failures generated from an external attacker machine, the lab moved beyond default alerting into practical detection engineering.

The project also highlighted important operational considerations such as:

* event correlation,
* alert tuning,
* source IP visibility,
* and limitations in automated response workflows.

The resulting environment provides a strong foundation for future blue-team labs involving active response, endpoint hardening, and advanced threat detection.

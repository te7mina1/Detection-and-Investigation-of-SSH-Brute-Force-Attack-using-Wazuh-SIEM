# SSH Brute-Force Attack Detection and Post-Compromise Investigation Using Wazuh

<!---
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-4A90E2?style=for-the-badge)
![Linux](https://img.shields.io/badge/Platform-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![SSH](https://img.shields.io/badge/Service-SSH-222222?style=for-the-badge)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-Mapped-red?style=for-the-badge)
--->

> A practical SOC home-lab investigation of **SSH password-guessing, successful authentication, and post-authentication account activity** using **Hydra, Wazuh, and Ubuntu authentication logs**.

---

## Project Overview

This project simulates and investigates an **SSH brute-force/password-guessing attack** against an Ubuntu SSH server.

The attack was launched from **Kali Linux using Hydra**, while **Wazuh** was used to detect and investigate suspicious authentication activity.

The investigation followed the attack through the following sequence:

```text
Password Guessing
        ↓
Successful SSH Authentication
        ↓
Account Creation
        ↓
Account Deletion
```

---

## Lab Environment

| Component          | Details          |
| ------------------ | ---------------- |
| **Attacker**       | Kali Linux       |
| **Attacker IP**    | `192.168.56.1`   |
| **Victim**         | Ubuntu Linux     |
| **Victim IP**      | `192.168.56.109` |
| **Service**        | SSH              |
| **Port**           | `22/TCP`         |
| **SIEM**           | Wazuh            |
| **Attack Tool**    | Hydra            |
| **Target Account** | `ubuntu`         |

### Lab Architecture

```text
                 SSH Authentication Attempts
                           │
                           ▼
              ┌────────────────────────┐
              │       Kali Linux       │
              │        Attacker        │
              │                        │
              │        Hydra           │
              │   192.168.56.1         │
              └───────────┬────────────┘
                          │
                       SSH/22
                          │
                          ▼
              ┌────────────────────────┐
              │        Ubuntu          │
              │        Victim          │
              │                        │
              │   192.168.56.109       │
              │      SSH Server        │
              └───────────┬────────────┘
                          │
                     Authentication
                          Logs
                          │
                          ▼
              ┌────────────────────────┐
              │         Wazuh          │
              │                        │
              │ Detection & Analysis   │
              └────────────────────────┘
```
---

## Attack Simulation

The attack was simulated from **Kali Linux** using **Hydra** against the Ubuntu SSH service.

The target username was assumed to already be known; therefore, **username discovery was outside the scope of this investigation**.

### Hydra Command

```bash
hydra -l ubuntu -P sshPass.txt ssh://192.168.56.109 -t 4 -V
```

### Attack Parameters

| Parameter    | Value            |
| ------------ | ---------------- |
| **Source**   | `192.168.56.1`   |
| **Target**   | `192.168.56.109` |
| **Service**  | SSH              |
| **Port**     | `22/TCP`         |
| **Username** | `ubuntu`         |
| **Tool**     | Hydra            |

The simulation eventually identified a valid lab credential and successfully authenticated to the Ubuntu SSH service.

<img width="1485" height="772" alt="hydraAttackSuccess" src="https://github.com/user-attachments/assets/7134818a-bc84-4076-b1a3-6d711a7c0ef8" />


---

## Detection with Wazuh

During review of Wazuh security events, an SSH detection alert identified **repeated authentication failures from the same source IP**.

This alert initiated the investigation into the SSH authentication logs and surrounding activity.

### Wazuh Detection

| Field              | Finding                         |
| ------------------ | ------------------------------- |
| **Rule ID**        | `100104`                        |
| **Level**          | `10`                            |
| **Detection**      | Possible SSH password guessing  |
| **Threshold**      | 5 failed attempts               |
| **Time Window**    | 120 seconds                     |
| **Source IP**      | `192.168.56.1`                  |
| **Destination**    | `192.168.56.109`                |
| **Target Account** | `ubuntu`                        |
| **MITRE ATT&CK**   | `T1110.001 – Password Guessing` |

### Example Event

```text
Failed password for ubuntu from 192.168.56.1
```

<img width="1919" height="719" alt="numberOfRulesFired" src="https://github.com/user-attachments/assets/93762a40-7ab8-4b3c-b32f-9d028b61e3da" />

---

## Investigation Findings

### 1. Repeated Authentication Failures

Multiple failed SSH authentication attempts were observed against the `ubuntu` account from the Kali host.

The activity occurred within a short period and satisfied the threshold configured in **Wazuh Rule `100104`**.

### Finding

| Investigation Element | Result                         |
| --------------------- | ------------------------------ |
| **Source**            | `192.168.56.1`                 |
| **Target**            | `192.168.56.109`               |
| **Account**           | `ubuntu`                       |
| **Service**           | SSH                            |
| **Activity**          | Repeated failed authentication |
| **Wazuh Rule**        | `100104`                       |
| **Detection**         | Possible SSH password guessing |

---

### 2. Successful SSH Authentication

The investigation then identified a successful SSH authentication from the same source:

```text
Accepted password for ubuntu from 192.168.56.1
```

Wazuh recorded the successful authentication using **Rule `5715`**.

This correlated with the Hydra result showing successful authentication during the controlled attack simulation.

### Successful Authentication

| Field              | Finding                                                    |
| ------------------ | ---------------------------------------------------------- |
| **Source IP**      | `192.168.56.1`                                             |
| **Destination IP** | `192.168.56.109`                                           |
| **Account**        | `ubuntu`                                                   |
| **Service**        | SSH                                                        |
| **Authentication** | Successful                                                 |
| **Wazuh Rule**     | `5715`                                                     |
| **Correlation**    | Successful authentication observed during Hydra simulation |


<img width="1637" height="882" alt="authenticationSuccess" src="https://github.com/user-attachments/assets/336abbcf-6b5a-477e-848a-39fe434d2777" />


---

### 3. Post-Authentication Account Creation

Shortly after the successful SSH login, a new local account was created:

```text
backdoor-user
```

### Account Creation Details

| Attribute        | Value                                       |
| ---------------- | ------------------------------------------- |
| **Username**     | `backdoor-user`                             |
| **UID**          | `1001`                                      |
| **GID**          | `1001`                                      |
| **Home**         | `/home/backdoor-user`                       |
| **Shell**        | `/bin/sh`                                   |
| **Terminal**     | `/dev/pts/3`                                |
| **Wazuh Rule**   | `5902`                                      |
| **MITRE ATT&CK** | `T1136.001 – Create Account: Local Account` |

The account creation occurred approximately **35 seconds after the successful SSH authentication**.

<img width="1636" height="883" alt="backdoor-userCreation" src="https://github.com/user-attachments/assets/093d4e21-7944-41d7-a1e7-13c01550a629" />

---

<img width="1609" height="519" alt="backdoorUserLogs" src="https://github.com/user-attachments/assets/481853a9-82eb-4d6e-99b3-b8f00fc16374" />


---

### 4. Account Deletion

The newly created account was subsequently deleted:

```text
userdel[20436]: delete user 'backdoor-user'
```

Wazuh detected the deletion using **Rule `5903`**.

Based on the observed timestamps, the account existed for approximately **84 seconds**.

### Account Lifecycle

| Event                    | Finding         |
| ------------------------ | --------------- |
| **Account Created**      | `backdoor-user` |
| **Creation Rule**        | `5902`          |
| **Account Deleted**      | `backdoor-user` |
| **Deletion Rule**        | `5903`          |
| **Approximate Lifetime** | 84 seconds      |
| **Status**               | Later deleted   |


<img width="1609" height="525" alt="userDeletion" src="https://github.com/user-attachments/assets/bc89b521-1e37-4f6c-aebe-56631600b9cd" />

---

<img width="1600" height="525" alt="userDeletionLog" src="https://github.com/user-attachments/assets/9360e3c7-2c50-4c34-938b-6555ba6ec48e" />



---

## Key Findings

| Investigation Area               | Finding                        |
| -------------------------------- | ------------------------------ |
| **Attack Source**                | Kali – `192.168.56.1`          |
| **Target**                       | Ubuntu – `192.168.56.109`      |
| **Service**                      | SSH / TCP `22`                 |
| **Target Account**               | `ubuntu`                       |
| **Initial Activity**             | Repeated failed authentication |
| **Detection Rule**               | Wazuh `100104`                 |
| **Successful Login**             | Confirmed                      |
| **Post-Authentication Activity** | Local account created          |
| **Created Account**              | `backdoor-user`                |
| **Creation Rule**                | `5902`                         |
| **Deletion Rule**                | `5903`                         |
| **Account Status**               | Later deleted                  |

---

## Attack Timeline

| Time         | Event                                   |
| ------------ | --------------------------------------- |
| **14:02:xx** | Repeated SSH authentication failures    |
| **14:02:11** | Wazuh password-guessing alert triggered |
| **14:12:49** | Successful SSH authentication           |
| **14:13:24** | `backdoor-user` created                 |
| **14:13:25** | Wazuh detected account creation         |
| **14:14:48** | `backdoor-user` deleted                 |
| **14:14:49** | Wazuh detected account deletion         |
| **14:14:53** | SSH/PAM session closed                  |


---

## MITRE ATT&CK Mapping

| Technique                         | ID          | Tactic            | Evidence                                        |
| --------------------------------- | ----------- | ----------------- | ----------------------------------------------- |
| **Password Guessing**             | `T1110.001` | Credential Access | Repeated failed SSH authentication              |
| **Valid Accounts**                | `T1078`     | Initial Access    | Successful authentication using a valid account |
| **Remote Services: SSH**          | `T1021.004` | Lateral Movement  | Successful SSH connection                       |
| **Create Account: Local Account** | `T1136.001` | Persistence       | `backdoor-user` was created                     |

---

## Attack Chain

The observed activity can be summarized as the following attack chain:

```text
┌──────────────────────────────┐
│  SSH Password Guessing       │
│  Multiple Failed Logins      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Wazuh Detection             │
│  Rule 100104                 │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Successful SSH Login        │
│  Account: ubuntu             │
│  Rule: 5715                  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Local Account Creation      │
│  backdoor-user               │
│  Rule: 5902                  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Account Deletion            │
│  backdoor-user               │
│  Rule: 5903                  │
└──────────────────────────────┘
```

---

## SOC Investigation Summary

The investigation began with a **Wazuh password-guessing alert** generated from repeated SSH authentication failures originating from `192.168.56.1`.

Analysis of the SSH events confirmed the activity targeted the Ubuntu SSH service on **TCP/22**. A successful authentication to the `ubuntu` account was then observed from the same source.

Following the successful login, the investigation identified the creation and subsequent deletion of the `backdoor-user` account.

The evidence demonstrates the progression from:

```text
SSH Password Guessing
        ↓
Successful Authentication
        ↓
Post-Authentication Account Activity
        ↓
Local Account Creation
        ↓
Account Deletion
```

---

## Recommended Security Controls

| Security Control                                              | Objective                                       |
| ------------------------------------------------------------- | ----------------------------------------------- |
| **Disable password-based SSH authentication where practical** | Reduce exposure to password-guessing attacks    |
| **Use SSH key-based authentication**                          | Strengthen SSH authentication                   |
| **Disable direct root SSH login**                             | Reduce direct privileged access                 |
| **Restrict SSH access to trusted hosts or networks**          | Limit the attack surface                        |
| **Implement MFA where supported**                             | Add an additional authentication factor         |
| **Alert on repeated authentication failures**                 | Detect password-guessing activity               |
| **Monitor unexpected account creation**                       | Identify potential post-authentication activity |
| **Monitor privileged and sudo activity**                      | Detect suspicious privilege use                 |
| **Regularly review local Linux accounts**                     | Identify unauthorized accounts                  |
| **Maintain centralized authentication logging**               | Support investigation and correlation           |

---

## Skills Demonstrated

| Skill Area                         | Demonstrated Capability                                    |
| ---------------------------------- | ---------------------------------------------------------- |
| **Wazuh SIEM**                     | Security event detection and investigation                 |
| **SOC Alert Triage**               | Investigating authentication alerts                        |
| **SSH Log Analysis**               | Reviewing authentication events                            |
| **Brute-Force Detection**          | Identifying repeated login failures                        |
| **Detection Engineering**          | Working with Wazuh detection rules                         |
| **Event Correlation**              | Connecting authentication and account activity             |
| **Incident Timeline Construction** | Building an attack timeline                                |
| **Linux Security Monitoring**      | Monitoring SSH and local account activity                  |
| **MITRE ATT&CK Mapping**           | Mapping observed behavior to ATT&CK techniques             |
| **Post-Compromise Investigation**  | Investigating activity following successful authentication |
| **Security Documentation**         | Documenting findings and recommendations                   |

---

## Lab Disclaimer

> This project was conducted entirely within an **authorized home-lab environment** using personally controlled virtual machines.
>
> The attack simulation was performed for **defensive security research, detection engineering, incident investigation, and SOC analyst training purposes**.

---

## Project Takeaway

This lab demonstrates how a SOC analyst can use **Wazuh and Linux authentication logs** to move beyond detecting individual failed login attempts and investigate the broader sequence of activity surrounding a suspected compromise.

The investigation connected multiple events into a single timeline:

```text
Password Guessing
        ↓
Wazuh Alert
        ↓
Successful Authentication
        ↓
Account Creation
        ↓
Account Deletion
```

This highlights the importance of **event correlation, timeline analysis, and monitoring post-authentication activity** when investigating SSH security incidents.


## Author

Randy
Cybersecurity | SOC Analysis | Blue Team | Vulnerability Management

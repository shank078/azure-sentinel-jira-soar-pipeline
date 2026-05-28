# 🛡️ Cloud-Native SOAR Pipeline: Azure Sentinel SIEM to Jira ITSM

![Azure](https://img.shields.io/badge/Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Microsoft Sentinel](https://img.shields.io/badge/Microsoft%20Sentinel-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Jira](https://img.shields.io/badge/Jira-0052CC?style=for-the-badge&logo=jira&logoColor=white)
![Logic Apps](https://img.shields.io/badge/Logic_Apps-5C2D91?style=for-the-badge&logo=azure&logoColor=white)
![KQL](https://img.shields.io/badge/KQL-Kusto%20Query-0078D4?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green.svg?style=for-the-badge)

## 📌 Executive Summary
In modern enterprise Security Operations Centers (SOCs), minimizing the Mean Time to Respond (MTTR) is critical to halting lateral threat movement. Relying on analysts to manually triage and copy alert variables from a SIEM platform into an IT service management queue creates operational drag and introduces human error.

This project demonstrates the end-to-end architectural design, engineering, and deployment of a zero-touch **Security Orchestration, Automation, and Response (SOAR)** pipeline. By bridging **Microsoft Sentinel** with **Atlassian Jira via a custom REST API integration**, this cloud environment autonomously detects identity-focused brute-force attacks in real-time and dynamically provisions highly contextualized mitigation tickets to the system administration queue.

---

## 🏗️ Architectural Topology & Telemetry Flow

```mermaid
graph LR
A[Attacker Script] -- Brute Force --> B(Microsoft Entra ID)
B -- Auth Telemetry --> C{Log Analytics Workspace}
C -- KQL Detection --> D[Sentinel SIEM]
D -- Incident Trigger --> E((Azure Logic App))
E -- JSON API Payload --> F[Jira ITSM Board]
```
* **Phase 1:** Adversarial script generates rapid authentication failures against the Entra ID gateway.
* **Phase 2:** Log Analytics Workspace ingests the raw telemetry via native cloud connectors.
* **Phase 3:** Custom KQL Analytics Rules flag specific error codes and fire a High-Severity incident.
* **Phase 4:** Sentinel Automation Rules intercept the alert and instantiate a serverless Logic App.
* **Phase 5:** The playbook parses dynamic incident data and pushes an HTTP POST payload to Jira.

---

## 🚀 Lifecycle Engineering & Implementation

### Part 1: Adversarial Emulation (The Trigger)
To validate the pipeline against real-world parameters, an offensive authentication loop was initiated using PowerShell. This simulated a rapid password spray attack, forcing the Entra ID gateway to reject the requests and populate the environment with actionable threat telemetry.

**Visual Proof: The Attack Vector**<br>
<img src="images/00-attack-simulation-brute-force.png" width="800" alt="Attack Simulation" />

---

### Part 2: Cloud Infrastructure & Storage Core
A secure, sandboxed resource perimeter was established in Azure. A high-throughput Log Analytics workspace was deployed to handle the ingestion, parsing, and storage of the incoming log streams.

**Visual Proof: Infrastructure Provisioning**
* **2.1** Defining Resource Boundaries:<br>
  <img src="images/02-resource-group-creation-form.png" width="800" alt="RG Creation Form" />
* **2.2** Core Container Deployed:<br>
  <img src="images/01-resource-group-created.png" width="800" alt="RG Created" />
* **2.3** Log Engine Provisioned:<br>
  <img src="images/03-log-analytics-workspace-created.png" width="800" alt="LAW Created" />

---

### Part 3: SIEM Mobilization & Telemetry Pipelines
With the storage layer active, Microsoft Sentinel was initialized. Native data connectors were mapped into the tenant's Identity Provider (IdP) to route critical authentication logs directly into the security matrix.

**Visual Proof: Platform Initialization & Ingestion**
* **3.1** Sentinel Activation:<br>
  <img src="images/04-microsoft-sentinel-enabled.png" width="800" alt="Sentinel Enabled" />
* **3.2** Defender Unification:<br>
  <img src="images/05-sentinel-workspace-connected-to-defender.png" width="800" alt="Defender Connected" />
* **3.3** Connector Discovery:<br>
  <img src="images/06-defender-data-connectors-entra-visible.png" width="800" alt="Entra Visible" />
* **3.4** Stream Configuration:<br>
  <img src="images/07-entra-id-connector-configuration.png" width="800" alt="Entra Config" />
* **3.5** Sign-In Data Verification:<br>
  <img src="images/08-entra-data-ingestion-confirmed.png" width="800" alt="Ingestion Confirmed" />
* **3.6** Audit Log Verification:<br>
  <img src="images/09-auditlogs-query-results.png" width="800" alt="Audit Query Results" />

---

### Part 4: Behavioral Detection Engineering (KQL)
Raw tables were elevated to actionable intelligence. Heuristic detection logic was authored using the Kusto Query Language (KQL) to isolate explicit password failure patterns (Error `50126`) and filter out standard operational noise.

**Visual Proof: Analytics & Detection**
* **4.1** Behavioral Isolation via KQL:<br>
  <img src="images/10-failed-signin-kql-live-results.png" width="800" alt="Live KQL Results" />
* **4.2** Alert Logic Compilation:<br>
  <img src="images/11-analytics-rule-created.png" width="800" alt="Rule Created" />
* **4.3** Historic Threshold Simulation:<br>
  <img src="images/12-analytics-rule-simulation-results.png" width="800" alt="Rule Simulation" />

---

### Part 5: ITSM Target Platform Setup
Before orchestrating the response, the destination engineering ticketing platform (Jira) required structural preparation to securely receive programmatic external API connections.

**Visual Proof: Atlassian Configuration**
* **5.1** Dedicated SOC Board:<br>
  <img src="images/13-jira-project-created.png" width="800" alt="Jira Project" />
* **5.2** Secure API Tokenization:<br>
  <img src="images/14-jira-api-token-created.png" width="800" alt="Jira API Token" />

---

### Part 6: SOAR Development & API Orchestration
The automated response code was built using a serverless Azure Logic App. This playbook converts incoming SIEM webhooks into dynamic outbound REST API requests, passing heavily nested JSON payload arrays flawlessly.

**Visual Proof: Automation Playbook Engineering**
* **6.1** Serverless Playbook Frame:<br>
  <img src="images/15-logic-app-playbook-created.png" width="800" alt="Logic App Created" />
* **6.2** Webhook Trigger Link:<br>
  <img src="images/16-sentinel-incident-trigger-configured.png" width="800" alt="Incident Trigger" />
* **6.3** Azure API Trust Verification:<br>
  <img src="images/18-azuresentinel-api-connection-authorized.png" width="800" alt="API Authorized" />
* **6.4** Outbound JSON Payload Design:<br>
  <img src="images/17-logic-app-http-action-jira-configured.png" width="800" alt="HTTP Action Config" />

---

### Part 7: IAM Controls & Automation Routing
To ensure autonomous execution, specific Role-Based Access Control (RBAC) permissions were assigned. An orchestration routing rule was programmed to instantly fire the playbook whenever the KQL logic tripped.

**Visual Proof: Access & Orchestration**
* **7.1** SIEM Automation Link:<br>
  <img src="images/19-sentinel-automation-rule-created.png" width="800" alt="Automation Rule Created" />
* **7.2** Least-Privilege IAM Assignment:<br>
  <img src="images/20-sentinel-logic-app-permission-granted.png" width="800" alt="RBAC Permissions" />
* **7.3** Pipeline Listener Active:<br>
  <img src="images/24-automation-rule-active-confirmed.png" width="800" alt="Rule Active" />

---

### Part 8: Execution Receipts & Incident Resolution
The pipeline was validated by executing the brute-force attack script one final time. The system performed flawlessly: capturing the threat, firing the alert, executing the serverless playbook, and autonomously generating the Jira issue.

**Visual Proof: The Complete Automated Lifecycle**
* **8.1** The SIEM Incident Triggers:<br>
  <img src="images/21-sentinel-incidents-created.png" width="800" alt="Incident Created" />
* **8.2** The Playbook Executes:<br>
  <img src="images/22-playbook-triggered-successfully.png" width="800" alt="Playbook Triggered" />
* **8.3** API Handshake Success (201 Created):<br>
  <img src="images/22b-logic-app-run-history-success.png" width="800" alt="Run History Success" />
* **8.4** Ticket Hits the Jira Board:<br>
  <img src="images/23-jira-ticket-auto-created.png" width="800" alt="Ticket Auto-Created" />
* **8.5** Deep JSON Payload Mapping Verified:<br>
  <img src="images/23b-jira-ticket-details-mapped.png" width="800" alt="Ticket Details Mapped" />

---

## 🧠 Core Competencies Proved
* **Cloud Infrastructure Engineering:** Architecting scalable storage environments using IaaS/PaaS paradigms in Microsoft Azure.
* **Modern SIEM Administration:** Configuring cross-platform security connectors and designing high-velocity ingestion pipelines.
* **Threat Detection via KQL:** Sifting through intensive identity logs to isolate behavioral threat signatures using the Kusto Query Language.
* **SOAR API Integration:** Developing serverless Logic App workflows to manipulate and format complex JSON arrays across REST endpoints.

---

## 👨‍💻 About the Author
I am a Junior Cyber Security Analyst and IT Support Specialist based in Canberra, ACT. I hold a Master of Information Technology (Cyber Security) from Charles Sturt University, graduating with a 4.92 GPA. 

Currently, I operate as an IT Support Specialist at Extratech, where I manage high-volume enterprise escalations, administer Microsoft Intune and Entra ID identity lifecycles, and monitor perimeter defenses. I specialize in bridging daily IT operations with proactive security measures, focusing on Azure/M365 administration and aligning environments with ASD Essential Eight frameworks.

Beyond enterprise operations, I actively design and deploy cloud-native security pipelines. My hands-on engineering experience includes deploying global threat intelligence dashboards, configuring zero-touch Windows Autopilot environments, and building Microsoft Sentinel SIEM architectures capable of intercepting and visualizing thousands of live brute-force attacks. I am currently advancing my SOC capabilities by pursuing the Blue Team Level 1 (BTL1) and Microsoft SC-200 certifications.

<br>

<div align="center">
  <a href="https://www.linkedin.com/in/shankarbaral1" target="_blank">
    <img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn Profile" />
  </a>
</div>

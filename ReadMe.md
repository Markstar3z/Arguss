# ARGUS: Automated Reconnaissance & Defensive Network Monitoring

ARGUS is a lightweight, host-and-network defense monitoring platform designed to maintain continuous situational awareness across local subnets. Built with a deterministic detection engine, ARGUS establishes baseline network profiles, detects unapproved host activity and port drift, and coordinates incident triage through an explainable risk-scoring model.

---

## Table of Contents
1. [Architectural Overview](#architectural-overview)
2. [Disambiguation: Detection Cycle vs. Alert Lifecycle](#disambiguation-detection-cycle-vs-alert-lifecycle)
3. [The ARGUS Detection Cycle](#the-argus-detection-cycle)
4. [The Alert Lifecycle](#the-alert-lifecycle)
5. [Tech Stack & Dependencies](#tech-stack--dependencies)
6. [Installation & Quickstart](#installation--quickstart)
7. [Conclusion & Project Impact](#conclusion--project-impact)

---

## Architectural Overview

ARGUS enforces a strict separation of concerns between its continuous background analysis engine and its analyst-facing incident triage system:

```
+───────────────────────────────────────────────────────────────────────────+
|                         ARGUS DETECTION ENGINE                            |
|                                                                           |
|   [ 1. Discovery ] ──> [ 2. Baseline Diff ] ──> [ 3. Threat Heuristics ]  |
|                                                          │                |
|                                                          ▼                |
|                                                [ 4. Risk Scoring ]        |
+──────────────────────────────────────────────────────────┬────────────────+
                                                           │ Alert Emitted
                                                           ▼
+───────────────────────────────────────────────────────────────────────────+
|                         ALERT & INCIDENT LIFECYCLE                        |
|                                                                           |
|   [ NEW / OPEN ] ──> [ ACKNOWLEDGED ] ──> [ UNDER INVESTIGATION ]         |
|                                                    │                      |
|                                 ┌──────────────────┴──────────────────┐   |
|                                 ▼                                     ▼   |
|                          [ RESOLVED ]                           [ SUPPRESSED ]
+───────────────────────────────────────────────────────────────────────────+
```

---

## Disambiguation: Detection Cycle vs. Alert Lifecycle

A common source of confusion in intrusion detection tooling is conflating the **telemetry evaluation loop** with the **incident management workflow**. ARGUS treats them as two distinct operations:

| Parameter | 1. The Detection Cycle | 2. The Alert Lifecycle |
| :--- | :--- | :--- |
| **Primary Actor** | Automated Core Engine / Daemon | Security Analyst / Operator |
| **Trigger Mechanism** | Scheduled interval or raw network frame | High-risk event emitted by detection engine |
| **Operational Domain** | Subnet discovery, ARP/IP tables, port states | Database ticket status, investigation notes |
| **Nature of Process** | Cyclic, stateless inspection against baseline | Stateful progression through workflow states |
| **End State** | Emits scored alerts or logs clean pass | Marked as `RESOLVED` or `SUPPRESSED` |

---

## The ARGUS Detection Cycle

The Detection Cycle is an automated background pipeline responsible for interrogating the local network and assessing deviations against a known-good configuration baseline.

```
       ┌─────────────────────────────────────────────────────────┐
       │         Step 1: Network Ingestion & Active Discovery     │
       └────────────────────────────┬────────────────────────────┘
                                    │
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │         Step 2: Baseline Configuration Comparison       │
       └────────────────────────────┬────────────────────────────┘
                                    │
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │         Step 3: Threat Heuristics & Anomaly Analysis    │
       └────────────────────────────┬────────────────────────────┘
                                    │
                                    ▼
       ┌─────────────────────────────────────────────────────────┐
       │         Step 4: Deterministic Scoring & Alert Dispatch  │
       └─────────────────────────────────────────────────────────┘
```

### Detailed Engine Stages

1. **Step 1: Network Ingestion & Active Discovery**
   * Transmits ARP probes across configured CIDR blocks to map active MAC-to-IP pairings.
   * Performs non-intrusive TCP SYN/Connect scans against critical service ports (e.g., 22, 80, 443, 445, 3389).
   * Gathers device hardware metadata (OUI lookups) and hostname broadcast details.

2. **Step 2: Baseline Configuration Comparison**
   * Pulls the authorized "golden baseline" profile from the database.
   * Runs a differential analysis between active observations and baseline records:
     * **New Hosts:** IP/MAC combinations not present in baseline.
     * **Missing Hosts:** Devices present in baseline but unresponsive.
     * **Port Drift:** New open ports or unexpectedly closed services on approved hosts.

3. **Step 3: Threat Heuristics & Anomaly Analysis**
   * Cross-references discovered differences with threat models:
     * *ARP Spoofing / Poisoning:* Single IP claimed by conflicting MAC addresses.
     * *Unauthorized Service Exposure:* Unapproved management ports (e.g., Telnet, SSH, RDP) appearing on standard endpoints.
     * *Rogue Infrastructure:* New devices attempting to broadcast DHCP offer packets or masquerade as default gateways.

4. **Step 4: Deterministic Scoring & Alert Dispatch**
   * Computes risk rating based on heuristic weights:
     $$R = \sum_{i=1}^{n} (w_i \cdot s_i)$$
     Where $w_i$ denotes the heuristic weight and $s_i$ denotes the severity scale of the finding.
   * If risk rating $R$ exceeds threshold $T$, an alert record is committed to the persistence layer and flagged for analyst triage.

---

## The Alert Lifecycle

Once the Detection Cycle creates an alert, it enters the Alert Lifecycle. This is an incident management state machine operated by human analysts to ensure alerts are triaged, investigated, and remediated without alert fatigue.

```
                  ┌──────────────────────┐
                  │      NEW / OPEN      │
                  └──────────┬───────────┘
                             │ (Analyst assigns/triages)
                             ▼
                  ┌──────────────────────┐
                  │     ACKNOWLEDGED     │
                  └──────────┬───────────┘
                             │ (Deep dive & correlation)
                             ▼
                  ┌──────────────────────┐
                  │ UNDER INVESTIGATION  │
                  └──────────┬───────────┘
                             │
            ┌────────────────┴────────────────┐
            │                                 │
            ▼                                 ▼
┌───────────────────────┐         ┌───────────────────────┐
│       RESOLVED        │         │      SUPPRESSED       │
│ (Remediation Applied) │         │ (Authorized Change /  │
│                       │         │     False Positive)   │
└───────────────────────┘         └───────────────────────┘
```

### State Definitions

* **State 1: New / Open**
  * The initial state committed by the Detection Engine.
  * Represents an unreviewed anomaly requiring analyst confirmation.
* **State 2: Acknowledged**
  * An operator claims the alert, setting status to acknowledged to avoid duplicate triage efforts across security staff.
* **State 3: Under Investigation**
  * The analyst actively inspects host history, packet captures, and host authentication logs to determine root cause.
* **State 4: Terminal Resolutions**
  * **Resolved:** The anomaly was malicious or unauthorized and has been mitigated (e.g., rogue device physically unplugged, switch port closed, or host isolated).
  * **Suppressed:** The anomaly was legitimate operational maintenance (e.g., approved server deployment). The operator approves updating the golden baseline to prevent recurring alerts.

---

## Tech Stack & Dependencies

* **Language:** Python 3.10+
* **Packet Capture & Discovery:** Scapy, Python Native Sockets
* **Web Framework & API:** Flask
* **Data Storage:** SQLite3
* **Frontend UI:** HTML5, CSS3, JavaScript (Fetch API)

---

## Installation & Quickstart

### Prerequisites
* Linux (recommended for raw socket performance) or Windows with Npcap installed.
* Python 3.10 or higher.
* Administrative/root privileges (required for raw packet socket access).

### Setup Instructions

1. **Clone Repository:**
   ```bash
   git clone https://github.com/your-username/argus.git
   cd argus
   ```

2. **Initialize Environment:**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **Initialize Database:**
   ```bash
   python init_db.py
   ```

4. **Launch ARGUS:**
   ```bash
   # Elevated privileges required for ARP/RAW sockets:
   sudo python app.py  # On Windows: Run terminal as Administrator
   ```
   Open your browser to `http://localhost:5000`.

---

## Conclusion & Project Impact

ARGUS resolves key architectural issues common to entry-level intrusion detection systems:
1. **Elimination of Architectural Confusion:** By strictly delineating the automated **Detection Cycle** (machine-driven telemetry) from the **Alert Lifecycle** (human-driven incident response), ARGUS ensures clean separation between monitoring loops and ticket state machines.
2. **Deterministic Baseline Integrity:** Rather than relying on unreliable black-box anomaly guessing, ARGUS utilizes explainable, deterministic baseline differentials to guarantee repeatable alerts.
3. **Structured Alert Triage:** Standardized incident states prevent alert fatigue, providing an audit trail from initial packet discovery to ticket resolution.

ARGUS proves that defensive network monitoring can remain lightweight, performant, and maintainable on commodity hardware while preserving enterprise-grade operational discipline.
# ARGUS

## Defensive Network Intelligence Platform

ARGUS is a defensive network intelligence and monitoring platform designed to observe authorized network assets, establish behavioral baselines, detect changes, track alerts, notify the operator, and provide an investigation oriented dashboard.

The project focuses on turning repeated network observations into useful security intelligence rather than treating individual scans as isolated results.

## Core Capabilities

* Authorized network host discovery
* Asset inventory
* Service and port discovery
* Observation history
* Baseline establishment
* Baseline deviation detection
* Service change detection
* Detection event generation
* Alert lifecycle management
* Alert resolution tracking
* Instant alert notifications through Telegram, email, or webhook
* Risk scoring
* Historical observation comparison
* Investigation oriented dashboard

## How ARGUS Works

ARGUS works in two stages.

* The **detection cycle** finds changes on the network. It runs on every scan.
* The **alert lifecycle** manages each change once it has been found.

The detection cycle produces detection events. The alert lifecycle takes over from there.

## Detection Cycle

Every scan runs through the same steps:

```text
Scan
  ↓
Record Observation
  ↓
Compare Against Baseline
  ↓
Detect Changes
  ↓
Generate Detection Events
```

1. **Scan:** discover hosts and the services running on them.
2. **Record Observation:** save the state of each asset with a timestamp.
3. **Compare Against Baseline:** check the observation against the asset's reference state.
4. **Detect Changes:** find services that were added, removed, or changed.
5. **Generate Detection Events:** turn each difference into an event with a severity and risk score.

An asset needs a baseline before it can be compared. If none exists, ARGUS cannot detect changes for that asset. Set one after a scan with:

```bash
python set_baseline.py
```

## Alert Lifecycle

Each detection event feeds the alert lifecycle:

```text
Alert Created  →  Operator Notified
      ↓
Active
      ↓
Updated on Every Scan While the Condition Persists
      ↓
Condition Clears
      ↓
Resolved
```

* **Alert Created:** a new event opens an alert and sends a notification.
* **Active:** the alert stays open while the condition exists.
* **Updated:** repeat detections raise the occurrence count. No repeat notification is sent.
* **Resolved:** when a later scan shows the condition is gone, the alert closes automatically.

Each alert tracks its first observation, latest observation, occurrence count, first seen time, last seen time, resolution time, and current status.

## Example

A controlled test shows both stages.

Starting state: asset `192.168.29.128` has no services, and that state is the baseline.

A temporary web server is started on port 8080.

```text
DETECTION CYCLE
New scan records the observation
  ↓
Compared with the baseline
  ↓
Service added: 8080/tcp (HTTP)
  ↓
Detection event generated

ALERT LIFECYCLE
Alert created and operator notified
  ↓
Active until the condition clears
```

The web server is then stopped.

```text
DETECTION CYCLE
New scan records the observation
  ↓
Service 8080/tcp is no longer present

ALERT LIFECYCLE
Condition cleared
  ↓
Alert resolved
```

## Alert Notifications

ARGUS pushes new alerts to you, so you do not have to sit at the terminal or keep the dashboard open.

Supported channels:

* Telegram
* Email (SMTP)
* Webhook (Discord bridges, Slack bridges, n8n, and similar tools)

How it behaves:

* A notification is sent once, when an alert is first created.
* If the same condition persists across later scans, no repeat notifications are sent.
* If a notification fails, the scan carries on. Notifications never break detection.
* If no channel is configured, nothing is sent and ARGUS works as before.
* The dashboard top bar shows whether notifications are on.

### Setup

Copy the example file and fill in only the channels you want:

```bash
cp .env.example .env
```

On Windows PowerShell, use `copy .env.example .env` instead.

Telegram (recommended, no password needed):

1. Message @BotFather on Telegram and create a bot to get a token.
2. Send any message to your new bot.
3. Open this address in a browser, with your token filled in, and find the number after `"chat":{"id":`:

```text
https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
```

4. Add both values to `.env`:

```text
ARGUS_TELEGRAM_TOKEN=your_bot_token
ARGUS_TELEGRAM_CHAT_ID=your_chat_id
```

Email (optional):

```text
ARGUS_SMTP_HOST=smtp.gmail.com
ARGUS_SMTP_PORT=587
ARGUS_SMTP_USER=you@gmail.com
ARGUS_SMTP_PASSWORD=your_app_password
ARGUS_EMAIL_TO=you@gmail.com
```

For Gmail, use an app password, never your real account password. App passwords need 2 step verification turned on.

Webhook (optional):

```text
ARGUS_WEBHOOK_URL=https://your-webhook-url
```

Other settings:

```text
ARGUS_NOTIFY_MIN_SEVERITY=informational
ARGUS_DASHBOARD_URL=http://127.0.0.1:5000
```

`ARGUS_NOTIFY_MIN_SEVERITY` can be informational, low, medium, high, or critical. Only alerts at or above this level are sent. The default is informational, so every new alert notifies you.

`ARGUS_DASHBOARD_URL` is the address used for the alert link inside each message.

Always run ARGUS from the project folder. The `.env` file and the database are both found relative to the folder you run from.

### Test it

```bash
python -m core.notifier
```

This sends a test message to every configured channel and prints whether each one worked.

### Keep your secrets safe

The `.env` file holds your tokens and passwords. It is listed in `.gitignore`, so it should never be committed. Only `.env.example` belongs in the repository.

## Risk Assessment

ARGUS includes a lightweight risk scoring system.

Risk calculations can consider:

* Event severity
* Event type
* Baseline deviation
* Alert state
* Service exposure
* Repeated occurrences

Each risk result contains a score, a level, and the reasons behind it.

Risk scoring is intended to support investigation and prioritization. It is not intended to replace human analysis.

## Dashboard

ARGUS includes a Flask based investigation dashboard with a dark, minimal interface. Alerts and events carry a colored edge that reflects their severity, and a notification indicator in the top bar shows whether alerts are being pushed to you.

The dashboard provides visibility into:

* Network assets
* Current asset state
* Observation history
* Baselines
* Detection events
* Active alerts
* Alert details
* Observation comparisons
* Risk information

Start the dashboard from the project folder:

```bash
python -m dashboard.app
```

Then open:

```text
http://127.0.0.1:5000
```

The dashboard has no login and runs in debug mode. Keep it on your own machine and do not expose it to a network.

## Project Structure

```text
argus/
│
├── core/
│   ├── alerts.py
│   ├── baseline.py
│   ├── database.py
│   ├── detection.py
│   ├── history.py
│   ├── models.py
│   ├── notifier.py
│   ├── parser.py
│   ├── risk.py
│   ├── scanner.py
│   └── main.py
│
├── dashboard/
│   ├── app.py
│   ├── data.py
│   │
│   ├── templates/
│   │   ├── base.html
│   │   ├── dashboard.html
│   │   ├── asset.html
│   │   ├── observation.html
│   │   ├── alert.html
│   │   └── compare.html
│   │
│   └── static/
│       └── css/
│           ├── style.css
│           └── theme.css
│
├── main.py
├── set_baseline.py
├── .env.example
├── README.md
└── .gitignore
```

## Core Components

**Scanner**
Responsible for network discovery and service scanning.

**Parser**
Processes scanner output into structured network information.

**Database**
Stores assets, observations, services, baselines, events, and alerts using SQLite.

**Baseline Engine**
Establishes a reference state and compares new observations against it.

**Detection Engine**
Identifies changes between observations and baseline states.

**Alert System**
Creates, updates, and resolves alerts based on detected conditions.

**Notifier**
Pushes newly created alerts to Telegram, email, or a webhook. It uses only the Python standard library.

**Risk Engine**
Calculates contextual risk scores for detection events and alerts.

**History**
Maintains the historical record of observations for each asset.

**Dashboard**
Provides the investigation and visualization layer through Flask. Styling lives in `style.css`, with the visual theme layered on top in `theme.css`.

## Database Model

ARGUS currently uses SQLite for persistent monitoring data.

The main entities are:

**Assets**
Represents discovered network hosts.

**Observations**
Represents the state of an asset at a specific point in time.

**Services**
Represents services discovered during an observation.

**Baselines**
Defines the established reference state for an asset.

**Events**
Represents changes detected between network states.

**Alerts**
Tracks detected conditions through their lifecycle.

## Observation History

Every scan produces an observation associated with an asset.

An observation can contain:

* Asset state
* Observation timestamp
* Service information
* Detection events
* Baseline relationship

This creates a historical record that can be used to investigate changes over time.

## Baseline Monitoring

ARGUS compares new observations against a reference observation called a baseline.

Set the baseline after a scan, while the network is in the state you consider normal:

```bash
python set_baseline.py
```

To set it for one asset only, pass its address:

```bash
python set_baseline.py 192.168.1.5
```

Future observations are then compared against that baseline. The comparison identifies:

* Added services
* Removed services
* Changed services

For example:

```text
Baseline
    ↓
0 services

Current Observation
    ↓
1 service

Difference
    ↓
1 service added
```

A baseline deviation is a signal for investigation, not automatic proof of malicious activity.

## Observation Comparison

ARGUS can compare two specific observations.

```text
Observation #5
      ↓
Observation #14
```

The comparison can identify:

* Added services
* Removed services
* Changed service metadata

This makes it possible to investigate exactly what changed between two points in time.

## Technology Stack

ARGUS is currently built with:

* Python
* SQLite
* Flask
* HTML
* CSS
* Nmap for network scanning

## Requirements

* Python 3
* Nmap installed and available in your PATH
* Flask (`pip install flask`)

On Windows, install Nmap from nmap.org and let it install Npcap. For the best discovery results, run scans from a terminal opened as Administrator on Windows, or with root on Linux.

## Running ARGUS

1. Enter the project folder and create a virtual environment

Linux or macOS:

```bash
cd ~/argus
python3 -m venv venv
source venv/bin/activate
pip install flask
```

Windows PowerShell:

```powershell
cd C:\path\to\argus
python -m venv venv
venv\Scripts\Activate.ps1
pip install flask
```

2. Optional: set up notifications (see Alert Notifications above)

3. Run a scan and enter an authorized network or a single device when asked

```bash
python main.py
```

Examples of what to enter: `192.168.1.0/24` for a whole network, or `192.168.1.23/32` for one device.

4. Set the baseline while the network is in its normal state

```bash
python set_baseline.py
```

5. Start the dashboard

```bash
python -m dashboard.app
```

6. Open the dashboard

```text
http://127.0.0.1:5000
```

7. Scan again later to detect changes. Alerts appear in the dashboard and are pushed to your configured channels.

ARGUS only detects changes when a scan is run. To monitor continuously, schedule `main.py` to run at regular intervals with cron on Linux or Task Scheduler on Windows.

## Development Checks

Before committing changes, Python files can be checked with:

```bash
python -m py_compile \
    dashboard/app.py \
    dashboard/data.py \
    core/alerts.py \
    core/notifier.py \
    core/risk.py
```

A successful compilation produces no output.

## Defensive Scope

ARGUS is designed for:

* Authorized network monitoring
* Defensive security research
* Network visibility
* Security learning
* Controlled laboratory environments

Only scan or monitor systems and networks for which you have explicit authorization.

## Current Project Status

ARGUS currently has a functioning:

* Network discovery pipeline
* Asset inventory
* Observation system
* Service tracking
* Baseline system
* Baseline comparison
* Change detection
* Detection event system
* Alert lifecycle management
* Alert resolution
* Alert notifications (Telegram, email, webhook)
* Risk assessment
* Observation comparison
* Flask investigation dashboard

The frontend can continue to be refined independently without changing the underlying detection architecture.

## Roadmap

Future improvements may include:

* Scheduled automated monitoring
* Saving each device as it is scanned, for faster feedback on large networks
* Setting baselines from the dashboard
* Improved dashboard visualizations
* Better observation timeline presentation
* More advanced risk analysis
* Expanded detection rules
* Additional network intelligence
* Authentication and access control
* Notification rules per asset or per event type
* Notifications when alerts resolve
* Reporting and export capabilities
* Improved investigation workflows

## Project Philosophy

ARGUS is built around a simple idea:

A network is not just a collection of machines. It is a changing system.

Understanding those changes over time makes it possible to build better visibility, better detection, and better investigations.

## License

This project is currently under active development.

License information will be added as the project matures.
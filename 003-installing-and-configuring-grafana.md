# 🛠️ Installing & Configuring Grafana

A practical reference guide for getting Grafana up and running — covering Grafana Cloud vs on-premises, installing Grafana on Ubuntu, customizing `grafana.ini`, and launching a full observability stack (Grafana, Prometheus, Loki, Tempo, and Alloy) with Docker Compose.

---

## 📋 Table of Contents

- [Introduction](#-introduction)
  - [What This Material Covers](#what-this-material-covers)
  - [Prerequisites](#prerequisites)
- [Deployment Options](#-deployment-options)
  - [Grafana Cloud](#grafana-cloud)
  - [On-Premises](#on-premises)
  - [Cloud vs On-Premises](#cloud-vs-on-premises)
  - [Factors to Consider Before Choosing](#factors-to-consider-before-choosing)
- [Installing Grafana on Ubuntu](#-installing-grafana-on-ubuntu)
  - [Step 1: Update the System](#step-1-update-the-system)
  - [Step 2: Install Dependencies](#step-2-install-dependencies)
  - [Step 3: Download the Package](#step-3-download-the-package)
  - [Step 4: Install the Package](#step-4-install-the-package)
  - [Step 5: Enable and Start the Service](#step-5-enable-and-start-the-service)
  - [Step 6: Verify the Service](#step-6-verify-the-service)
  - [Accessing the Grafana UI](#accessing-the-grafana-ui)
- [Configuring Grafana](#-configuring-grafana)
  - [Configuration File](#configuration-file)
  - [Back Up Before Editing](#back-up-before-editing)
  - [Log Files](#log-files)
  - [External Database](#external-database)
  - [Saving and Applying Changes](#saving-and-applying-changes)
- [Running the Stack with Docker Compose](#-running-the-stack-with-docker-compose)
  - [Preparation](#preparation)
  - [Start the Stack](#start-the-stack)
  - [Stack Components](#stack-components)
  - [Accessing Grafana](#accessing-grafana)
  - [Checking the Data Sources](#checking-the-data-sources)
  - [Modifying Component Configuration](#modifying-component-configuration)
- [Quick Reference](#-quick-reference)
- [Best Practices](#-best-practices)

---

## 🎯 Introduction

### What This Material Covers

- The two ways to run Grafana: **Grafana Cloud** (managed) and **on-premises** (self-hosted)
- Step-by-step installation of Grafana on an Ubuntu machine
- Customizing Grafana through `grafana.ini` (log location, external database)
- Running the complete stack with Docker Compose — no need to install each component by hand
- Builds on the concepts in [001 — Foundations of Observability](001-foundations-of-observability.md)

### Prerequisites

- **Ubuntu machine** — A server or virtual machine you can reach remotely, with `sudo` access (for the on-premises section)
- **Docker Desktop** — Installed and running on your local computer (for the Docker Compose section)
- **Web browser** — To open the Grafana UI

---

## 🌐 Deployment Options

> **Analogy:** Grafana Cloud is like renting a fully furnished apartment — you move in today and the landlord handles every repair, but you pay rent and live by the building's rules. On-premises is like building your own house — total freedom and control, but every leaky pipe is your problem.

### Grafana Cloud

- A **fully managed**, cloud-based observability platform for collecting **metrics, logs, and traces**
- Offers a **free plan** — ideal for learning and practice
- Sign up for a free account at [grafana.com/products/cloud](https://grafana.com/products/cloud)

**Pros**

- **Practical** — No setup, configuration, or infrastructure deployment
- **Always up to date** — Automatically upgraded to the latest stable version
- **Ready to use** — Complex infrastructure is available out of the box

**Cons**

- **High cost at scale** — Unlike the open-source version, large-scale usage gets expensive
- **Vendor lock-in** — You are tied to the provider's infrastructure
- **Security & privacy concerns** — Your data and metrics live on third-party infrastructure

### On-Premises

- You **install and run Grafana yourself** on your own machines
- Can be used **for free** through the open-source version

**Pros**

- **Full control** — Complete authority over your data and deployments
- **Higher security** — Easier to guarantee security and regulatory compliance
- **Unlimited customization** — Adapt the system to needs a cloud version may not support
- **Potentially free** — The open-source version costs nothing

**Cons**

- **Maintenance overhead** — Needs a dedicated team to run and maintain the platform
- **Scalability challenges** — You must design the infrastructure to scale with growing workloads
- **Manual updates** — Upgrading to newer versions is your job

### Cloud vs On-Premises

| | Grafana Cloud | On-Premises |
| --- | --- | --- |
| Setup effort | None — ready to use | You install and configure everything |
| Updates | Automatic | Manual |
| Cost | Free plan for learning; expensive at large scale | Free with open source, but needs people to run it |
| Control | Limited to what the provider offers | Full control over data and deployment |
| Security & compliance | Data stored on third-party infrastructure | Easier to guarantee compliance |
| Customization | Limited to out-of-the-box features | Unlimited |
| Scalability | Handled by the provider | You design it yourself |
| Best for | Learning, small teams, fast start | Strict compliance, deep customization |

### Factors to Consider Before Choosing

For a **production** environment, weigh both options against these questions:

| Factor | Question to Ask |
| --- | --- |
| **Team resources** | Do we have enough people to maintain an observability platform ourselves? |
| **Customization needs** | Are the out-of-the-box features of Grafana Cloud enough, or do we need deep customization? |
| **Security & compliance** | Does the company have strict data-storage policies (e.g. GDPR in Europe restricting data transfer to external environments)? |
| **Budget** | How much money is available to build and maintain the platform? |

> **Key Insight:** Self-hosting is not the only option. Grafana Cloud's free plan is the fastest way to start learning — self-host when control, compliance, or cost at scale demand it.

---

## 🐧 Installing Grafana on Ubuntu

> **Analogy:** Installing Grafana by hand is like assembling flat-pack furniture. First you clear the room (update the system), then you gather the tools (dependencies), then you build it (install the package), and finally you plug it in (start the service).

```text
apt-get update
  ↓
Install dependencies
  ↓
Download the .deb package
  ↓
dpkg -i
  ↓
systemctl enable --now
  ↓
Verify: active (running)
```

The examples below use a fresh Ubuntu virtual machine accessed remotely, so the whole installation is shown from a clean system.

### Step 1: Update the System

Refresh the package repositories right after connecting to the server:

```bash
sudo apt-get update
```

> **Note:** This step is mandatory — without it, package dependencies may not download and install correctly.

### Step 2: Install Dependencies

Grafana needs a few supporting libraries to run properly on Linux:

```bash
sudo apt-get install -y adduser libfontconfig1 musl
```

| Package | Purpose |
| --- | --- |
| `adduser` | User management |
| `libfontconfig1` | Font management |
| `musl` | A C library widely used by software on Linux |

> **Note:** The `-y` flag automatically answers "yes" to every confirmation prompt.

### Step 3: Download the Package

Download the official Debian package (`.deb`) from Grafana's website. You can use either `wget` or `curl`.

```bash
wget -4 https://dl.grafana.com/grafana/release/13.3.1/grafana_13.3.1_42849527790_linux_amd64.deb
```

| Architecture | Package Variant |
| --- | --- |
| Standard Intel/AMD | `amd64` |
| ARM | `arm64` |

> **Note:** These commands use version `13.3.1`. If you download a different version, use the same file name in the next step.

### Step 4: Install the Package

Install the downloaded file with the `dpkg` package manager:

```bash
sudo dpkg -i grafana_13.3.1_42849527790_linux_amd64.deb
```

### Step 5: Enable and Start the Service

The package ships with a **system service** file, so you only need to reload `systemd`, enable the service to start on boot, and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
```

- **`daemon-reload`** — Reloads the `systemd` configuration
- **`enable`** — Makes Grafana start automatically at boot
- **`--now`** — Also starts the service immediately

### Step 6: Verify the Service

Check that Grafana is running without problems:

```bash
sudo systemctl status grafana-server
```

- **Expected result:** the status shows **`active (running)`**

### Accessing the Grafana UI

1. Open a browser and go to `http://<IP-ADDRESS>:3000` — your server's public IP followed by Grafana's default port, `3000`
2. On the login page, use the default credentials:

| Field | Value |
| --- | --- |
| **Username** | `admin` |
| **Password** | `admin` |

3. Grafana immediately asks you to change the password

> **Gotcha:** `admin` / `admin` is public knowledge. Always set a new password on first login.

---

## 🔧 Configuring Grafana

> **Analogy:** Editing the config file is like rearranging a shop's layout. Take a photo of the original first so you can restore it if things go wrong — and remember the new layout only opens to customers after the shop closes and reopens.

### Configuration File

- The main configuration file is **`/etc/grafana/grafana.ini`**
- Use it to change things like **where logs are stored** or **which database Grafana uses**
- The file location is the same whether Grafana runs directly on Ubuntu or inside Docker

### Back Up Before Editing

Make a copy of the original file so you have a fallback if you make a mistake:

```bash
# Run inside /etc/grafana
sudo cp grafana.ini custom.ini
```

### Log Files

- By default, Grafana logs are stored in **`/var/log/grafana`**
- To move them elsewhere, change the path in the configuration file

> **⚠️ Very Important:** Remove the **semicolon (`;`)** at the start of any line you change. A semicolon marks a line as a **comment**. The most common mistake is editing the value but forgetting the semicolon — so the change never takes effect.

```ini
; Commented out — has no effect
;logs = /var/log/grafana

; Active — Grafana reads this line
logs = /var/log/grafana
```

### External Database

By default, Grafana uses **SQLite**, stored locally on the same server. In about 90% of cases SQLite is more than enough, because you can always back up dashboards and data sources as JSON files.

Switch to an external database (**MySQL** or **PostgreSQL**) in these two situations:

| Situation | Why |
| --- | --- |
| **Preventing data loss** | Especially when running Grafana in a Docker container — containers can restart at any time |
| **Multiple instances** | Several Grafana instances need to share the same data (synchronized dashboards and data sources) |

**How to switch in `grafana.ini`:**

- Find the database section
- Change the type from `sqlite3` to `mysql` or `postgres`
- Enter the hostname, username, and password of your external database

```ini
[database]
type = postgres
host = <DB-HOST>:5432
user = <DB-USER>
password = <DB-PASSWORD>
```

### Saving and Applying Changes

1. Edit the file with a text editor such as `nano`:

```bash
sudo nano /etc/grafana/grafana.ini
```

2. Save with **Ctrl + O**, then exit with **Ctrl + X**
3. **Restart Grafana** so the new configuration is read:

**Stop the service**

```bash
sudo systemctl stop grafana-server
```

- *Verify:* `sudo systemctl status grafana-server` shows **`inactive`**

**Start the service**

```bash
sudo systemctl start grafana-server
```

- *Verify:* the status is back to **`active (running)`**

> **Tip:** Running `stop` and `start` as two separate steps lets you check the service status in between, which makes it easier to confirm the new configuration loaded cleanly.

---

## 🐳 Running the Stack with Docker Compose

> **Analogy:** Installing every component separately is like buying each ingredient and cooking every dish yourself. Docker Compose is a meal kit — one box, one command, and everything arrives ready to cook.

Besides installing Grafana by hand, you can run the whole stack with the course's companion GitHub repository ([aussiearef/grafana-udemy](https://github.com/aussiearef/grafana-udemy)). It contains all the configuration files and scripts you need, including a `docker` folder with a [`docker-compose.yaml`](https://github.com/aussiearef/grafana-udemy/blob/main/docker/docker-compose.yaml).

### Preparation

- Install and run **Docker Desktop** on your computer — download it from `docker.com`, then restart your computer
- **Download or clone** the repository, then open a terminal inside the `docker` folder
- Copy the `docker-compose.yaml` file into a directory where your user has **full control**

> **Note:** The stack creates several subdirectories on first run, so the directory must be writable by your user.

### Start the Stack

Launch every service at once:

```bash
docker compose up -d
```

> **Note:** Docker first downloads any images that aren't already on your machine.

### Stack Components

Docker Desktop shows several containers running together — no need to install each one separately:

| Component | Role |
| --- | --- |
| **Grafana** | Visualization UI (port `3000`) |
| **Prometheus** | Metrics (port `9090` for receiving metrics) |
| **Loki** | Logs |
| **Tempo** | Traces (OpenTelemetry) |
| **Grafana Alloy** | Telemetry collector — its config is downloaded via `wget` on first run |
| **Practice microservices** | Extra services for hands-on practice with Grafana Tempo |

- **Port mapping** between your machine and the containers is already configured

### Accessing Grafana

1. Open `http://localhost:3000` in a browser
2. Log in with the default credentials:

| Field | Value |
| --- | --- |
| **Username** | `admin` |
| **Password** | `admin` |

3. Reset the password when prompted on first login

### Checking the Data Sources

Open the **Data Sources** menu — three sources were added automatically: **Loki**, **Prometheus**, and **Tempo**.

- **Prometheus (Metrics)** — Go to **Explore**, select a metric (e.g. the sample shoe sales / shoe type data), and click **Run Query** to see metrics generated automatically for practice
- **Tempo (Traces)** — Collects **OpenTelemetry traces**. Trace data and the **service graph** are already prepared, so you can explore them without building anything first

### Modifying Component Configuration

On the first run, the local folder is automatically filled with a subdirectory and configuration files for each component.

**Example: customizing Grafana Alloy**

1. By default, the configuration file is downloaded via `wget` inside `docker-compose.yaml` when the container is first created
2. To use and modify your own local configuration:
   - Open your local `docker-compose.yaml`
   - Remove the duplicated `wget` commands that download the default templates from GitHub
   - Stop the running stack (this also removes the containers from Docker Desktop):

   ```bash
   docker compose down
   ```

   - Edit the configuration file in the component's folder
   - Start the stack again:

   ```bash
   docker compose up -d
   ```

> **⚠️ Important:** Every time you change a configuration file (`docker-compose.yaml` or a component's config), run `docker compose down` and then `docker compose up -d` so the containers pick up the change.

---

## 🎯 Quick Reference

| Concept | Problem Solved | Key Trade-off |
| --- | --- | --- |
| **Grafana Cloud** | Zero setup, always up to date, free plan for learning | Costly at scale, vendor lock-in, data on third-party infrastructure |
| **On-Premises** | Full control, compliance, unlimited customization | Needs a team; you handle scaling and updates |
| **`.deb` install (`dpkg`)** | Installs Grafana directly on an Ubuntu machine | Manual updates; you manage the server |
| **`grafana.ini`** | One place to change logs, database, and more | A forgotten `;` silently disables your change |
| **SQLite (default)** | No setup, enough for ~90% of cases | Data can be lost when a container restarts; can't be shared across instances |
| **MySQL / PostgreSQL** | Persistent, shared storage for Grafana data | An extra database to run and maintain |
| **Docker Compose stack** | Whole observability stack with one command | Every config change needs `down` then `up` |

---

## 💡 Best Practices

**✅ Do This**

- **Start with the Grafana Cloud free plan** — The fastest way to learn without any setup
- **Weigh team, customization, compliance, and budget** — Before choosing a production deployment
- **Run `apt-get update` first** — So dependencies install without problems
- **Match the package architecture** — `amd64` for Intel/AMD, `arm64` for ARM
- **Verify with `systemctl status`** — Look for `active (running)` after every start
- **Change the default password immediately** — `admin` / `admin` should never stay
- **Back up `grafana.ini` before editing** — Keep a copy such as `custom.ini`
- **Remove the `;` on every line you change** — Otherwise the setting stays commented out
- **Use an external database for Docker or multi-instance setups** — MySQL or PostgreSQL prevent data loss and keep instances in sync
- **Back up dashboards and data sources as JSON** — A simple safety net when staying on SQLite
- **Keep `docker-compose.yaml` in a directory you fully control** — The stack creates subdirectories there
- **Run `docker compose down` then `up -d` after every change** — So containers load the new configuration

**❌ Avoid This**

- **Changing a value but leaving the `;` in place** — The most common configuration mistake
- **Editing `grafana.ini` without a backup** — One typo and you have nothing to roll back to
- **Keeping the default `admin` password** — Public knowledge, easy to exploit
- **Forgetting to restart after a config change** — The old configuration keeps running
- **Using SQLite for containers that restart or for multiple instances** — Data gets lost or goes out of sync
- **Self-hosting without a team to maintain it** — Maintenance, scaling, and updates fall entirely on you

> Reference: [Grafana Cloud](https://grafana.com/products/cloud) · [Grafana Udemy course repository — docker-compose.yaml](https://github.com/aussiearef/grafana-udemy/blob/main/docker/docker-compose.yaml)

# 🔥 Prometheus

A practical reference guide for learning Prometheus from scratch — covering how it works, installation on Ubuntu, collecting metrics with exporters, the data model, metric types, and querying with PromQL.

---

## 📋 Table of Contents

- [Introduction](#-introduction)
  - [Who Is This For?](#who-is-this-for)
  - [What This Material Covers](#what-this-material-covers)
  - [Why Learn Prometheus?](#why-learn-prometheus)
- [What Is Prometheus?](#-what-is-prometheus)
  - [How Prometheus Works](#how-prometheus-works)
  - [Main Components](#main-components)
- [Installation on Ubuntu](#-installation-on-ubuntu)
  - [Preparation](#preparation)
  - [Create User & Directories](#create-user--directories)
  - [Download & Install the Binary](#download--install-the-binary)
  - [Set Ownership & Permissions](#set-ownership--permissions)
  - [Configure the Systemd Service](#configure-the-systemd-service)
  - [Start & Verify](#start--verify)
- [Collecting Metrics](#-collecting-metrics)
  - [Three Ways to Collect Metrics](#three-ways-to-collect-metrics)
  - [Client Library](#client-library)
  - [Exporter](#exporter)
  - [Pushgateway](#pushgateway)
- [Node Exporter](#️-node-exporter)
  - [What Is Node Exporter?](#what-is-node-exporter)
  - [Metrics Collected](#metrics-collected)
  - [Example: Monitoring an App Server](#example-monitoring-an-app-server)
- [Data Model](#-data-model)
  - [Time Series](#time-series)
  - [Metric Name & Labels](#metric-name--labels)
  - [Metric Types](#metric-types)
- [PromQL Data Types](#-promql-data-types)
  - [Scalar](#scalar)
  - [Instant Vector](#instant-vector)
  - [Range Vector](#range-vector)
- [Label Selectors](#-label-selectors)
  - [How Selectors Work](#how-selectors-work)
  - [Selector Operators](#selector-operators)
- [Operators](#-operators)
  - [Arithmetic Operators](#arithmetic-operators)
  - [Vector Matching](#vector-matching)
  - [Comparison Operators](#comparison-operators)
  - [Set Operators](#set-operators)
- [Aggregation](#-aggregation)
  - [Aggregation Operators](#aggregation-operators)
  - [Grouping with by and without](#grouping-with-by-and-without)
  - [Aggregation Over Time](#aggregation-over-time)
- [Time Offset](#-time-offset)
  - [What Is Offset?](#what-is-offset)
  - [Where to Place Offset](#where-to-place-offset)
- [Functions](#-functions)
  - [Rates for Counters](#rates-for-counters)
  - [Changes for Gauges](#changes-for-gauges)
  - [Data Availability](#data-availability)
  - [Math & Clamping](#math--clamping)
  - [Logarithms & Sorting](#logarithms--sorting)
  - [Time & Calendar](#time--calendar)
- [Quick Reference](#-quick-reference)
- [Best Practices](#-best-practices)

---

## 🎯 Introduction

### Who Is This For?

- **DevOps / SRE Engineer** — Sets up and operates metric collection for servers and services
- **Backend Programmer** — Exposes application metrics and queries them during incidents
- **System Administrator** — Monitors Linux servers with Node Exporter

### What This Material Covers

- How Prometheus collects and stores metrics
- Hands-on installation of Prometheus and Node Exporter on Ubuntu
- Querying metrics with PromQL
- Builds on the concepts in [001 — Foundations of Observability](001-foundations-of-observability.md)

### Why Learn Prometheus?

- **Industry standard** — The default metrics system in the cloud-native ecosystem (CNCF graduated)
- **Rich ecosystem** — Hundreds of exporters for databases, load balancers, and cloud services
- **Pairs with Grafana** — The most common stack for dashboards and alerting
- **Career growth** — Required knowledge for DevOps and SRE roles

---

## 🔥 What Is Prometheus?

> **Analogy:** A doctor doing rounds in a hospital. Instead of waiting for patients to call, the doctor visits every bed on a fixed schedule (every 15 seconds), reads each patient's monitor, and writes the readings into a medical chart with the exact time. Later, the doctor can flip through the chart to see how each patient has changed.

### How Prometheus Works

- An open-source **monitoring system and time series database (TSDB)**
- Uses the **scrape (pull) model** — Prometheus visits each target's `/metrics` endpoint on a schedule
- Default scrape interval is **every 15 seconds**
- Stores every value with a **timestamp**, then lets you query it with **PromQL**

> **Key Insight:** Prometheus never accepts pushed data directly. Everything — even data from short-lived jobs — is pulled. This is what makes a failed scrape a free "this service is down" signal.

### Main Components

| Component | Role |
| --- | --- |
| **Prometheus Server** | Scrapes targets, stores time series, answers PromQL queries |
| **Client Library** | Exposes metrics from inside your own application code |
| **Exporter** | Translates a third-party system's state into Prometheus format |
| **Pushgateway** | Holds metrics pushed by short-lived jobs so Prometheus can scrape them |
| **Alertmanager** | Routes alerts to email, Slack, etc. |

---

## 📦 Installation on Ubuntu

> **Analogy:** Setting up a new employee. Give them their own account with no keys to anything else (system user), a desk and a filing cabinet (directories), their tools (binary), and a work schedule so they show up every day — even after a holiday (systemd service).

### Preparation

Check the processor architecture so you download the right release file.

```bash
# x86_64 = amd64, aarch64 = arm64
uname -m

sudo apt update && sudo apt upgrade -y
sudo apt install wget -y
```

### Create User & Directories

Create a **service account without login access**, plus the data and config directories.

```bash
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus

sudo mkdir -p /var/lib/prometheus
sudo mkdir -p /etc/prometheus/rules
sudo mkdir -p /etc/prometheus/rules.d
sudo mkdir -p /etc/prometheus/files_sd
```

### Download & Install the Binary

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.16.0/prometheus-3.16.0.linux-amd64.tar.gz
tar xvf prometheus-3.16.0.linux-amd64.tar.gz
cd prometheus-3.16.0.linux-amd64

# Move binaries into the system PATH
sudo mv prometheus promtool /usr/local/bin/

# Copy the main config
sudo cp prometheus.yml /etc/prometheus/

# Verify
prometheus --version
```

### Set Ownership & Permissions

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chmod -R 775 /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

### Configure the Systemd Service

Run Prometheus in the **background** and start it automatically on **reboot**.

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

> **Note:** In Prometheus 3.x, the `--web.console.templates` and `--web.console.libraries` flags are gone — the web console assets are embedded in the binary.

### Start & Verify

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

Once the status is **`active (running)`**, open the web UI:

```text
http://<SERVER_IP>:9090
```

> **Gotcha:** If the page doesn't load, check that port **9090/TCP** is allowed in both UFW and your cloud provider's Security Group.

---

## 📥 Collecting Metrics

![Prometheus Data Collection](images/006.png)

> **Analogy:** Prometheus only speaks one language. If you built the car yourself, you install the sensors directly (client library). If you didn't, you hire an interpreter to sit next to it (exporter). And if the courier can't wait for you to come home, they leave the parcel in a drop box (Pushgateway).

### Three Ways to Collect Metrics

| Situation | Use |
| --- | --- |
| You have the application's source code | **Client Library** |
| Third-party system / server / database | **Exporter** |
| Short-lived job that dies right after finishing | **Pushgateway** |

### Client Library

- For applications **you build yourself** (Python, Go, Java, Ruby, .NET)
- Add the **official Prometheus library** to the code
- The library exposes a `/metrics` endpoint for Prometheus to scrape

### Exporter

- For **third-party systems** whose code you can't change:
  - Databases — **MySQL**, **PostgreSQL**
  - Load balancers — **HAProxy**
  - Cloud — **AWS CloudWatch**
  - Operating systems — **Linux**, **Windows Server**
  - **IoT** devices — traffic light sensors, temperature sensors
- A small program installed **next to the target** that reads its state and **translates it into Prometheus format**
- Prometheus scrapes the exporter **every 15 seconds** by default

> **Gotcha:** Don't write cron scripts that push data into Prometheus. It doesn't scale, and millions of IoT devices pushing at once can take Prometheus down. Use an exporter instead.

### Pushgateway

- For **short-lived jobs** (*batch jobs*, *serverless functions*) that die before Prometheus can scrape them
- Acts as a **temporary holding place**:
  1. The job **pushes** its metrics to Pushgateway before it exits
  2. Prometheus **scrapes** Pushgateway on its normal schedule

> **Note:** Pushgateway does **not** turn Prometheus into a push system. Prometheus still pulls — just from Pushgateway.

---

## 🖥️ Node Exporter

> **Analogy:** A medical monitor strapped to an athlete. It constantly reads heart rate, temperature, and blood pressure. Prometheus is the doctor who comes by every 15 seconds to read the monitor and decide whether the athlete is in shape or exhausted.

### What Is Node Exporter?

- **Not Node.js** — it has nothing to do with JavaScript
- **"Node"** in Prometheus means a machine running a **Unix-based kernel** (Linux, Ubuntu)
- The **official exporter** from the Prometheus team for **kernel-level system metrics**

### Metrics Collected

- **CPU Usage** — How busy the processor is
- **Memory Usage** — RAM usage
- **Disk Usage** — Storage capacity
- **Network I/O** — Network traffic in and out

Can be extended with **pluggable metric collectors**.

### Example: Monitoring an App Server

```text
Prometheus Server (10.30.0.119:9090)
  ↓ scrape every 15s
App Server + Node Exporter (10.30.0.120:9100)
```

**1. Install Node Exporter on the App Server (`10.30.0.120`)**

```bash
sudo groupadd --system node_exporter
sudo useradd -s /sbin/nologin --system -g node_exporter node_exporter

wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
tar xvf node_exporter-1.12.1.linux-amd64.tar.gz
cd node_exporter-1.12.1.linux-amd64

sudo mv node_exporter /usr/local/bin/
node_exporter --version
```

**2. Run it as a service bound to the private IP**

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=10.30.0.120:9100

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

**3. Allow only Prometheus to reach port 9100 (optional)**

```bash
sudo ufw allow from 10.30.0.119 to any port 9100 proto tcp
```

**4. Register the target on the Prometheus Server (`10.30.0.119`)**

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  # ... default prometheus job

  - job_name: 'app-server'
    static_configs:
      - targets: ['10.30.0.120:9100']
```

```bash
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

**5. Verify** — open `http://10.30.0.119:9090` → **Status → Targets** → `app-server` should be **UP**.

> **Tip:** Always run `promtool check config` before restarting. A YAML typo will stop Prometheus from starting.

---

## 🧱 Data Model

> **Analogy:** A hospital chart. Each line is one measurement for one patient (a time series): *"Bed 12, heart rate, 08:00:15 → 72"*. The patient's bed and ward are the labels; "heart rate" is the metric name; the reading and time are the sample.

### Time Series

- Every value is stored together with a **timestamp** (Unix time)
- A series of values for the same thing over time is a **time series**

### Metric Name & Labels

- Each time series is uniquely identified by its **metric name + set of labels**
- **Labels** are optional *key-value* pairs inside curly braces, separated by commas

```promql
<metric_name>{<label_key>="<label_value>", ...}

http_requests_total{method="GET", code="200"}
```

### Metric Types

| Type | Behavior | Example |
| --- | --- | --- |
| **Counter** | Only goes **up** (resets to 0 on restart) | Total requests, total errors |
| **Gauge** | Goes **up and down** | Temperature, memory usage, queue length |
| **Histogram** | Counts values into **buckets** | Request duration distribution |
| **Summary** | Calculates **quantiles** on the client side | p95 / p99 latency |

> **Key Insight:** The metric type decides which functions make sense. Use `rate()` on counters and `delta()` on gauges — mixing them up gives meaningless numbers.

---

## 🔢 PromQL Data Types

> **Analogy:** A single number on a sticky note (scalar). A photo of every patient monitor taken right now (instant vector). A video clip of every monitor over the last 5 minutes (range vector).

### Scalar

- A **single value** with no time series
- **Float** — Both `1` and `1.5` are stored as floats
- **String** — Text in quotes (`"..."` or `'...'`)

### Instant Vector

- A set of time series, each with **one latest sample**

```promql
node_network_transmit_errs_total
node_network_transmit_errs_total{device="eth0"}
```

### Range Vector

- A set of time series, each with **a range of samples** going back in time
- Marked with square brackets `[...]`

```promql
node_network_transmit_errs_total[5m]
```

With the default 15-second scrape interval, `[5m]` returns about 20 samples per series.

**Time units** (*case-sensitive*):

| Unit | Meaning |
| --- | --- |
| `ms` | Milliseconds |
| `s` | Seconds |
| `m` | Minutes |
| `h` | Hours |
| `d` | Days (24 hours) |
| `w` | Weeks (7 days) |
| `y` | Years (365 days) |

> **Gotcha:** There is no unit for months.

---

## 🏷️ Label Selectors

> **Analogy:** Searching a hospital's patient list. "Ward = ICU" narrows it down; "Ward = ICU, Doctor = Smith" narrows it further. Every extra filter must be true at the same time.

### How Selectors Work

- A query starts with the **metric name**, then narrows it with filters inside `{}`
- **Comma (`,`) means AND** — every filter must match

### Selector Operators

| Operator | Meaning |
| --- | --- |
| `=` | Exactly equal |
| `!=` | Not equal |
| `=~` | Matches regex |
| `!~` | Does not match regex |

```promql
prometheus_http_requests_total{code=~"2.*", handler!="/metrics"}
```

> **Gotcha:** Label values are always **strings**. `code="200"` works; `code=200` without quotes is an error, and there is no automatic conversion.

> **Tip:** Avoid regexes that can match an empty string — `code=~".*"` also matches series that don't have the `code` label at all.

---

## ➕ Operators

> **Analogy:** Combining two class attendance sheets. You can only add Budi's scores together if Budi appears on both sheets. If a name is on only one sheet, there's nothing to pair it with, so it's left out.

### Arithmetic Operators

| Operator | Meaning |
| --- | --- |
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Division |
| `%` | Modulo |
| `^` | Power |

| Combination | Behavior | Example |
| --- | --- | --- |
| Scalar ⊕ Scalar | Returns a scalar | `2 + 2` → `4` |
| Vector ⊕ Scalar | Applied to **every** element | `550`, `333` minus `10` → `540`, `323` |
| Vector ⊕ Vector | Applied to **matching** pairs only | See below |

Operators always return **new values** — the stored data is never changed.

### Vector Matching

- Elements are paired by **exactly the same label set**
- The **metric name is ignored** when matching, and the result has no metric name
- Elements **without a pair are dropped**

**Example:** `api_errors_total + api_retries_total`

| Labels | `api_errors_total` | `api_retries_total` | Result |
| --- | --- | --- | --- |
| `{endpoint="/login"}` | 10 | 3 | **13** ✅ |
| `{endpoint="/register"}` | 5 | 1 | **6** ✅ |
| `{endpoint="/checkout"}` | 2 | — | **dropped** ❌ |

### Comparison Operators

| Operator | Meaning |
| --- | --- |
| `==` | Equal |
| `!=` | Not equal |
| `>` / `<` | Greater / less than |
| `>=` / `<=` | Greater / less than or equal |

- **Booleans are numbers** — `1` is true, `0` is false
- **Vector vs scalar** — Works as a **filter**: only elements meeting the condition are returned
- **Vector vs vector** — Matches pairs by labels first, then filters
- **`bool` modifier** — Returns `0`/`1` instead of filtering; required for scalar vs scalar

```promql
node_load1 > 2          # filter: only series above 2
node_load1 > bool 2     # every series, value 1 or 0
5 == bool 5             # 1
```

### Set Operators

| Operator | Name | Result |
| --- | --- | --- |
| `and` | Intersection | Left elements that **have a match** on the right |
| `or` | Union | All left elements, plus right elements with no match on the left |
| `unless` | Difference | Left elements that **have no match** on the right |

- Work on **instant vectors only**
- Matching is by **label set**, not by value
- Must be written in **lowercase**

---

## 📊 Aggregation

> **Analogy:** A school principal doesn't read every student's score. They ask for the average per class, the top 3 students, or the total number of students. Aggregation turns many rows into a few meaningful numbers.

### Aggregation Operators

| Operator | Result |
| --- | --- |
| `sum` | Total |
| `avg` | Average |
| `min` / `max` | Smallest / largest value |
| `count` | Number of elements |
| `count_values` | Number of elements per distinct value |
| `group` | Groups elements; value is always `1` |
| `topk(n, v)` | The *n* largest elements |
| `bottomk(n, v)` | The *n* smallest elements |
| `stddev` / `stdvar` | Population standard deviation / variance |

### Grouping with by and without

- **`by (label)`** — Keep **only** the labels you name

  ```promql
  sum(node_cpu_seconds_total) by (mode)
  ```

- **`without (label)`** — Drop the labels you name, keep the rest

  ```promql
  sum(node_cpu_seconds_total) without (mode)
  ```

### Aggregation Over Time

- Regular aggregation works on **instant vectors** only — `sum(metric[2h])` is an error
- For range vectors, use the **`_over_time`** functions, which summarize each series **over its own history**

| Function | Result over the range |
| --- | --- |
| `avg_over_time` | Average |
| `sum_over_time` | Total |
| `min_over_time` / `max_over_time` | Smallest / largest value |
| `count_over_time` | Number of samples |
| `stddev_over_time` / `stdvar_over_time` | Standard deviation / variance |

```promql
avg_over_time(node_load1[1h])
```

> **Note:** `sum(...)` collapses many series into one at a single moment. `sum_over_time(...)` keeps every series but collapses its history.

---

## ⏪ Time Offset

> **Analogy:** Instead of looking at the patient monitor now, you flip the chart back to the reading from 8 hours ago.

### What Is Offset?

- By default, a query returns the **latest** value
- **`offset`** shifts the query back in time using the usual duration units

```promql
prometheus_http_requests_total offset 10m
```

### Where to Place Offset

`offset` must come **right after the metric selector**, not at the end of the query.

```promql
# ✅ Correct
avg(prometheus_http_requests_total offset 8h) by (code)

# ❌ Wrong
avg(prometheus_http_requests_total) by (code) offset 8h
```

---

## 🧮 Functions

> **Analogy:** A calculator attached to the medical chart. It can tell you how fast a value is rising, whether a reading is missing, or round and sort the numbers before you show them on a dashboard.

### Rates for Counters

Counters only go up, so their raw value is rarely useful. Ask **how fast** they grow instead.

| Function | Result |
| --- | --- |
| `rate(v[d])` | Average per-second increase over the range |
| `irate(v[d])` | Per-second increase from the **last two samples** |
| `increase(v[d])` | Total increase over the range |

```promql
rate(prometheus_http_requests_total[5m])
```

> **Tip:** These functions handle counter resets (e.g. after a restart) automatically. Always use them on counters instead of `delta()`.

### Changes for Gauges

| Function | Result |
| --- | --- |
| `delta(v[d])` | Difference between the first and last value in the range |
| `idelta(v[d])` | Difference between the **last two samples** |

```promql
delta(node_hwmon_temp_celsius[2h])
```

> **Gotcha:** Only use `delta` and `idelta` on **gauges**, never on counters.

### Data Availability

| Function | Input | Result |
| --- | --- | --- |
| `absent(v)` | Instant vector | Data exists → **empty**. No data → one element with value **`1`** |
| `absent_over_time(v[d])` | Range vector | Same as `absent`, over a time range |

Useful for alerting when a metric **disappears**.

### Math & Clamping

| Function | Result | Example |
| --- | --- | --- |
| `abs()` | Absolute value | `-5` → `5` |
| `ceil()` | Round up | `1.6` → `2` |
| `floor()` | Round down | `1.6` → `1` |
| `clamp_min(v, min)` | Raise values below `min` to `min` | — |
| `clamp_max(v, max)` | Lower values above `max` to `max` | — |
| `clamp(v, min, max)` | Both bounds at once | — |

Clamping is handy for keeping dashboard graphs (e.g. Grafana) readable.

### Logarithms & Sorting

| Function | Result | Example |
| --- | --- | --- |
| `log2(v)` | Base-2 logarithm | `2` → `1` |
| `log10(v)` | Base-10 logarithm | `10` → `1` |
| `ln(v)` | Natural logarithm | `e` → `1` |
| `sort(v)` | Ascending order | — |
| `sort_desc(v)` | Descending order | — |

### Time & Calendar

| Function | Result |
| --- | --- |
| `time()` | Current Unix timestamp in seconds (scalar) |
| `timestamp(v)` | When each sample was scraped; shifts back too when used with `offset` |
| `day_of_month()` | `1`–`31` |
| `day_of_week()` | `0`–`6`, where **`0` = Sunday** and `6` = Saturday |

---

## 🎯 Quick Reference

| Concept | Problem Solved | Key Trade-off |
| --- | --- | --- |
| **Scrape (pull) model** | Detects dead targets immediately | Can't reach short-lived jobs directly |
| **Client Library** | Exposes metrics from your own code | Requires changing the application |
| **Exporter** | Monitors systems you can't modify | One more process to run per target |
| **Pushgateway** | Captures metrics from short-lived jobs | Becomes a single point of failure; stale data stays until deleted |
| **Node Exporter** | Linux server health out of the box | Linux/Unix only |
| **Labels** | Slice one metric by many dimensions | Too many label values blow up storage (high cardinality) |
| **Counter** | Tracks totals that only grow | Raw value is meaningless without `rate()` |
| **Gauge** | Tracks values that go up and down | Can miss spikes between scrapes |
| **Instant Vector** | Latest value of every series | No history |
| **Range Vector** | History of every series | Must be passed through a function before graphing |
| **Aggregation** | Summarizes many series into a few | Loses per-series detail |
| **`_over_time` functions** | Summarizes each series' history | Only works on range vectors |
| **`offset`** | Compares now against the past | Must sit right after the selector |

---

## 💡 Best Practices

**✅ Do This**

- **Run Prometheus and exporters as dedicated system users** — never as root
- **Validate config with `promtool check config`** before every restart
- **Bind exporters to a private IP and firewall them** — only Prometheus should reach `/metrics`
- **Use `rate()` or `increase()` on counters** — never graph a raw counter
- **Keep label values bounded** — methods, status codes, and endpoints, not user IDs or request IDs
- **Use Pushgateway only for batch jobs** — long-running services should be scraped directly
- **Alert on `absent()`** for critical metrics so you know when a target stops reporting

**❌ Avoid This**

- **Exposing exporters to the public internet** — metrics leak details about your infrastructure
- **Pushing data with cron scripts** — use an exporter instead
- **Using `delta()` on counters** — it breaks on counter resets; use `rate()` or `increase()`
- **Putting `offset` at the end of a query** — it belongs right after the selector
- **Writing label values without quotes** — `code=200` is an error; use `code="200"`

> Reference: [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/) · [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/) · [Node Exporter](https://github.com/prometheus/node_exporter)

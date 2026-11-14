# Study Observability 🔭

This repository contains a reference guide for observability — covering monitoring vs observability, telemetry data (MELT), monitoring methods, metric collection, and hands-on Prometheus with Node Exporter and PromQL.

## Installation 🔧

1. **Clone the Repository**:

   ```bash
   git clone https://github.com/dzarurizkyy/study-observability.git
   cd study-observability
   ```

2. **Open the Guide**:
   - Read the material on GitHub, or open it in any Markdown viewer
   - Diagrams are stored in the `images/` folder and referenced from the guides

<br />

   > **Note:** The theory guide needs no tools. The Prometheus guide needs an Ubuntu server (or VM) to follow the installation steps.

## List of Material 📚

- 🔭 **[Foundations of Observability](001-foundations-of-observability.md)**

  Monolith vs microservices, monitoring and its 3 key questions, observability, MELT telemetry, RED / USE / Four Golden Signals / Core Web Vitals, push vs scrape, and MTTD / MTTR:

  ```text
  UI Layer              → Core Web Vitals
  Service Layer         → RED Method, Four Golden Signals
  Infrastructure Layer  → USE Method, Four Golden Signals
  ```

- 🔥 **[Prometheus](002-prometheus.md)**

  How Prometheus works, installation on Ubuntu, client libraries, exporters, Pushgateway, Node Exporter, the data model, metric types, and PromQL:

  ```text
  Prometheus Server (:9090)
    ↓ scrape every 15s
  Node Exporter (:9100) on the App Server
  ```

  ```promql
  # Requests per second over the last 5 minutes, grouped by status code
  sum(rate(prometheus_http_requests_total[5m])) by (code)
  ```

## 📍 References

- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

## 👨‍💻 Contributors

- [Dzaru Rizky Fathan Fortuna](https://www.linkedin.com/in/dzarurizky)

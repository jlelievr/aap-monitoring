# aap-monitoring
AAP 2.6 Monitoring Stack with Prometheus and Grafana

# AAP 2.6 Monitoring Stack (Podman + Systemd)

A lightweight, daemonless monitoring solution for **Ansible Automation Platform 2.6** using **Podman**, **Prometheus**, **Grafana**, and **Postgres Exporter**.

This project uses `podman play kube` to deploy the entire stack from a single YAML definition ("Infrastructure as Code") and manages the lifecycle via `systemd`.

## 🏗 Architecture
* **Podman Pod:** `monitoring-stack` (Shared network namespace)
* **Prometheus** (Port 9090): Scrapes AAP Controller & Postgres Exporter.
* **Grafana** (Port 3000): Visualization Dashboard.
* **Postgres Exporter**: Bridges internal DB stats to Prometheus.

---

## 📋 Prerequisites
* **OS:** RHEL 8/9, CentOS Stream, or Fedora.
* **Tools:** `podman` installed (`sudo dnf install podman`).
* **Network:** Ports **3000** and **9090** must be open.

---

## ⚙️ 1. AAP & Database Preparation (Required)

### A. Configure AAP Controller
1.  Log in to **Ansible Automation Controller** as Admin.
2.  Navigate to **Access > Users** -> **Add**. Create a user named `prometheus`.
3.  Go to **Roles** -> **Add**. Assign the **System Auditor** role (read-only access).
4.  Log in as `prometheus`. Go to **User Details > Tokens** -> **Add**.
    * Scope: **Read**.
    * **Save and Copy the token**.

### B. Configure PostgreSQL
1.  SSH into your AAP Database node.
2.  Connect to Postgres:
    ```bash
    sudo su - postgres
    psql
    ```
3.  Run these SQL commands:
    ```sql
    CREATE USER postgres_exporter WITH PASSWORD 'CHANGE_THIS_SECURE_PASSWORD';
    ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;
    GRANT pg_monitor TO postgres_exporter;
    ```

## 📝 2. Configuration

Before deploying, update the configuration files included in this repository to match your environment.

### A. Update Prometheus Config
Open `prometheus/prometheus.yml` and update the following fields:
* `targets`: Change to your AAP Controller FQDN (e.g., `ansible.example.com`).
* `bearer_token`: Paste the AAP Token generated in Step 1.

### B. Update Pod Definition
Open `monitoring-pod.yaml` and find the `postgres-exporter` container section. Update the `DATA_SOURCE_NAME` environment variable:

```yaml
# Inside monitoring-pod.yaml
- name: DATA_SOURCE_NAME
  value: "postgresql://postgres_exporter:<YOUR_DB_PASSWORD>@<YOUR_DB_HOST>:5432/postgres?sslmode=disable"
---

## 📊 4. Grafana Configuration

Once the stack is running, configure the dashboard.

1.  **Login:**
    * Go to `http://<YOUR_IP>:3000`
    * User: `admin` / Password: `admin` (Change on first login).

2.  **Connect Data Source:**
    * Navigate to **Connections > Data Sources > Add data source**.
    * Select **Prometheus**.
    * **Prometheus server URL:** `http://localhost:9090`
        * *(Note: Use `localhost` because the containers share the Pod network).*
    * Click **Save & test**.

3.  **Import Dashboards:**
    * Navigate to **Dashboards > New > Import**.
    * **AAP Controller Metrics:** Enter ID `12609` and click Load. Select your Prometheus source and Import.
    * **PostgreSQL Metrics:** Enter ID `9628` and click Load. Select your Prometheus source and Import.

---

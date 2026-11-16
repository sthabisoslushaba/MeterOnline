# Meter Portal – your own PnPscada-style metering website on AWS

```
Meter ──RS485/RS232── Riecktron 4G modem ──(4G, TCP client mode)──►  AWS EC2 (Cape Town)
                                                                     ├─ collector  (talks to meters, ports 5001+)
                                                                     ├─ web/API    (this portal, FastAPI)
                                                                     ├─ PostgreSQL (readings)
                                                                     └─ Caddy      (HTTPS for your domain)
```

## What's included

| Area | Features |
|---|---|
| Dashboard | Energy today/month, meters online, open alarms, 30-day daily energy chart, meter table |
| Meter page | Live V/I/kW/kVA/PF, 30-min load profile (kVA or kW) with NMD line, TOU energy (daily/monthly), monthly bill estimate, CSV export |
| Alarms | Comms loss, NMD exceeded, low power factor – acknowledge / history |
| Admin | Sites, meters (serial, protocol, CT ratio, NMD, collector port, tariff), users with per-site access, TOU tariffs (JSON editor), collector API docs |
| Security | Login with hashed passwords, HTTP-only session cookie, admin / viewer roles, viewers only see their sites, API key for the collector |
| Collector | Listens for Riecktron modems (one TCP port per modem), Modbus RTU driver, 30-min kWh/kvarh from register deltas, pushes to `/api/ingest` |

## 1. Try it on your PC (5 minutes)

```bash
pip install -r requirements.txt
python -m app.demo                 # creates the database, admin user and 60 days of demo data
uvicorn app.main:app --reload      # open http://localhost:8000
```
Login: `admin@example.com` / `ChangeMe123!` (change this under "Change password").

## 2. Deploy on AWS

**Automated pipeline (recommended):** GitHub Actions tests every push and deploys `main` to EC2 through AWS SSM.
Full setup: [`deploy/PIPELINE.md`](deploy/PIPELINE.md). The manual steps below are what the pipeline automates.


1. **Launch EC2** in region **af-south-1 (Cape Town)**: Ubuntu 24.04, `t3.small` is enough to start (2 GB RAM), 30 GB gp3 disk.
2. **Elastic IP**: allocate one and attach it to the instance. This is the IP you send to the modems.
3. **Security group** inbound rules:
   - 22 (SSH) – your office IP only
   - 80, 443 – anywhere (website)
   - 5001-5100 TCP – anywhere (Riecktron modems; you can't lock this to an IP because mobile IPs change)
4. **DNS**: create an A record, e.g. `meters.sinanienergy.com`, pointing to the Elastic IP.
5. **Install and start**:
   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose-v2
   git clone <your repo> meterportal && cd meterportal     # or copy the zip with scp
   cp .env.example .env && nano .env                         # set DOMAIN and all passwords/keys
   sudo docker compose up -d --build
   ```
   Caddy gets the HTTPS certificate automatically. Open `https://<your domain>` and log in with ADMIN_EMAIL / ADMIN_PASSWORD.
6. **Backups**: turn on daily EBS snapshots (AWS Data Lifecycle Manager), or run a nightly
   `docker compose exec db pg_dump -U meterportal meterportal | gzip > backup.sql.gz` and copy it to S3.

## 3. Connect the meters

1. In **Admin → Meters**, add each meter with its **serial** and a **collector TCP port** (5001, 5002 …, one per modem).
2. Fill in `collector/meters.json` (copy `meters.example.json`) with the same serial, port, CT ratio and your meter's **Modbus register map** (from the meter manual).
3. Start the collector: `sudo docker compose --profile collector up -d --build`
4. Send each Riecktron modem its setup SMS (RS485 example):
   ```
   APN:internet
   IP2:<Elastic IP or domain>
   Port2:5001
   Mode2:client
   INT2:30
   IDX2:2
   BAUD2:9600
   Reset
   ```
5. Within a minute the meter shows **Online**. Load-profile bars build up every 30 minutes.

**DLMS meters (Kamstrup, Hexing, Landis+Gyr, Elster):** the included driver speaks Modbus RTU. For DLMS,
replace `read_meter()` in `collector/collector.py` with a DLMS client (e.g. the `gurux-dlms` Python library),
reading the meter's own 30-minute load profile and sending it in the same `/api/ingest` format.

## 4. Collector → portal API

```
POST /api/ingest          Header: X-API-Key: <INGEST_API_KEY>
{ "serial": "KAM-001234",
  "intervals": [{"ts": "2026-11-16T10:30:00+02:00", "kwh": 12.4, "kvarh": 3.1, "kwh_export": 0}],
  "instant":   {"v1": 231.2, "i1": 41.2, "kw": 27.8, "kva": 28.5, "pf": 0.976, "hz": 50.01} }
```
`ts` is the **end** of the 30-minute interval. Re-sending an interval overwrites it, so backfilling is safe.

## 5. Tariffs

**Admin → Tariffs** ships with an *example* TOU tariff in the Eskom-style layout (peak / standard / off-peak,
high season Jun–Aug). **The periods and rates are placeholders.** Enter the current published figures for
each tariff you bill on (Megaflex, Miniflex, municipal, etc.) and assign it to each meter.

## Project layout
```
app/        FastAPI backend (models, auth, TOU engine, alarms, API routes, demo data)
web/        Website (HTML/CSS/JS + Chart.js, no build step)
collector/  Riecktron/Modbus collector + example register map
```

## Next steps
- Email/SMS alarm notifications: hook into `raise_alarm()` in `app/alarms.py` (AWS SNS or SES).
- PDF invoices, tenant portal branding, solar export reporting, public-holiday calendar for TOU.
- Many meters (500+): move to a larger instance or RDS PostgreSQL and run several collectors.
# MeterOnline

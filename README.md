# SIEM Deployment: Graylog 7 + OpenSearch + MongoDB

## Overview

This document summarizes a single-node Graylog SIEM deployment used to aggregate, search, and alert on security and operational events across a multi-node home lab. It is written for a portfolio audience, focusing on architecture, component integration, operational quirks, and the reasoning behind specific deployment choices.

The goal of this system is to centralize log visibility: security events from vulnerability scanners, network device telemetry, system-level syslog, and structured data from automation pipelines all land in one searchable platform with alerting capability.

## Environment

| Field | Value |
|-------|-------|
| System | Dedicated SIEM VM |
| IP Address | (internal) |
| OS | Ubuntu Server 24.04 LTS |
| Components | Graylog 7, OpenSearch 2.x, MongoDB 8.0 |
| vCPU / RAM / Disk | 2–4 / 4–8 GB / 40–80 GB |
| Key Ports | 9000 (Graylog Web/API), 9200 (OpenSearch), 27017 (MongoDB) |
| Start Order | MongoDB → OpenSearch → Graylog |

## Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│  SIEM VM (Ubuntu 24.04)                                          │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────────┐  │
│  │ Graylog 7│───▶│ OpenSearch 2 │    │    MongoDB 8.0        │  │
│  │  :9000   │    │   :9200      │    │     :27017            │  │
│  │ Web + API│    │  single-node │    │  Graylog metadata     │  │
│  └──────────┘    └──────────────┘    └───────────────────────┘  │
│       │                                                          │
│       │ Inputs: Syslog UDP, Raw TCP, Beats, HTTP                 │
└───────┼──────────────────────────────────────────────────────────┘
        │
        │  Receives logs from:
        ▼
┌────────────────────────────────────────────────────────────────┐
│  Log Sources                                                    │
│  • Cribl Stream (forwarded JSON — vuln data, enriched events)   │
│  • rsyslog (Linux VMs and containers)                           │
│  • Proxmox hosts (syslog)                                       │
│  • Network devices (Omada controller, switches)                 │
│  • Automation pipelines (OpenVAS export, custom scripts)        │
└────────────────────────────────────────────────────────────────┘
```

Graylog owns search, alerting, dashboards, and stream routing. OpenSearch provides the indexing and storage backend. MongoDB stores Graylog's configuration metadata (users, streams, pipeline rules, alert definitions). All three run as native packages on one VM — no containers.

## Input Architecture

| Input | Port/Protocol | Source | Purpose |
|-------|---------------|--------|---------|
| Syslog UDP | 1514/UDP | Linux hosts, Proxmox | General system logs |
| Raw TCP JSON | 20025/TCP | Cribl Stream | Structured security events (OpenVAS, enriched) |
| Beats | 5044/TCP | Filebeat agents (if deployed) | Application logs |

Inputs are configured through the Graylog web UI and persisted in MongoDB. Each input type maps to a stream for routing, retention, and access control.

## Design Decisions

- **Single-node OpenSearch with security disabled:** This is a home lab, not production. Removing TLS and authentication from OpenSearch simplifies troubleshooting and reduces startup failure modes. The tradeoff is accepted because the VM sits on a private network segment.
- **MongoDB 8 with AVX requirement:** MongoDB 7/8 requires AVX CPU instructions. The VM uses `cpu: host` in Proxmox to pass through the host CPU flags. This sacrifices cross-node live migration but is mandatory for the current MongoDB version.
- **Native packages over containers:** Running Graylog, OpenSearch, and MongoDB as systemd services makes log inspection, configuration, and recovery more transparent than nested container debugging. For a single-node deployment, the operational simplicity outweighs container portability.
- **Start order dependency:** Services must start in order: MongoDB → OpenSearch → Graylog. If Graylog starts before its backends are ready, login and indexing can break silently.
- **JVM heap sizing:** OpenSearch heap (`-Xms`/`-Xmx`) set to roughly half of system RAM. Exceeding 50% starves the OS page cache and degrades search performance.

## Pipeline Processing

Graylog pipelines parse and normalize incoming data:

1. **JSON flattening** — Raw TCP JSON input from Cribl is parsed and nested fields are extracted into top-level searchable fields (e.g., `openvas_severity`, `openvas_nvt_name`).
2. **Stream routing** — Events are routed to streams by source type (security, infrastructure, network) for separate retention and dashboard views.
3. **Field type enforcement** — Numeric fields like severity scores are validated at ingest to enable range queries and histogram dashboards.

## Retention and Index Management

Index sets are configured per-stream with rotation based on size and time:

| Stream | Rotation | Retention | Notes |
|--------|----------|-----------|-------|
| Security events | Daily or 1 GB | 90 days | OpenVAS, alerts |
| Infrastructure | Daily or 2 GB | 30 days | Syslog, system events |
| Default | Daily or 1 GB | 14 days | Catch-all |

## Quirks and Gotchas

- **AVX requirement:** MongoDB 7/8 absolutely requires AVX. Using `cpu: host` in Proxmox is mandatory for this stack.
- **OpenSearch demo config breaks install:** The `install_demo_configuration.sh` script in the postinst can fail and leave the package in a broken `dpkg` state. Commenting it out and running `dpkg --configure -a` is the fix.
- **Config key mismatch:** Graylog 7 still uses `elasticsearch_hosts` in `server.conf` even though it connects to OpenSearch. Don't look for an `opensearch_hosts` key — it doesn't exist.
- **Start order dependency:** MongoDB → OpenSearch → Graylog. If Graylog starts before backends are ready, login can break.
- **JVM heap sizing:** Don't exceed 50% of system RAM for OpenSearch heap.
- **`vm.max_map_count`:** OpenSearch requires `vm.max_map_count=262144`. Without it, OpenSearch will refuse to start.

## What This Demonstrates

This deployment shows more than installing a log aggregator. It demonstrates SIEM architecture decisions in a realistic environment: component selection, dependency ordering, input design, pipeline processing, retention planning, and integration with upstream security tooling (vulnerability scanners, log routers, automation pipelines).

The system serves as the central visibility layer for the entire lab — if something generates a log or a finding, it ends up here in a searchable, alertable form.

---

Sanitized for public portfolio use.

For step-by-step deployment instructions, see [OPERATIONS.md](./OPERATIONS.md).

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PromSNMP is a multi-module project that bridges SNMP network device monitoring with Prometheus metrics collection. It consists of three modules:

- **promsnmp-metrics**: Spring Boot 3.5 application that discovers SNMP agents, collects metrics via SNMP OID walks, and exposes Prometheus-compatible endpoints (`/snmp`, `/metrics`, `/targets`)
- **promsnmp-common**: Shared DTOs, models (Agent, UserAgent, CommunityAgent, NetworkDevice), and utilities
- **promsnmp-controller**: Console and inventory service for PromSNMP Metrics Collectors (proprietary)

## Build Commands

```bash
# Build promsnmp-common first (required dependency)
cd promsnmp-common && mvn install

# Build promsnmp-metrics (from promsnmp-metrics directory)
make                        # Compile and assemble JAR (skips tests)
mvn install                 # Build with tests
mvn test -Dtest=MyTestClass # Run a single test class
mvn test -Dtest=MyTestClass#testMethodName  # Run a single test method
make clean                  # Clean build artifacts
make oci                    # Build Docker container image

# Run locally
java -jar target/metrics-*.jar
```

## Release Process

```bash
make release RELEASE_VERSION=x.y.z  # Creates release in local repo
git push                             # Push snapshot version
git push origin vx.y.z              # Push tag to trigger artifact build
```

## Environment Variables

Key configuration switches in `ServiceApiConfig`:
- `PROM_METRICS_API`: snmp/demo/direct - selects metrics service backend
- `PROM_DISCOVERY_API`: snmp/demo - selects discovery service backend
- `METRICS_REPO_API`: snmp/demo/direct - selects repository backend

See `deployment/docker-compose.yml` for full list including cache, discovery scheduling, and site configuration.

## Architecture

### promsnmp-metrics Layer Structure

**Controllers** (`controllers/`)
- `MetricsController`: Primary Prometheus scrape endpoint (`/snmp?target=...`, `/metrics`)
- `DiscoveryController`: REST API for SNMP discovery operations
- `PromSnmpController`: Utility endpoints (`/promSnmp/hello`, `/promSnmp/evictCache`, `/promSnmp/threadPools`)

**Services** (`services/`)
- Strategy pattern implementations switchable via environment variables
- `PrometheusMetricsService` interface with SNMP-based, resource-based, and direct implementations
- `CachedMetricsService`: Caffeine cache wrapper (configurable TTL)

**Repositories** (`repositories/`)
- `SnmpMetricsRepository`: Live SNMP queries, computes interface utilization histograms
- JPA repositories (`jpa/`): NetworkDevice, Agent, DiscoverySeed persistence

**SNMP Integration** (`snmp/`, `utils/`)
- Uses SNMP4J 3.7.1 library
- `SnmpAgentConfig`: Connection configuration
- Protocol mappers for SNMPv3 auth (MD5, SHA) and privacy (DES, 3DES, AES)

### Data Flow: Metrics Scrape

1. Prometheus calls `/snmp?target=device.example.com`
2. Controller → CachedMetricsService → SnmpMetricsRepository
3. SNMP OID walk collects: sysUpTime, ifName, ifDescr, ifHCInOctets, ifHCOutOctets, packet counts
4. Computes interface utilization histograms
5. Returns OpenMetrics text format (RFC 9154)

## Tech Stack

- Java 21, Spring Boot 3.5, Spring Data JPA
- SNMP4J 3.7.1 for SNMP operations
- Caffeine for caching, Micrometer/Prometheus for metrics
- H2 in-memory database, Lombok, SpringDoc OpenAPI
- Spring Shell 3.4 for CLI commands

## Development Stack

```bash
cd promsnmp-metrics/deployment && docker compose up -d
```
- Grafana: http://localhost:3000 (admin/admin)
- Prometheus: http://localhost:9090
- PromSNMP: http://localhost:8080
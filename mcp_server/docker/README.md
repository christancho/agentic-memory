# Docker Deployment for Graphiti MCP Server

This directory contains Docker Compose configurations for running the Graphiti MCP server with graph database backends: FalkorDB (combined image) and Neo4j.

## Quick Start

```bash
# Default configuration (FalkorDB combined image)
docker-compose up

# Neo4j (separate containers)
docker-compose -f docker-compose-neo4j.yml up
```

## Environment Variables

Create a `.env` file in this directory with your API keys:

```bash
# Required
OPENAI_API_KEY=your-api-key-here

# Optional
GRAPHITI_GROUP_ID=main
SEMAPHORE_LIMIT=10

# Database-specific variables (see database sections below)
```

## Database Configurations

### FalkorDB (Combined Image)

**File:** `docker-compose.yml` (default)

The default configuration uses a combined Docker image that bundles both FalkorDB and the MCP server together for simplified deployment.

#### Configuration

```bash
# Environment variables
FALKORDB_URI=redis://localhost:6379  # Connection URI (services run in same container)
FALKORDB_PASSWORD=  # Password (default: empty)
FALKORDB_DATABASE=default_db  # Database name (default: default_db)
```

#### Accessing Services

- **FalkorDB (Redis):** redis://localhost:6379
- **FalkorDB Web UI:** http://localhost:3000
- **MCP Server:** http://localhost:8000

#### Data Management

**Backup:**
```bash
docker run --rm -v mcp_server_falkordb_data:/var/lib/falkordb/data -v $(pwd):/backup alpine \
  tar czf /backup/falkordb-backup.tar.gz -C /var/lib/falkordb/data .
```

**Restore:**
```bash
docker run --rm -v mcp_server_falkordb_data:/var/lib/falkordb/data -v $(pwd):/backup alpine \
  tar xzf /backup/falkordb-backup.tar.gz -C /var/lib/falkordb/data
```

**Clear Data:**
```bash
docker-compose down
docker volume rm mcp_server_falkordb_data
docker-compose up
```

#### Gotchas
- Both FalkorDB and MCP server run in the same container
- FalkorDB uses Redis persistence mechanisms (AOF/RDB)
- Default configuration has no password - add one for production
- Health check only monitors FalkorDB; MCP server startup visible in logs

See [README-falkordb-combined.md](README-falkordb-combined.md) for detailed information about the combined image.

### Neo4j

**File:** `docker-compose-neo4j.yml`

Neo4j runs as a separate container service with its own web interface.

#### Configuration

```bash
# Environment variables
NEO4J_URI=bolt://neo4j:7687  # Connection URI (default: bolt://neo4j:7687)
NEO4J_USER=neo4j  # Username (default: neo4j)
NEO4J_PASSWORD=demodemo  # Password (default: demodemo)
NEO4J_DATABASE=neo4j  # Database name (default: neo4j)
USE_PARALLEL_RUNTIME=false  # Enterprise feature (default: false)
```

#### Accessing Neo4j

- **Web Interface:** http://localhost:7474
- **Bolt Protocol:** bolt://localhost:7687
- **MCP Server:** http://localhost:8000

Default credentials: `neo4j` / `demodemo`

#### Data Management

**Backup:**
```bash
# Backup both data and logs volumes
docker run --rm -v docker_neo4j_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/neo4j-data-backup.tar.gz -C /data .
docker run --rm -v docker_neo4j_logs:/logs -v $(pwd):/backup alpine \
  tar czf /backup/neo4j-logs-backup.tar.gz -C /logs .
```

**Restore:**
```bash
# Restore both volumes
docker run --rm -v docker_neo4j_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/neo4j-data-backup.tar.gz -C /data
docker run --rm -v docker_neo4j_logs:/logs -v $(pwd):/backup alpine \
  tar xzf /backup/neo4j-logs-backup.tar.gz -C /logs
```

**Clear Data:**
```bash
docker-compose -f docker-compose-neo4j.yml down
docker volume rm docker_neo4j_data docker_neo4j_logs
docker-compose -f docker-compose-neo4j.yml up
```

#### Gotchas
- Neo4j takes 30+ seconds to start up - wait for the health check
- The web interface requires authentication even for local access
- Memory heap is configured for 512MB initial, 1GB max
- Page cache is set to 512MB
- Enterprise features like parallel runtime require a license

## Switching Between Databases

To switch from FalkorDB to Neo4j (or vice versa):

1. **Stop current setup:**
   ```bash
   docker-compose down  # Stop FalkorDB combined image
   # or
   docker-compose -f docker-compose-neo4j.yml down  # Stop Neo4j
   ```

2. **Start new database:**
   ```bash
   docker-compose up  # Start FalkorDB combined image
   # or
   docker-compose -f docker-compose-neo4j.yml up  # Start Neo4j
   ```

Note: Data is not automatically migrated between different database types. You'll need to export from one and import to another using the MCP API.

## Troubleshooting

### Port Conflicts

If port 8000 is already in use:
```bash
# Find what's using the port
lsof -i :8000

# Pick a different host port (set in your shell or in a .env file next to the compose file)
MCP_HOST_PORT=8001 docker compose up
```

Host-side ports and the bind address can be set with environment variables, without editing the compose files:

| Variable | Default | Used by |
|---|---|---|
| `MCP_BIND_ADDRESS` | `127.0.0.1` | MCP server, all compose files |
| `MCP_HOST_PORT` | `8000` | MCP server, all compose files |
| `DB_BIND_ADDRESS` | `127.0.0.1` | FalkorDB (6379) / Neo4j Bolt (7687) |
| `FALKORDB_HOST_PORT` | `6379` | `docker-compose.yml`, `docker-compose-falkordb.yml` |
| `NEO4J_BOLT_HOST_PORT` | `7687` | `docker-compose-neo4j.yml` |
| `DB_UI_BIND_ADDRESS` | `127.0.0.1` | FalkorDB Browser (3000) / Neo4j Browser (7474) |
| `FALKORDB_BROWSER_HOST_PORT` | `3000` | `docker-compose.yml`, `docker-compose-falkordb.yml` |
| `NEO4J_HTTP_HOST_PORT` | `7474` | `docker-compose-neo4j.yml` |

Set a bind address to `0.0.0.0` or a specific interface IP to reach that port from other machines. The bind addresses are separate so you can expose the MCP server without also exposing the database, which has no password by default. Only the host side changes; services still listen on their default ports inside the containers.

### Container Won't Start

1. Check logs:
   ```bash
   docker-compose logs graphiti-mcp
   ```

2. Verify `.env` file exists and contains valid API keys:
   ```bash
   cat .env | grep API_KEY
   ```

3. Ensure Docker has enough resources allocated

### Database Connection Issues

**FalkorDB:**
- Test Redis connectivity: `docker compose exec graphiti-falkordb redis-cli ping`
- Check FalkorDB logs: `docker compose logs graphiti-falkordb`
- Verify both services started: Look for "FalkorDB is ready!" and "Starting MCP server..." in logs

**Neo4j:**
- Wait for health check to pass (can take 30+ seconds)
- Check Neo4j logs: `docker-compose -f docker-compose-neo4j.yml logs neo4j`
- Verify credentials match environment variables

**FalkorDB:**
- Test Redis connectivity: `redis-cli -h localhost ping`

### Data Not Persisting

1. Verify volumes are created:
   ```bash
   docker volume ls | grep docker_
   ```

2. Check volume mounts in container:
   ```bash
   docker inspect graphiti-mcp | grep -A 5 Mounts
   ```

3. Ensure proper shutdown:
   ```bash
   docker-compose down  # Not docker-compose down -v (which removes volumes)
   ```

### Performance Issues

**FalkorDB:**
- Adjust `SEMAPHORE_LIMIT` environment variable
- Monitor with: `docker stats graphiti-falkordb`
- Check Redis memory: `docker compose exec graphiti-falkordb redis-cli info memory`

**Neo4j:**
- Increase heap memory in docker-compose-neo4j.yml
- Adjust page cache size based on data size
- Check query performance in Neo4j browser

## Docker Resources

### Volumes

Each database configuration uses named volumes for data persistence:
- FalkorDB (combined): `falkordb_data`
- Neo4j: `neo4j_data`, `neo4j_logs`

### Networks

All configurations use the default bridge network. Services communicate using container names as hostnames.

### Resource Limits

No resource limits are set by default. To add limits, modify the docker-compose file:

```yaml
services:
  graphiti-mcp:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G
```

## Configuration Files

Each database has a dedicated configuration file in `../config/`:
- `config-docker-falkordb-combined.yaml` - FalkorDB combined image configuration
- `config-docker-neo4j.yaml` - Neo4j configuration

These files are mounted read-only into the container at `/app/mcp/config/config.yaml` (for combined image) or `/app/config/config.yaml` (for Neo4j).
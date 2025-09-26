# Dgraph Backup and Administration Guide

This guide covers the complete process for backing up, restoring, and administering a Dgraph database cluster.

## 🔄 Backup and Restoration Process

### Prerequisites

- SSH access to target server
- Docker and Docker Compose installed
- Backup files (export-backup.zip containing **g01.rdf.gz**, **g01.schema.gz** and **g01.gql_schema.gz**)

### Step 1: Transfer and Extract Backup Files

```bash
# Transfer backup file to target server
scp -r export-backup.zip dgraph.example.com:~/backup.zip

# Connect to server and extract files
ssh dgraph.example.com
unzip backup.zip

# Clean up unnecessary files
rm -rf __MACOSX

# Extract schema files
gunzip *schema.gz

# Generate HMAC secret file for ACL authentication (32 random characters)
tr -dc 'a-zA-Z0-9' < /dev/urandom | dd bs=1 count=32 of=hmac_secret_file
```

### Step 2: Docker Compose Configuration

Create or update the `docker-compose.yml` file:

```bash
nano docker-compose.yml
```

**Docker Compose Configuration:**

```yaml
services:
  # Dgraph Zero - Cluster orchestrator and shard assignment
  zero:
    image: dgraph/dgraph:v25.0.0-preview6
    container_name: zero
    working_dir: /data
    command: dgraph zero --my=zero:5080
    volumes:
      - dgraph:/data
    expose:
      - "5080"  # GRPC port
      - "6080"  # HTTP port
    restart: unless-stopped
    networks:
      - dgraph
    
  # Dgraph Alpha - Data serving node
  alpha:
    image: dgraph/dgraph:v25.0.0-preview6
    container_name: alpha
    working_dir: /data
    command: dgraph alpha --my=alpha:7080 --zero=zero:5080
    environment:
      # Enable ACL with HMAC secret file
      DGRAPH_ALPHA_ACL: secret-file=/data/acl/hmac_secret_file
      # Allow connections from any IP (adjust for production)
      DGRAPH_ALPHA_SECURITY: whitelist=0.0.0.0/0
    volumes:
      - dgraph:/data
      - ./hmac_secret_file:/data/acl/hmac_secret_file
    ports:
      - "8080:8080"  # HTTP API port
      - "9080:9080"  # GRPC port
    restart: unless-stopped
    depends_on:
      - zero
    networks:
      - dgraph
      
networks:
  dgraph:
    name: dgraph
    driver: bridge
    
volumes:
  dgraph:
    name: dgraph
    driver: local
```

### Step 3: Restore Data Using Bulk Import

```bash
# Start the cluster
docker compose up -d

# Stop alpha temporarily for bulk import
docker compose stop alpha

# Copy backup files to zero container
docker cp g01.schema zero:/data/
docker cp g01.gql_schema zero:/data/
docker cp g01.rdf.gz zero:/data/

# Enter zero container and perform bulk import
docker exec -it zero /bin/bash

# Inside the container:
# Remove existing data directory
rm -rf p

# Perform bulk import (-f: RDF file, -s: schema file)
dgraph bulk -f g01.rdf.gz -s g01.schema -g g01.gql_schema

# Move imported data to correct location
mv out/0/p .

# Clean up temporary files
rm -rf out/ g01.rdf.gz g01.schema g01.gql_schema.gz
exit

# Restart alpha to serve the imported data
docker compose start alpha

# Clean up host files
rm backup.zip g01.rdf.gz g01.schema g01.gql_schema
```

---

## 🔐 User and Group Management

### Authentication and Login

**Login as Root User (groot):**

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--data '{
  "query": "mutation {
    login(userId: \"groot\", password: \"password\", namespace: 0) {
      response {
        accessJWT
        refreshJWT
      }
    }
  }",
  "variables": {}
}'
```

> **Note:** Replace `access_token` in subsequent commands with the `accessJWT` value from the login response.

### User Management

#### Reset Root Password

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    updateUser(
      input: {
        filter: { name: { eq: \"groot\" } }
        set: { password: \"SuperSecuredPassword\" }
      }
    ) {
      user {
        name
      }
    }
  }",
  "variables": {}
}'
```

#### Add New User

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    addUser(input: [{ name: \"alice\", password: \"whiterabbit\" }]) {
      user {
        name
      }
    }
  }",
  "variables": {}
}'
```

#### Delete User

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    deleteUser(filter: { name: { eq: \"alice\" } }) {
      msg
      numUids
    }
  }",
  "variables": {}
}'
```

### Group Management

#### Create New Group

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    addGroup(input: [{ name: \"dev\" }]) {
      group {
        name
        users {
          name
        }
      }
    }
  }",
  "variables": {}
}'
```

#### Add User to Group

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    updateUser(
      input: {
        filter: { name: { eq: \"alice\" } }
        set: { groups: [{ name: \"dev\" }] }
      }
    ) {
      user {
        name
        groups {
          name
        }
      }
    }
  }",
  "variables": {}
}'
```

#### Remove User from Group

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    updateUser(
      input: {
        filter: { name: { eq: \"alice\" } }
        remove: { groups: [{ name: \"dev\" }] }
      }
    ) {
      user {
        name
        groups {
          name
        }
      }
    }
  }",
  "variables": {}
}'
```

#### Delete Group

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    deleteGroup(filter: { name: { eq: \"dev\" } }) {
      msg
      numUids
    }
  }",
  "variables": {}
}'
```

---

## 📤 Data Export Options

### Local Export

Export data to the local container filesystem:

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    export(input: {
      format: \"rdf\"
    }) {
      response {
        message
        code
      }
    }
  }",
  "variables": {}
}'
```

### S3 Export

Export data directly to Amazon S3:

```bash
curl --location 'http://localhost:8080/admin' \
--header 'Content-Type: application/json' \
--header 'X-Dgraph-AccessToken: access_token' \
--data '{
  "query": "mutation {
    export(input: {
      destination: \"s3://s3.<region>.amazonaws.com/<bucket_name>\"
      accessKey: \"access_key\"
      secretKey: \"secret_key\"
    }) {
      response {
        message
        code
      }
    }
  }",
  "variables": {}
}'
```

> **Security Note:** Replace `<region>`, `<bucket_name>`, `access_key`, and `secret_key` with your actual AWS S3 credentials and configuration.

---

## 🔧 Important Notes

- **Production Security**: Change the whitelist setting from `0.0.0.0/0` to restrict access to specific IP ranges
- **Password Security**: Always change default passwords before production use
- **Backup Strategy**: Regular exports should be scheduled for data protection
- **Access Tokens**: JWT tokens expire and may need to be refreshed for long-running operations
- **Container Persistence**: The `dgraph` volume ensures data persists across container restarts

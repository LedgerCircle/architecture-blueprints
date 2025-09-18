# LedgerLoop Deployment & Operations Guide

## Overview

This document provides comprehensive guidance for deploying, monitoring, and operating the LedgerLoop system in production environments. It covers infrastructure requirements, deployment procedures, monitoring strategies, and operational best practices.

## Infrastructure Requirements

### Minimum Production Environment

#### Computing Resources

**Frontend Services**
- **Instances**: 3 x t3.medium (2 vCPU, 4 GB RAM)
- **Load Balancer**: Application Load Balancer with SSL termination
- **CDN**: CloudFront for static asset delivery
- **Auto Scaling**: Target 70% CPU utilization

**Backend Services**
- **API Services**: 5 x t3.large (2 vCPU, 8 GB RAM)
- **Background Workers**: 3 x t3.medium (2 vCPU, 4 GB RAM)
- **Message Queue**: 2 x t3.medium (Redis Cluster)
- **Auto Scaling**: Target 80% CPU utilization

**Database Tier**
- **Primary Database**: RDS PostgreSQL (db.r5.xlarge - 4 vCPU, 32 GB RAM)
- **Read Replicas**: 2 x db.r5.large (2 vCPU, 16 GB RAM)
- **Cache Layer**: ElastiCache Redis (cache.r5.large - 2 vCPU, 13 GB RAM)
- **Backup**: Automated daily backups with 30-day retention

#### Storage Requirements

**Database Storage**
- **Primary**: 1 TB SSD with provisioned IOPS (3000 IOPS)
- **Backup**: 5 TB for automated backups and point-in-time recovery
- **Archive**: S3 Glacier for long-term compliance storage

**File Storage**
- **Application Files**: 500 GB SSD for application data
- **Document Storage**: IPFS cluster with 2 TB distributed storage
- **Log Storage**: 1 TB with automated rotation and archival

#### Network Requirements

**Bandwidth**
- **Internet Gateway**: 10 Gbps with burst capability
- **Internal Network**: 25 Gbps between availability zones
- **Database Network**: Dedicated 10 Gbps for database cluster

**Security Groups**
```yaml
# Security group configurations
web_tier:
  ingress:
    - port: 443 (HTTPS)
      source: 0.0.0.0/0
    - port: 80 (HTTP - redirect to HTTPS)
      source: 0.0.0.0/0
  egress:
    - all traffic to app_tier

app_tier:
  ingress:
    - port: 3000-3010
      source: web_tier
    - port: 8080 (health checks)
      source: load_balancer
  egress:
    - port: 5432 to db_tier
    - port: 6379 to cache_tier
    - port: 443 to external_apis

db_tier:
  ingress:
    - port: 5432
      source: app_tier
    - port: 6379 (Redis)
      source: app_tier
  egress:
    - none (database tier isolated)
```

### High Availability Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Production Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│  Internet Gateway & Route 53                                    │
├─────────────────────────────────────────────────────────────────┤
│  CloudFront CDN & WAF                                           │
├─────────────────────────────────────────────────────────────────┤
│  Application Load Balancer (Multi-AZ)                          │
├─────────────────────────────────────────────────────────────────┤
│  Availability Zone A        │        Availability Zone B       │
│  ┌─────────────────────────┐│┌─────────────────────────────────┐│
│  │ Web Tier               ││││ Web Tier                       ││
│  │ - React Apps (2x)      ││││ - React Apps (2x)              ││
│  │ - NGINX Reverse Proxy  ││││ - NGINX Reverse Proxy          ││
│  └─────────────────────────┘│└─────────────────────────────────┘│
│  ┌─────────────────────────┐│┌─────────────────────────────────┐│
│  │ Application Tier       ││││ Application Tier               ││
│  │ - Node.js APIs (3x)    ││││ - Node.js APIs (3x)            ││
│  │ - Background Workers   ││││ - Background Workers           ││
│  │ - Message Queue        ││││ - Message Queue                ││
│  └─────────────────────────┘│└─────────────────────────────────┘│
│  ┌─────────────────────────┐│┌─────────────────────────────────┐│
│  │ Data Tier              ││││ Data Tier                      ││
│  │ - PostgreSQL Primary   ││││ - PostgreSQL Read Replica      ││
│  │ - Redis Master         ││││ - Redis Replica                ││
│  │ - IPFS Node            ││││ - IPFS Node                    ││
│  └─────────────────────────┘│└─────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Deployment Procedures

### Container Orchestration

#### Kubernetes Configuration

```yaml
# Namespace for LedgerLoop
apiVersion: v1
kind: Namespace
metadata:
  name: ledgerloop-prod
  labels:
    environment: production
    team: platform

---
# API Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ledgerloop-api
  namespace: ledgerloop-prod
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: ledgerloop-api
  template:
    metadata:
      labels:
        app: ledgerloop-api
        version: v1.0.0
    spec:
      containers:
      - name: api
        image: ledgerloop/api:v1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secrets
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secrets
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false

---
# Service for API
apiVersion: v1
kind: Service
metadata:
  name: ledgerloop-api-service
  namespace: ledgerloop-prod
spec:
  selector:
    app: ledgerloop-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ledgerloop-api-hpa
  namespace: ledgerloop-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ledgerloop-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### CI/CD Pipeline

#### GitHub Actions Workflow

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ledgerloop/api

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Security Scan
      uses: github/super-linter@v4
      env:
        DEFAULT_BRANCH: main
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        VALIDATE_JAVASCRIPT: true
        VALIDATE_TYPESCRIPT: true
        VALIDATE_DOCKERFILE: true
    
    - name: Run Dependency Scan
      run: |
        npm audit --audit-level=high
        npm run security:scan

  test:
    needs: security-scan
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: ledgerloop_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm run test:unit
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgresql://postgres:test@localhost:5432/ledgerloop_test
        REDIS_URL: redis://localhost:6379
    
    - name: Run E2E tests
      run: npm run test:e2e

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.image.outputs.image }}
      digest: ${{ steps.build.outputs.digest }}
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
    
    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - id: image
      run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}" >> $GITHUB_OUTPUT

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Staging
      run: |
        echo "Deploying ${{ needs.build.outputs.image }} to staging"
        # Deploy to staging environment
        kubectl set image deployment/ledgerloop-api api=${{ needs.build.outputs.image }} -n ledgerloop-staging
        kubectl rollout status deployment/ledgerloop-api -n ledgerloop-staging --timeout=300s

  integration-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Integration Tests
      run: |
        npm run test:integration:staging
        npm run test:e2e:staging

  deploy-production:
    needs: [build, integration-tests]
    runs-on: ubuntu-latest
    environment: production
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Production
      run: |
        echo "Deploying ${{ needs.build.outputs.image }} to production"
        
        # Blue-Green Deployment
        kubectl patch deployment ledgerloop-api -p '{"spec":{"template":{"spec":{"containers":[{"name":"api","image":"${{ needs.build.outputs.image }}"}]}}}}' -n ledgerloop-prod
        
        # Wait for rollout
        kubectl rollout status deployment/ledgerloop-api -n ledgerloop-prod --timeout=600s
        
        # Health check
        kubectl wait --for=condition=available --timeout=300s deployment/ledgerloop-api -n ledgerloop-prod
        
        # Post-deployment tests
        npm run test:smoke:production
```

### Database Migration Strategy

#### Migration Management

```javascript
// Database migration system
class MigrationManager {
  constructor(client) {
    this.client = client;
    this.migrationsPath = './migrations';
  }

  async runMigrations() {
    // Create migrations table if it doesn't exist
    await this.createMigrationsTable();
    
    // Get pending migrations
    const pendingMigrations = await this.getPendingMigrations();
    
    console.log(`Found ${pendingMigrations.length} pending migrations`);
    
    // Run migrations in transaction
    await this.client.transaction(async (trx) => {
      for (const migration of pendingMigrations) {
        console.log(`Running migration: ${migration.name}`);
        
        try {
          await migration.up(trx);
          await this.recordMigration(trx, migration.name);
          console.log(`✓ Migration ${migration.name} completed`);
        } catch (error) {
          console.error(`✗ Migration ${migration.name} failed:`, error);
          throw error;
        }
      }
    });
  }

  async createMigrationsTable() {
    await this.client.raw(`
      CREATE TABLE IF NOT EXISTS migrations (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) NOT NULL UNIQUE,
        executed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
      )
    `);
  }

  async getPendingMigrations() {
    const executedMigrations = await this.client('migrations')
      .select('name')
      .then(rows => rows.map(row => row.name));

    const allMigrations = fs.readdirSync(this.migrationsPath)
      .filter(file => file.endsWith('.js'))
      .sort()
      .map(file => ({
        name: file,
        ...require(path.join(this.migrationsPath, file))
      }));

    return allMigrations.filter(migration => 
      !executedMigrations.includes(migration.name)
    );
  }
}

// Example migration file: 001_create_users_table.js
module.exports = {
  async up(knex) {
    return knex.schema.createTable('users', (table) => {
      table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
      table.string('email').unique().notNullable();
      table.string('password_hash');
      table.jsonb('profile').notNullable();
      table.specificType('xrpl_addresses', 'text[]');
      table.string('kyc_status').defaultTo('pending');
      table.integer('credit_score');
      table.boolean('is_active').defaultTo(true);
      table.timestamps(true, true);
      
      table.index('email');
      table.index('kyc_status');
    });
  },

  async down(knex) {
    return knex.schema.dropTable('users');
  }
};
```

## Monitoring & Observability

### Metrics Collection

#### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "ledgerloop_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'ledgerloop-api'
    static_configs:
      - targets: ['api:3000']
    metrics_path: '/metrics'
    scrape_interval: 10s
    
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
    
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

#### Custom Metrics

```javascript
// Application metrics
const promClient = require('prom-client');

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
});

const activeUsers = new promClient.Gauge({
  name: 'ledgerloop_active_users',
  help: 'Number of active users',
  collect() {
    // Implement logic to get active user count
    this.set(getActiveUserCount());
  }
});

const transactionCount = new promClient.Counter({
  name: 'ledgerloop_transactions_total',
  help: 'Total number of transactions',
  labelNames: ['type', 'status']
});

const circleCount = new promClient.Gauge({
  name: 'ledgerloop_circles_total',
  help: 'Total number of circles',
  labelNames: ['status']
});

// Middleware to track request metrics
function metricsMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || 'unknown', res.statusCode)
      .observe(duration);
  });
  
  next();
}
```

### Alerting Rules

```yaml
# ledgerloop_rules.yml
groups:
- name: ledgerloop_alerts
  rules:
  
  # High error rate
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }} errors per second"

  # Database connection issues
  - alert: DatabaseConnectionFailure
    expr: up{job="postgres"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Database connection failed"
      description: "PostgreSQL database is not responding"

  # High response time
  - alert: HighResponseTime
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High response time"
      description: "95th percentile response time is {{ $value }}s"

  # Memory usage
  - alert: HighMemoryUsage
    expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage"
      description: "Memory usage is {{ $value | humanizePercentage }}"

  # Failed transactions
  - alert: FailedTransactions
    expr: increase(ledgerloop_transactions_total{status="failed"}[10m]) > 10
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High number of failed transactions"
      description: "{{ $value }} transactions failed in the last 10 minutes"
```

### Logging Strategy

#### Structured Logging

```javascript
// Logging configuration
const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'ledgerloop-api',
    version: process.env.APP_VERSION,
    environment: process.env.NODE_ENV
  },
  transports: [
    // Console output for development
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    
    // File output for production
    new winston.transports.File({
      filename: '/var/log/ledgerloop/error.log',
      level: 'error'
    }),
    
    new winston.transports.File({
      filename: '/var/log/ledgerloop/combined.log'
    }),
    
    // Elasticsearch for centralized logging
    new ElasticsearchTransport({
      level: 'info',
      clientOpts: {
        node: process.env.ELASTICSEARCH_URL
      },
      index: 'ledgerloop-logs'
    })
  ]
});

// Request logging middleware
function requestLogger(req, res, next) {
  const start = Date.now();
  
  // Log request
  logger.info('Request received', {
    method: req.method,
    url: req.url,
    userAgent: req.get('user-agent'),
    ip: req.ip,
    userId: req.user?.id,
    requestId: req.id
  });
  
  // Log response
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    logger.info('Request completed', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration,
      userId: req.user?.id,
      requestId: req.id
    });
  });
  
  next();
}

// Error logging
function errorLogger(error, req, res, next) {
  logger.error('Application error', {
    error: error.message,
    stack: error.stack,
    method: req.method,
    url: req.url,
    userId: req.user?.id,
    requestId: req.id
  });
  
  next(error);
}
```

## Backup & Disaster Recovery

### Backup Strategy

#### Database Backups

```bash
#!/bin/bash
# backup.sh - Database backup script

set -e

# Configuration
DB_HOST=${DB_HOST:-localhost}
DB_PORT=${DB_PORT:-5432}
DB_NAME=${DB_NAME:-ledgerloop}
DB_USER=${DB_USER:-postgres}
BACKUP_DIR=${BACKUP_DIR:-/backups}
S3_BUCKET=${S3_BUCKET:-ledgerloop-backups}
RETENTION_DAYS=${RETENTION_DAYS:-30}

# Create backup directory
mkdir -p $BACKUP_DIR

# Generate backup filename
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/ledgerloop_$TIMESTAMP.sql"

echo "Starting database backup..."

# Create database backup
pg_dump -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  --verbose --clean --no-owner --no-privileges \
  --format=custom --compress=9 \
  --file=$BACKUP_FILE

# Verify backup
if [ $? -eq 0 ]; then
  echo "Database backup completed successfully: $BACKUP_FILE"
  
  # Compress backup
  gzip $BACKUP_FILE
  BACKUP_FILE_GZ="$BACKUP_FILE.gz"
  
  # Upload to S3
  aws s3 cp $BACKUP_FILE_GZ s3://$S3_BUCKET/database/$(basename $BACKUP_FILE_GZ)
  
  # Verify upload
  if [ $? -eq 0 ]; then
    echo "Backup uploaded to S3 successfully"
    rm $BACKUP_FILE_GZ
  else
    echo "Failed to upload backup to S3"
    exit 1
  fi
  
  # Clean up old backups
  find $BACKUP_DIR -name "ledgerloop_*.sql.gz" -mtime +$RETENTION_DAYS -delete
  
  # Clean up old S3 backups
  aws s3 ls s3://$S3_BUCKET/database/ | \
    awk '{print $4}' | \
    while read file; do
      if [ $(aws s3api head-object --bucket $S3_BUCKET --key database/$file --query 'LastModified' --output text | \
           xargs -I {} date -d {} +%s) -lt $(date -d "$RETENTION_DAYS days ago" +%s) ]; then
        aws s3 rm s3://$S3_BUCKET/database/$file
      fi
    done
    
else
  echo "Database backup failed"
  exit 1
fi

echo "Backup process completed"
```

#### IPFS Backup Strategy

```javascript
// IPFS backup management
class IPFSBackupManager {
  constructor(ipfsClient, s3Client) {
    this.ipfs = ipfsClient;
    this.s3 = s3Client;
    this.backupBucket = process.env.IPFS_BACKUP_BUCKET;
  }

  async backupIPFSData() {
    try {
      // Get all pinned hashes
      const pinnedHashes = [];
      for await (const pin of this.ipfs.pin.ls()) {
        pinnedHashes.push(pin.cid.toString());
      }

      console.log(`Found ${pinnedHashes.length} pinned objects`);

      // Export and backup each object
      for (const hash of pinnedHashes) {
        await this.backupObject(hash);
      }

      // Create backup manifest
      const manifest = {
        timestamp: new Date().toISOString(),
        totalObjects: pinnedHashes.length,
        hashes: pinnedHashes
      };

      await this.uploadManifest(manifest);
      
      console.log('IPFS backup completed successfully');
    } catch (error) {
      console.error('IPFS backup failed:', error);
      throw error;
    }
  }

  async backupObject(hash) {
    try {
      // Export object from IPFS
      const chunks = [];
      for await (const chunk of this.ipfs.cat(hash)) {
        chunks.push(chunk);
      }
      const data = Buffer.concat(chunks);

      // Upload to S3
      const key = `ipfs-objects/${hash}`;
      await this.s3.upload({
        Bucket: this.backupBucket,
        Key: key,
        Body: data,
        StorageClass: 'STANDARD_IA' // Infrequent access
      }).promise();

      console.log(`Backed up object: ${hash}`);
    } catch (error) {
      console.error(`Failed to backup object ${hash}:`, error);
      throw error;
    }
  }

  async restoreFromBackup(hash) {
    try {
      const key = `ipfs-objects/${hash}`;
      const object = await this.s3.getObject({
        Bucket: this.backupBucket,
        Key: key
      }).promise();

      // Add back to IPFS
      const result = await this.ipfs.add(object.Body);
      await this.ipfs.pin.add(result.cid);

      console.log(`Restored object: ${hash}`);
      return result.cid.toString();
    } catch (error) {
      console.error(`Failed to restore object ${hash}:`, error);
      throw error;
    }
  }
}
```

### Disaster Recovery Plan

#### Recovery Time Objectives (RTO)

| Component | RTO | RPO | Recovery Strategy |
|-----------|-----|-----|-------------------|
| Frontend | 15 minutes | 1 hour | Multi-region deployment |
| API Services | 30 minutes | 5 minutes | Auto-scaling + health checks |
| Database | 1 hour | 15 minutes | Read replicas + point-in-time recovery |
| IPFS Storage | 2 hours | 1 hour | Distributed nodes + S3 backup |
| XRPL Integration | 5 minutes | 0 | Blockchain inherent redundancy |

#### Failover Procedures

```bash
#!/bin/bash
# disaster-recovery.sh - Automated disaster recovery

REGION_PRIMARY="us-east-1"
REGION_SECONDARY="us-west-2"
ENVIRONMENT="production"

echo "Starting disaster recovery process..."

# Check primary region health
check_primary_health() {
  curl -f --max-time 30 https://api-primary.ledgerloop.com/health || return 1
}

# Failover to secondary region
failover_to_secondary() {
  echo "Failing over to secondary region..."
  
  # Update Route 53 to point to secondary region
  aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch file://failover-changeset.json \
    --region $REGION_SECONDARY
  
  # Scale up secondary region
  kubectl scale deployment ledgerloop-api --replicas=5 \
    --context=secondary-cluster
  
  # Promote read replica to primary
  aws rds promote-read-replica \
    --db-instance-identifier ledgerloop-replica-secondary \
    --region $REGION_SECONDARY
  
  echo "Failover completed"
}

# Perform health check
if ! check_primary_health; then
  echo "Primary region is unhealthy, initiating failover"
  failover_to_secondary
  
  # Send alerts
  aws sns publish \
    --topic-arn $ALERT_TOPIC_ARN \
    --message "Disaster recovery activated - failed over to secondary region" \
    --subject "LedgerLoop Disaster Recovery Alert"
else
  echo "Primary region is healthy"
fi
```

## Performance Optimization

### Database Performance

#### Query Optimization

```sql
-- Performance monitoring queries

-- Slow query analysis
SELECT 
  query,
  calls,
  total_time,
  mean_time,
  rows,
  100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 20;

-- Index usage analysis
SELECT 
  schemaname,
  tablename,
  attname,
  n_distinct,
  correlation
FROM pg_stats
WHERE schemaname = 'public'
ORDER BY n_distinct DESC;

-- Lock analysis
SELECT 
  t.schemaname,
  t.tablename,
  l.locktype,
  l.mode,
  l.granted,
  a.usename,
  a.query,
  a.query_start,
  age(now(), a.query_start) AS duration
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
JOIN pg_tables t ON l.relation = t.schemaname||'.'||t.tablename::regclass
WHERE NOT l.granted
ORDER BY a.query_start;
```

#### Connection Pooling

```javascript
// Database connection pool configuration
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  
  // Pool configuration
  min: 10,                    // Minimum connections
  max: 100,                   // Maximum connections
  idleTimeoutMillis: 30000,   // Close idle connections after 30s
  connectionTimeoutMillis: 5000, // Fail fast if can't connect
  
  // SSL configuration
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: false
  } : false,
  
  // Connection validation
  application_name: 'ledgerloop-api',
  
  // Performance tuning
  statement_timeout: 30000,   // 30 second query timeout
  query_timeout: 30000,
  keepAlive: true,
  keepAliveInitialDelayMillis: 10000,
});

// Connection monitoring
pool.on('connect', (client) => {
  console.log('New database connection established');
});

pool.on('error', (err, client) => {
  console.error('Database pool error:', err);
  // Implement alerting here
});

pool.on('remove', (client) => {
  console.log('Database connection removed from pool');
});

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('Closing database pool...');
  pool.end(() => {
    console.log('Database pool closed');
    process.exit(0);
  });
});
```

### Caching Strategy

#### Redis Configuration

```javascript
// Redis cluster configuration
const Redis = require('ioredis');

const redis = new Redis.Cluster([
  {
    host: process.env.REDIS_HOST_1,
    port: process.env.REDIS_PORT_1
  },
  {
    host: process.env.REDIS_HOST_2,
    port: process.env.REDIS_PORT_2
  },
  {
    host: process.env.REDIS_HOST_3,
    port: process.env.REDIS_PORT_3
  }
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
    connectTimeout: 5000,
    commandTimeout: 3000,
    retryDelayOnFailover: 100,
    maxRetriesPerRequest: 3,
    lazyConnect: true
  },
  enableOfflineQueue: false,
  retryDelayOnSlotDataRequest: 200,
  maxRedirections: 16,
  scaleReads: 'slave'
});

// Cache service implementation
class CacheService {
  constructor(redisClient) {
    this.redis = redisClient;
    this.defaultTTL = 3600; // 1 hour
  }

  async get(key) {
    try {
      const value = await this.redis.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set(key, value, ttl = this.defaultTTL) {
    try {
      await this.redis.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }

  async del(key) {
    try {
      await this.redis.del(key);
    } catch (error) {
      console.error('Cache delete error:', error);
    }
  }

  async invalidatePattern(pattern) {
    try {
      const keys = await this.redis.keys(pattern);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } catch (error) {
      console.error('Cache invalidation error:', error);
    }
  }

  // Cache-aside pattern
  async getOrSet(key, fetchFunction, ttl = this.defaultTTL) {
    let value = await this.get(key);
    
    if (value === null) {
      value = await fetchFunction();
      if (value !== null && value !== undefined) {
        await this.set(key, value, ttl);
      }
    }
    
    return value;
  }
}
```

## Security Operations

### Security Monitoring

#### SIEM Integration

```javascript
// Security event correlation
class SecurityEventProcessor {
  constructor(siemClient) {
    this.siem = siemClient;
    this.riskThresholds = {
      low: 0.3,
      medium: 0.6,
      high: 0.8,
      critical: 0.9
    };
  }

  async processSecurityEvent(event) {
    // Enrich event with context
    const enrichedEvent = await this.enrichEvent(event);
    
    // Calculate risk score
    const riskScore = await this.calculateRiskScore(enrichedEvent);
    enrichedEvent.riskScore = riskScore;
    
    // Determine severity
    const severity = this.determineSeverity(riskScore);
    enrichedEvent.severity = severity;
    
    // Send to SIEM
    await this.siem.sendEvent(enrichedEvent);
    
    // Trigger automated response if needed
    if (riskScore >= this.riskThresholds.high) {
      await this.triggerAutomatedResponse(enrichedEvent);
    }
    
    return enrichedEvent;
  }

  async enrichEvent(event) {
    const enriched = { ...event };
    
    // Add geolocation data
    if (event.sourceIP) {
      enriched.geolocation = await this.getGeolocation(event.sourceIP);
    }
    
    // Add user context
    if (event.userId) {
      enriched.userContext = await this.getUserContext(event.userId);
    }
    
    // Add threat intelligence
    if (event.sourceIP) {
      enriched.threatIntel = await this.getThreatIntelligence(event.sourceIP);
    }
    
    return enriched;
  }

  async calculateRiskScore(event) {
    let score = 0;
    
    // Geographic risk
    if (event.geolocation?.country !== event.userContext?.expectedCountry) {
      score += 0.3;
    }
    
    // Time-based risk
    const hourOfDay = new Date().getHours();
    if (hourOfDay < 6 || hourOfDay > 22) {
      score += 0.2;
    }
    
    // Velocity risk
    const recentEvents = await this.getRecentEvents(event.userId, '1h');
    if (recentEvents.length > 10) {
      score += 0.4;
    }
    
    // Threat intelligence risk
    if (event.threatIntel?.isMalicious) {
      score += 0.6;
    }
    
    // Device risk
    if (event.deviceFingerprint !== event.userContext?.knownDevices) {
      score += 0.3;
    }
    
    return Math.min(score, 1.0);
  }
}
```

### Incident Response Automation

```javascript
// Automated incident response
class IncidentResponseAutomation {
  constructor() {
    this.responsePlaybooks = new Map();
    this.initializePlaybooks();
  }

  initializePlaybooks() {
    // Brute force attack response
    this.responsePlaybooks.set('BRUTE_FORCE_ATTACK', {
      immediate: [
        'lockAccount',
        'blockSourceIP',
        'notifySecurityTeam'
      ],
      investigation: [
        'analyzeAttackPattern',
        'checkForOtherTargets',
        'reviewAccessLogs'
      ],
      recovery: [
        'unlockAccountAfterVerification',
        'updateSecurityRules'
      ]
    });

    // Data exfiltration response
    this.responsePlaybooks.set('DATA_EXFILTRATION', {
      immediate: [
        'isolateAccount',
        'revokeAllTokens',
        'blockDataAccess',
        'alertManagement'
      ],
      investigation: [
        'identifyExfiltratedData',
        'traceDataFlow',
        'assessImpact'
      ],
      recovery: [
        'notifyAffectedUsers',
        'implementAdditionalControls',
        'conductSecurityReview'
      ]
    });
  }

  async executeResponse(incidentType, context) {
    const playbook = this.responsePlaybooks.get(incidentType);
    if (!playbook) {
      throw new Error(`No playbook found for incident type: ${incidentType}`);
    }

    const results = {
      immediate: [],
      investigation: [],
      recovery: []
    };

    // Execute immediate actions
    for (const action of playbook.immediate) {
      try {
        const result = await this.executeAction(action, context);
        results.immediate.push({ action, result, status: 'success' });
      } catch (error) {
        results.immediate.push({ action, error: error.message, status: 'failed' });
      }
    }

    // Schedule investigation actions
    this.scheduleInvestigation(playbook.investigation, context);

    return results;
  }

  async executeAction(action, context) {
    switch (action) {
      case 'lockAccount':
        return await this.lockAccount(context.userId);
      
      case 'blockSourceIP':
        return await this.blockIP(context.sourceIP);
      
      case 'isolateAccount':
        return await this.isolateAccount(context.userId);
      
      case 'revokeAllTokens':
        return await this.revokeAllTokens(context.userId);
      
      default:
        throw new Error(`Unknown action: ${action}`);
    }
  }
}
```

This comprehensive deployment and operations guide provides the foundation for successfully deploying, monitoring, and maintaining the LedgerLoop system in production environments. The guide emphasizes automation, monitoring, and security best practices to ensure reliable and secure operations.
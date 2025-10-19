# Chapter 8: The Muscular System (Compute/Processing Power)

## Introduction: Scaling Computational Workloads

Just as muscles provide the physical power for organisms to move, act, and respond to their environment, your infrastructure's compute layer provides the processing power to handle workloads, respond to requests, and execute tasks at scale.

**What You'll Learn:**
- Container orchestration with Kubernetes
- Job queue systems (BullMQ, Temporal)
- Auto-scaling strategies and implementation
- Resource management (CPU, memory, GPU)
- Distributed computing patterns
- Performance optimization techniques
- Cost optimization strategies

**Real-World Scenario:**

Your application needs to:
- Handle variable traffic (10 requests/sec → 10,000 requests/sec)
- Process background jobs (video encoding, report generation)
- Run scheduled tasks (backups, cleanups, ML training)
- Respond quickly under load
- Scale automatically based on demand
- Minimize infrastructure costs

This chapter provides production-ready solutions for all these requirements.

---

## The Evolution of Compute Management

### Phase 1: Single Server (2000s)

```
┌────────────────────────────┐
│      One Big Server        │
│                            │
│  ┌──────────────────────┐ │
│  │  Web App             │ │
│  │  Background Jobs     │ │
│  │  Database            │ │
│  │  Cache               │ │
│  └──────────────────────┘ │
│                            │
│  Resources: 8 CPU, 16GB   │
└────────────────────────────┘

Problems:
❌ Single point of failure
❌ Can't scale horizontally
❌ Resource contention
❌ Expensive to upgrade
```

### Phase 2: Multiple Servers (2010s)

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Web 1   │  │  Web 2   │  │  Web 3   │
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
      └─────────────┴─────────────┘
                    │
      ┌─────────────▼─────────────┐
      │     Load Balancer         │
      └───────────────────────────┘

┌──────────┐  ┌──────────┐
│ Worker 1 │  │ Worker 2 │
└──────────┘  └──────────┘

Improvements:
✅ Redundancy (no single point of failure)
✅ Horizontal scaling (add more servers)

Limitations:
⚠️ Manual management
⚠️ Resource underutilization
⚠️ Complex deployment
```

### Phase 3: Virtual Machines (2015+)

```
┌─────────────────────────────────────┐
│          Physical Server            │
│  ┌────────┐  ┌────────┐  ┌────────┐│
│  │  VM 1  │  │  VM 2  │  │  VM 3  ││
│  │  App A │  │  App B │  │  App C ││
│  │  2 CPU │  │  4 CPU │  │  2 CPU ││
│  │  4GB   │  │  8GB   │  │  4GB   ││
│  └────────┘  └────────┘  └────────┘│
└─────────────────────────────────────┘

Improvements:
✅ Better resource utilization
✅ Isolation between apps
✅ Cloud providers (AWS, GCP, Azure)

Limitations:
⚠️ Slow to start (minutes)
⚠️ Resource overhead (each VM needs full OS)
⚠️ Still manual scaling
```

### Phase 4: Containers + Orchestration (2020+)

```
┌─────────────────────────────────────────────┐
│         Kubernetes Cluster                  │
│                                             │
│  Node 1          Node 2          Node 3    │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐│
│  │ Pod: 3  │    │ Pod: 5  │    │ Pod: 4  ││
│  │ Pods    │    │ Pods    │    │ Pods    ││
│  └─────────┘    └─────────┘    └─────────┘│
│                                             │
│  Auto-scaling: ━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│  Self-healing: ✓                           │
│  Load balancing: ✓                         │
│  Rolling updates: ✓                        │
└─────────────────────────────────────────────┘

Modern Advantages:
✅ Auto-scaling (horizontal + vertical)
✅ Self-healing (automatic recovery)
✅ Fast startup (seconds)
✅ Efficient resource usage
✅ Declarative configuration
✅ Built-in load balancing
✅ Rolling updates (zero downtime)
```

This is the architecture we'll implement in this chapter.

---

## Core Compute Patterns

### Pattern 1: Request-Response (Synchronous)

User waits for immediate response.

```
┌──────────┐      Request       ┌──────────┐
│  Client  │ ──────────────────>│   API    │
│          │                     │  Server  │
│          │ <──────────────────│          │
└──────────┘      Response      └──────────┘
              (100-500ms)

Use Cases:
- API endpoints
- Web page rendering
- Database queries
- Authentication

Requirements:
- Low latency (<500ms)
- High availability (99.9%+)
- Auto-scaling for traffic spikes
- Load balancing
```

### Pattern 2: Background Jobs (Asynchronous)

User doesn't wait, job processes in background.

```
┌──────────┐      Submit       ┌──────────┐
│  Client  │ ─────────────────>│   API    │
│          │                    │  Server  │
│          │ <─────────────────│          │
└──────────┘   "Job queued"    └────┬─────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  Job Queue   │
                              └──────┬───────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │   Worker     │
                              │  (processes  │
                              │   in bg)     │
                              └──────────────┘

Use Cases:
- Video/image processing
- Report generation
- Email sending
- Data imports/exports
- ML model training

Characteristics:
- Can take seconds to hours
- Retry on failure
- Priority queues
- Progress tracking
```

### Pattern 3: Scheduled Tasks (Cron Jobs)

Tasks run on schedule, regardless of user activity.

```
Schedule: "0 2 * * *" (Every day at 2 AM)
    ↓
┌────────────────┐
│  Scheduler     │
│  (Kubernetes   │
│   CronJob)     │
└────────┬───────┘
         │
         ▼
  ┌──────────────┐
  │   Execute    │
  │   Task Pod   │
  └──────────────┘

Use Cases:
- Database backups
- Log cleanup
- Report generation
- Data synchronization
- Health checks
- Certificate renewal

Characteristics:
- Time-based execution
- Idempotent (safe to run multiple times)
- Monitoring/alerting
- Resource limits
```

### Pattern 4: Stream Processing (Continuous)

Always-on processing of data streams.

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Kafka     │─────>│   Stream    │─────>│  Database   │
│  (Events)   │      │  Processor  │      │   (Output)  │
└─────────────┘      │  (Always    │      └─────────────┘
                     │   running)  │
                     └─────────────┘

Use Cases:
- Real-time analytics
- Fraud detection
- Log processing
- Metrics aggregation
- Event transformation

Characteristics:
- Stateful processing
- Exactly-once semantics
- Checkpointing
- High throughput
- Low latency
```

### Pattern 5: Batch Processing (Large Scale)

Process large datasets in batches.

```
Trigger: Schedule or Event
    ↓
┌─────────────────────────────────┐
│    Parallel Batch Processing    │
│                                 │
│  Worker 1 ─┐                   │
│  Worker 2 ─┤  Process chunks   │
│  Worker 3 ─┤  in parallel      │
│  Worker 4 ─┤                   │
│  Worker 5 ─┘                   │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│      Aggregate Results          │
└─────────────────────────────────┘

Use Cases:
- ETL pipelines
- ML training
- Data migrations
- Report generation
- Video encoding

Characteristics:
- Data parallelism
- Fault tolerance
- Progress tracking
- Resource intensive
```

---

## Building Block 1: Kubernetes (Container Orchestration)

Kubernetes manages containers at scale with automatic scaling, healing, and deployment.

### Kubernetes Architecture

```
┌─────────────────────────────────────────────────┐
│              Kubernetes Cluster                 │
│                                                 │
│  Control Plane:                                 │
│  ┌────────────────────────────────────────┐   │
│  │ API Server | Scheduler | Controller    │   │
│  │ etcd (cluster state storage)           │   │
│  └────────────────────────────────────────┘   │
│                                                 │
│  Worker Nodes:                                  │
│  ┌──────────────┐  ┌──────────────┐           │
│  │    Node 1    │  │    Node 2    │           │
│  │  ┌────────┐  │  │  ┌────────┐  │           │
│  │  │ Pod 1  │  │  │  │ Pod 3  │  │           │
│  │  │ Pod 2  │  │  │  │ Pod 4  │  │           │
│  │  └────────┘  │  │  └────────┘  │           │
│  │  Kubelet     │  │  Kubelet     │           │
│  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────┘

Key Concepts:
- Pod: Smallest deployable unit (1+ containers)
- Node: Worker machine (VM or physical)
- Deployment: Manages pod replicas
- Service: Stable endpoint for pods
- Ingress: External access (HTTP/HTTPS)
```

### Installing Kubernetes Locally (K3s)

K3s is a lightweight Kubernetes distribution, perfect for learning and development.

```bash
# Install K3s (single command!)
curl -sfL https://get.k3s.io | sh -

# Check it's running
sudo k3s kubectl get nodes

# Output:
# NAME     STATUS   ROLES                  AGE   VERSION
# ubuntu   Ready    control-plane,master   1m    v1.27.3+k3s1

# Set up kubectl alias (for convenience)
echo "alias kubectl='sudo k3s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Verify
kubectl get pods -A
```

### Your First Kubernetes Deployment

**deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 3  # Run 3 copies
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m      # 0.1 CPU cores
            memory: 128Mi  # 128 megabytes
          limits:
            cpu: 200m      # Max 0.2 CPU cores
            memory: 256Mi  # Max 256 megabytes
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

**Deploy it:**

```bash
# Apply configuration
kubectl apply -f deployment.yaml

# Check deployment
kubectl get deployments
# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# web-app   3/3     3            3           1m

# Check pods
kubectl get pods
# NAME                       READY   STATUS    RESTARTS   AGE
# web-app-7d4b9c8f6b-abc12   1/1     Running   0          1m
# web-app-7d4b9c8f6b-def34   1/1     Running   0          1m
# web-app-7d4b9c8f6b-ghi56   1/1     Running   0          1m

# Check service
kubectl get service web-service
# NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# web-service   LoadBalancer   10.43.123.45    localhost     80:30123/TCP   1m

# Test it
curl http://localhost
```

### Horizontal Pod Autoscaler (HPA)

Automatically scale based on CPU/memory usage.

**hpa.yaml:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # Scale when memory > 80%
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50  # Scale down by max 50% at a time
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100  # Can double pods at once
        periodSeconds: 15
      - type: Pods
        value: 4  # Or add 4 pods at once
        periodSeconds: 15
      selectPolicy: Max  # Use whichever scales faster
```

**Apply and test:**

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# Watch scaling in action
kubectl get hpa -w

# Generate load to trigger scaling
kubectl run load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://web-service; done"

# Watch pods scale up
kubectl get pods -w

# After load stops, watch pods scale down (after 5 min)
```

### Production-Ready Deployment

**app-deployment.yaml:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  REDIS_HOST: "redis.production.svc.cluster.local"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_PASSWORD: "your-secure-password"
  JWT_SECRET: "your-jwt-secret"
  API_KEY: "your-api-key"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  labels:
    app: api
    version: v1.2.3
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Add 1 extra pod during update
      maxUnavailable: 0  # Keep all pods running during update
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
        version: v1.2.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Anti-affinity: Don't put pods on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api
              topologyKey: kubernetes.io/hostname

      # Init container: Wait for database
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c', 'until nc -z postgres.production.svc.cluster.local 5432; do echo waiting for db; sleep 2; done']

      containers:
      - name: api
        image: myorg/api:v1.2.3
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        env:
        - name: PORT
          value: "8080"
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DATABASE_PASSWORD
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: JWT_SECRET

        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi

        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2

        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

        # Security
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL

        # Mount temporary storage
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache

      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}

      # Node selection (optional)
      nodeSelector:
        workload: api
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
  sessionAffinity: None
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
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

---

## Building Block 2: BullMQ (Job Queue)

BullMQ provides robust job queue functionality with Redis.

### Why Job Queues?

```
Without Queue:
┌──────────┐  Heavy task (30s)  ┌──────────┐
│  Client  │ ──────────────────>│   API    │
│          │                     │  (blocks │
│  Waits   │                     │  30 sec) │
│  30 sec  │ <──────────────────│          │
└──────────┘     Response       └──────────┘

Problems:
❌ Client timeout
❌ Server blocked
❌ No retries
❌ No priority
❌ Can't scale

With Queue:
┌──────────┐  Submit (50ms)  ┌──────────┐  Enqueue  ┌──────────┐
│  Client  │ ───────────────>│   API    │─────────>│  Queue   │
│          │                  │          │           └─────┬────┘
│          │ <───────────────│          │                 │
└──────────┘  "Job queued"   └──────────┘                 │
                                                           ▼
                                                    ┌──────────┐
                                                    │  Worker  │
                                                    │ (process │
                                                    │  in bg)  │
                                                    └──────────┘

Benefits:
✅ Instant response
✅ Scalable workers
✅ Automatic retries
✅ Priority queues
✅ Progress tracking
```

### BullMQ Setup

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  # API server (adds jobs to queue)
  api:
    build: ./api
    container_name: api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      - redis

  # Worker (processes jobs from queue)
  worker:
    build: ./worker
    container_name: worker
    restart: unless-stopped
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      - redis
    deploy:
      replicas: 3  # Run 3 workers

  # BullMQ Board (web UI)
  bullmq-board:
    image: deadly simple/bull-board:latest
    container_name: bullmq-board
    restart: unless-stopped
    ports:
      - "3001:3000"
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
    depends_on:
      - redis

volumes:
  redis_data:
```

### Producer (API Server)

**api/index.js:**

```javascript
import express from 'express';
import { Queue } from 'bullmq';

const app = express();
app.use(express.json());

// Create queue connection
const videoQueue = new Queue('video-processing', {
  connection: {
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379,
  },
});

const emailQueue = new Queue('email-sending', {
  connection: {
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379,
  },
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Submit video processing job
app.post('/api/videos/process', async (req, res) => {
  const { videoUrl, userId } = req.body;

  try {
    // Add job to queue
    const job = await videoQueue.add(
      'transcode',  // Job name
      {
        videoUrl,
        userId,
        formats: ['720p', '1080p', '4k'],
        timestamp: new Date().toISOString(),
      },
      {
        // Job options
        attempts: 3,  // Retry up to 3 times
        backoff: {
          type: 'exponential',
          delay: 5000,  // Start with 5 second delay
        },
        removeOnComplete: 100,  // Keep last 100 completed jobs
        removeOnFail: 200,      // Keep last 200 failed jobs
      }
    );

    res.json({
      jobId: job.id,
      status: 'queued',
      message: 'Video processing started',
    });
  } catch (error) {
    console.error('Failed to queue job:', error);
    res.status(500).json({ error: 'Failed to queue job' });
  }
});

// Check job status
app.get('/api/jobs/:jobId', async (req, res) => {
  const { jobId } = req.params;

  try {
    const job = await videoQueue.getJob(jobId);

    if (!job) {
      return res.status(404).json({ error: 'Job not found' });
    }

    const state = await job.getState();
    const progress = job.progress;

    res.json({
      jobId: job.id,
      state,
      progress,
      data: job.data,
      returnvalue: job.returnvalue,
      failedReason: job.failedReason,
      attemptsMade: job.attemptsMade,
    });
  } catch (error) {
    console.error('Failed to get job:', error);
    res.status(500).json({ error: 'Failed to get job status' });
  }
});

// Submit batch email job
app.post('/api/emails/batch', async (req, res) => {
  const { emails } = req.body;  // Array of emails

  try {
    // Add multiple jobs at once
    const jobs = emails.map((email, index) => ({
      name: 'send-email',
      data: {
        to: email.to,
        subject: email.subject,
        body: email.body,
      },
      opts: {
        priority: email.priority || 5,  // 1 (highest) to 10 (lowest)
        delay: index * 100,  // Stagger by 100ms
      },
    }));

    await emailQueue.addBulk(jobs);

    res.json({
      message: `${emails.length} emails queued`,
      count: emails.length,
    });
  } catch (error) {
    console.error('Failed to queue emails:', error);
    res.status(500).json({ error: 'Failed to queue emails' });
  }
});

// Scheduled job (run at specific time)
app.post('/api/reports/schedule', async (req, res) => {
  const { reportType, scheduledFor } = req.body;

  try {
    const delay = new Date(scheduledFor) - new Date();

    const job = await videoQueue.add(
      'generate-report',
      { reportType },
      {
        delay,  // Delay in milliseconds
        jobId: `report-${reportType}-${Date.now()}`,  // Custom job ID
      }
    );

    res.json({
      jobId: job.id,
      scheduledFor,
      message: 'Report scheduled',
    });
  } catch (error) {
    console.error('Failed to schedule report:', error);
    res.status(500).json({ error: 'Failed to schedule report' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`✅ API server listening on port ${PORT}`);
});
```

### Consumer (Worker)

**worker/index.js:**

```javascript
import { Worker } from 'bullmq';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

// Video processing worker
const videoWorker = new Worker(
  'video-processing',
  async (job) => {
    console.log(`🎬 Processing job ${job.id}: ${job.name}`);

    const { videoUrl, formats } = job.data;

    try {
      // Update progress: 0%
      await job.updateProgress(0);

      for (let i = 0; i < formats.length; i++) {
        const format = formats[i];
        console.log(`  📹 Transcoding to ${format}...`);

        // Simulate video processing
        // In production: use ffmpeg or video processing service
        await new Promise(resolve => setTimeout(resolve, 5000));

        // Update progress
        const progress = Math.round(((i + 1) / formats.length) * 100);
        await job.updateProgress(progress);

        console.log(`  ✅ ${format} complete (${progress}%)`);
      }

      // Return result
      return {
        videoUrl,
        outputs: formats.map(format => ({
          format,
          url: `https://cdn.example.com/${job.id}/${format}.mp4`,
        })),
        completedAt: new Date().toISOString(),
      };
    } catch (error) {
      console.error(`❌ Job ${job.id} failed:`, error);
      throw error;  // Job will be retried
    }
  },
  {
    connection: {
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
    },
    concurrency: 5,  // Process up to 5 jobs concurrently
    limiter: {
      max: 10,  // Max 10 jobs
      duration: 1000,  // Per second
    },
  }
);

// Email sending worker
const emailWorker = new Worker(
  'email-sending',
  async (job) => {
    console.log(`📧 Sending email ${job.id}`);

    const { to, subject, body } = job.data;

    try {
      // Simulate email sending
      // In production: use SendGrid, AWS SES, etc.
      await new Promise(resolve => setTimeout(resolve, 500));

      console.log(`✅ Email sent to ${to}`);

      return {
        to,
        sentAt: new Date().toISOString(),
        messageId: `msg-${job.id}`,
      };
    } catch (error) {
      console.error(`❌ Email ${job.id} failed:`, error);
      throw error;
    }
  },
  {
    connection: {
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
    },
    concurrency: 20,  // Send up to 20 emails concurrently
  }
);

// Event handlers
videoWorker.on('completed', (job) => {
  console.log(`✅ Video job ${job.id} completed`);
});

videoWorker.on('failed', (job, err) => {
  console.error(`❌ Video job ${job.id} failed:`, err.message);
});

emailWorker.on('completed', (job) => {
  console.log(`✅ Email job ${job.id} completed`);
});

emailWorker.on('failed', (job, err) => {
  console.error(`❌ Email job ${job.id} failed:`, err.message);
});

console.log('👷 Worker started, waiting for jobs...');
```

### Advanced: Rate Limiting and Priority

```javascript
// api/advanced-queue.js
import { Queue } from 'bullmq';

const queue = new Queue('api-calls', {
  connection: { host: 'redis', port: 6379 },
  defaultJobOptions: {
    // Global rate limiter
    limiter: {
      max: 100,  // 100 requests
      duration: 60000,  // Per minute
      groupKey: 'api',  // Apply limit to all jobs with this key
    },
  },
});

// Priority queue example
async function processHighPriorityUser(userId, task) {
  await queue.add(
    'user-task',
    { userId, task },
    {
      priority: 1,  // Highest priority (1-10, lower = higher priority)
      jobId: `high-${userId}-${Date.now()}`,
    }
  );
}

async function processNormalUser(userId, task) {
  await queue.add(
    'user-task',
    { userId, task },
    {
      priority: 5,  // Normal priority
    }
  );
}

async function processLowPriorityTask(task) {
  await queue.add(
    'background-task',
    { task },
    {
      priority: 10,  // Lowest priority
    }
  );
}

// Rate limit per user
async function addUserSpecificJob(userId, data) {
  await queue.add(
    'user-job',
    { userId, data },
    {
      limiter: {
        max: 10,  // 10 jobs
        duration: 60000,  // Per minute
        groupKey: `user-${userId}`,  // Per-user limit
      },
    }
  );
}
```

---

## Building Block 3: Temporal (Workflow Engine)

Temporal provides durable workflow execution with built-in retry, timeout, and state management.

### Why Temporal?

```
Complex Workflow Example: E-commerce Order

Without Temporal (fragile):
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Charge    │─>│   Reserve   │─>│    Ship     │
│   Card      │  │  Inventory  │  │   Order     │
└─────────────┘  └─────────────┘  └─────────────┘

Problems:
❌ What if payment succeeds but inventory fails?
❌ What if network fails mid-process?
❌ How to retry just the failed step?
❌ How to track progress?
❌ How to handle timeouts?

With Temporal (robust):
┌────────────────────────────────────────┐
│       Temporal Workflow                │
│                                        │
│  1. Charge card ✅                    │
│     → Failed? Retry 3x with backoff   │
│  2. Reserve inventory ✅              │
│     → Timeout after 30s               │
│  3. Ship order ✅                     │
│     → Track with activity ID          │
│  4. Send confirmation ✅              │
│                                        │
│  State persisted at each step!        │
│  Can survive crashes, restarts        │
└────────────────────────────────────────┘

Benefits:
✅ Automatic state persistence
✅ Built-in retry logic
✅ Timeout handling
✅ Can run for days/weeks
✅ Survives process restarts
✅ Visibility into execution
```

### Temporal Setup

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  postgresql:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: temporal
      POSTGRES_USER: temporal
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  temporal:
    image: temporalio/auto-setup:latest
    depends_on:
      - postgresql
    environment:
      DB: postgresql
      DB_PORT: 5432
      POSTGRES_USER: temporal
      POSTGRES_PWD: temporal
      POSTGRES_SEEDS: postgresql
    ports:
      - "7233:7233"
    volumes:
      - temporal_data:/etc/temporal

  temporal-ui:
    image: temporalio/ui:latest
    depends_on:
      - temporal
    environment:
      TEMPORAL_ADDRESS: temporal:7233
      TEMPORAL_CORS_ORIGINS: http://localhost:3000
    ports:
      - "8088:8080"

volumes:
  postgres_data:
  temporal_data:
```

### Temporal Workflow (TypeScript)

**workflows/order-processing.ts:**

```typescript
import { proxyActivities, sleep } from '@temporalio/workflow';
import type * as activities from '../activities';

// Create activity stubs with timeout and retry configuration
const {
  chargeCard,
  reserveInventory,
  shipOrder,
  sendConfirmation,
  refundCard,
  releaseInventory,
} = proxyActivities<typeof activities>({
  startToCloseTimeout: '5 minutes',
  retry: {
    initialInterval: '1 second',
    backoffCoefficient: 2,
    maximumInterval: '1 minute',
    maximumAttempts: 3,
  },
});

export interface OrderWorkflowInput {
  orderId: string;
  customerId: string;
  items: Array<{ sku: string; quantity: number }>;
  totalAmount: number;
  shippingAddress: string;
}

export async function orderProcessingWorkflow(
  input: OrderWorkflowInput
): Promise<string> {
  const { orderId, customerId, items, totalAmount, shippingAddress } = input;

  console.log(`🛒 Starting order workflow for ${orderId}`);

  let paymentId: string | undefined;
  let inventoryReservationId: string | undefined;

  try {
    // Step 1: Charge card
    console.log(`💳 Charging card for ${totalAmount}...`);
    paymentId = await chargeCard({
      customerId,
      amount: totalAmount,
      orderId,
    });
    console.log(`✅ Payment successful: ${paymentId}`);

    // Step 2: Reserve inventory
    console.log(`📦 Reserving inventory...`);
    inventoryReservationId = await reserveInventory({
      items,
      orderId,
    });
    console.log(`✅ Inventory reserved: ${inventoryReservationId}`);

    // Step 3: Ship order (with timeout)
    console.log(`🚚 Shipping order...`);
    const trackingNumber = await shipOrder({
      orderId,
      items,
      shippingAddress,
    });
    console.log(`✅ Order shipped: ${trackingNumber}`);

    // Step 4: Send confirmation email
    console.log(`📧 Sending confirmation...`);
    await sendConfirmation({
      customerId,
      orderId,
      trackingNumber,
    });
    console.log(`✅ Confirmation sent`);

    // Wait 24 hours for potential cancellation
    await sleep('24 hours');

    return `Order ${orderId} completed successfully`;
  } catch (error) {
    console.error(`❌ Order workflow failed:`, error);

    // Compensating transactions (rollback)
    if (inventoryReservationId) {
      console.log(`🔙 Releasing inventory...`);
      await releaseInventory({ reservationId: inventoryReservationId });
    }

    if (paymentId) {
      console.log(`💰 Refunding payment...`);
      await refundCard({ paymentId });
    }

    throw error;
  }
}

// Long-running workflow example
export async function subscriptionRenewalWorkflow(
  userId: string,
  planId: string
): Promise<void> {
  console.log(`📅 Starting subscription for user ${userId}`);

  while (true) {
    // Wait 30 days
    await sleep('30 days');

    console.log(`💳 Renewing subscription for ${userId}...`);

    try {
      const paymentId = await chargeCard({
        customerId: userId,
        amount: 29.99,
        orderId: `sub-${userId}-${Date.now()}`,
      });

      console.log(`✅ Subscription renewed: ${paymentId}`);

      await sendConfirmation({
        customerId: userId,
        orderId: paymentId,
        trackingNumber: '',
      });
    } catch (error) {
      console.error(`❌ Renewal failed:`, error);

      // Notify user of failed renewal
      // Give them 7 days to fix payment
      await sleep('7 days');

      // Retry once more
      try {
        await chargeCard({
          customerId: userId,
          amount: 29.99,
          orderId: `sub-retry-${userId}-${Date.now()}`,
        });
      } catch (retryError) {
        console.error(`❌ Retry failed, canceling subscription`);
        // Cancel subscription
        break;
      }
    }
  }

  console.log(`🛑 Subscription ended for ${userId}`);
}
```

**activities/order-activities.ts:**

```typescript
// Activities are the actual work that gets done
// They can fail, timeout, and will be retried by Temporal

export async function chargeCard(input: {
  customerId: string;
  amount: number;
  orderId: string;
}): Promise<string> {
  console.log(`Charging $${input.amount} to customer ${input.customerId}`);

  // Simulate API call to payment processor
  // In production: Stripe, PayPal, etc.
  await simulateDelay(1000);

  // Simulate occasional failures
  if (Math.random() < 0.1) {
    throw new Error('Payment gateway timeout');
  }

  const paymentId = `pay_${Date.now()}`;
  console.log(`Payment successful: ${paymentId}`);

  return paymentId;
}

export async function reserveInventory(input: {
  items: Array<{ sku: string; quantity: number }>;
  orderId: string;
}): Promise<string> {
  console.log(`Reserving inventory for order ${input.orderId}`);

  await simulateDelay(500);

  // Check if items are in stock
  for (const item of input.items) {
    // In production: query database
    const inStock = Math.random() > 0.05;
    if (!inStock) {
      throw new Error(`Item ${item.sku} out of stock`);
    }
  }

  const reservationId = `res_${Date.now()}`;
  console.log(`Inventory reserved: ${reservationId}`);

  return reservationId;
}

export async function shipOrder(input: {
  orderId: string;
  items: Array<{ sku: string; quantity: number }>;
  shippingAddress: string;
}): Promise<string> {
  console.log(`Shipping order ${input.orderId} to ${input.shippingAddress}`);

  await simulateDelay(2000);

  // In production: call shipping API
  const trackingNumber = `TRK${Date.now()}`;
  console.log(`Tracking number: ${trackingNumber}`);

  return trackingNumber;
}

export async function sendConfirmation(input: {
  customerId: string;
  orderId: string;
  trackingNumber: string;
}): Promise<void> {
  console.log(`Sending confirmation to customer ${input.customerId}`);

  await simulateDelay(500);

  // In production: send email via SendGrid, AWS SES, etc.
  console.log(`Email sent successfully`);
}

export async function refundCard(input: { paymentId: string }): Promise<void> {
  console.log(`Refunding payment ${input.paymentId}`);

  await simulateDelay(1000);

  // In production: call payment processor refund API
  console.log(`Refund completed`);
}

export async function releaseInventory(input: {
  reservationId: string;
}): Promise<void> {
  console.log(`Releasing inventory reservation ${input.reservationId}`);

  await simulateDelay(500);

  // In production: update database
  console.log(`Inventory released`);
}

function simulateDelay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

**worker.ts:**

```typescript
import { Worker } from '@temporalio/worker';
import * as activities from './activities';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./workflows'),
    activities,
    taskQueue: 'orders',
    maxConcurrentActivityTaskExecutions: 10,
    maxConcurrentWorkflowTaskExecutions: 100,
  });

  await worker.run();
}

run().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

**client.ts:**

```typescript
import { Connection, Client } from '@temporalio/client';
import { orderProcessingWorkflow } from './workflows/order-processing';

async function startOrderWorkflow() {
  const connection = await Connection.connect({
    address: 'localhost:7233',
  });

  const client = new Client({ connection });

  const handle = await client.workflow.start(orderProcessingWorkflow, {
    taskQueue: 'orders',
    workflowId: `order-${Date.now()}`,
    args: [
      {
        orderId: 'ORD-12345',
        customerId: 'USER-789',
        items: [
          { sku: 'WIDGET-1', quantity: 2 },
          { sku: 'GADGET-2', quantity: 1 },
        ],
        totalAmount: 99.99,
        shippingAddress: '123 Main St, City, State 12345',
      },
    ],
  });

  console.log(`Started workflow: ${handle.workflowId}`);

  // Wait for result
  const result = await handle.result();
  console.log(`Workflow completed: ${result}`);
}

startOrderWorkflow().catch(console.error);
```

---

## Auto-Scaling Strategies

### Metric-Based Scaling

Scale based on resource usage.

```yaml
# kubernetes/hpa-cpu-memory.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 50
  metrics:
  # Scale on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Scale on memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100  # Double pods
        periodSeconds: 15
      - type: Pods
        value: 4  # Or add 4 pods
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min
      policies:
      - type: Percent
        value: 50  # Remove half
        periodSeconds: 60
```

### Custom Metrics Scaling

Scale based on application metrics (request rate, queue depth, etc.).

```yaml
# kubernetes/hpa-custom-metrics.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 50
  metrics:
  # Scale based on requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"  # 1000 RPS per pod
  # Scale based on queue depth
  - type: External
    external:
      metric:
        name: redis_queue_depth
        selector:
          matchLabels:
            queue: "jobs"
      target:
        type: AverageValue
        averageValue: "30"  # 30 jobs per pod
```

### Scheduled Scaling

Scale based on time (handle predictable traffic patterns).

```yaml
# kubernetes/scheduled-scaling.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-morning
spec:
  schedule: "0 8 * * 1-5"  # 8 AM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: scaler
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment app --replicas=20
          restartPolicy: OnFailure
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-evening
spec:
  schedule: "0 18 * * 1-5"  # 6 PM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: scaler
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment app --replicas=5
          restartPolicy: OnFailure
```

---

## Cost Optimization

### Strategy 1: Spot Instances (AWS/GCP/Azure)

Use cheaper spot/preemptible instances for fault-tolerant workloads.

```yaml
# kubernetes/spot-node-pool.yaml
apiVersion: v1
kind: Node
metadata:
  labels:
    node.kubernetes.io/instance-type: spot
spec:
  taints:
  - key: spot
    value: "true"
    effect: NoSchedule
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-worker
spec:
  replicas: 10
  template:
    spec:
      # Tolerate spot instance taint
      tolerations:
      - key: spot
        operator: Equal
        value: "true"
        effect: NoSchedule

      # Prefer spot instances
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                - spot

      containers:
      - name: worker
        image: myorg/batch-worker:latest
        # Handle spot termination gracefully
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 30"]
```

### Strategy 2: Vertical Pod Autoscaler

Right-size pods automatically.

```yaml
# kubernetes/vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  updatePolicy:
    updateMode: "Auto"  # Automatically resize pods
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
      controlledResources:
      - cpu
      - memory
```

### Strategy 3: Cluster Autoscaler

Scale cluster nodes based on demand.

```bash
# AWS EKS example
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed \
  --asg-access

# Enable cluster autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

### Strategy 4: Resource Quotas

Prevent resource over-allocation.

```yaml
# kubernetes/resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"  # Max 100 CPU cores requested
    requests.memory: 200Gi  # Max 200 GB memory requested
    limits.cpu: "200"  # Max 200 CPU cores limit
    limits.memory: 400Gi  # Max 400 GB memory limit
    pods: "100"  # Max 100 pods
```

---

## Monitoring and Observability

### Prometheus Metrics

**app/metrics.go:**

```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    // Request metrics
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )

    // Job metrics
    jobsProcessed = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "jobs_processed_total",
            Help: "Total jobs processed",
        },
        []string{"queue", "status"},
    )

    jobProcessingDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "job_processing_duration_seconds",
            Help:    "Job processing duration",
            Buckets: prometheus.ExponentialBuckets(0.1, 2, 10),
        },
        []string{"queue", "job_type"},
    )

    // Queue metrics
    queueDepth = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "queue_depth",
            Help: "Current queue depth",
        },
        []string{"queue"},
    )

    // Resource metrics
    goroutines = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "goroutines_count",
            Help: "Number of goroutines",
        },
    )
)

func main() {
    // Expose metrics endpoint
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### Grafana Dashboard

**grafana/compute-dashboard.json:**

```json
{
  "dashboard": {
    "title": "Compute & Jobs Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{path}}"
          }
        ]
      },
      {
        "title": "Request Latency (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{method}} {{path}}"
          }
        ]
      },
      {
        "title": "Job Processing Rate",
        "targets": [
          {
            "expr": "rate(jobs_processed_total{status=\"success\"}[5m])",
            "legendFormat": "Success - {{queue}}"
          },
          {
            "expr": "rate(jobs_processed_total{status=\"failed\"}[5m])",
            "legendFormat": "Failed - {{queue}}"
          }
        ]
      },
      {
        "title": "Queue Depth",
        "targets": [
          {
            "expr": "queue_depth",
            "legendFormat": "{{queue}}"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [1000],
                "type": "gt"
              }
            }
          ]
        }
      },
      {
        "title": "Pod CPU Usage",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{pod=~\"app-.*\"}[5m])",
            "legendFormat": "{{pod}}"
          }
        ]
      },
      {
        "title": "Pod Memory Usage",
        "targets": [
          {
            "expr": "container_memory_working_set_bytes{pod=~\"app-.*\"}",
            "legendFormat": "{{pod}}"
          }
        ]
      }
    ]
  }
}
```

---

## Troubleshooting Guide

### Problem 1: Pods Crashing (CrashLoopBackOff)

**Symptoms:**
- Pods continuously restart
- Status shows "CrashLoopBackOff"
- Application unavailable

**Diagnosis:**

```bash
# Check pod status
kubectl get pods

# View pod events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container logs

# Check resource limits
kubectl top pod <pod-name>
```

**Common Causes & Solutions:**

1. **Application Error:**
```bash
# Fix: Check application logs
kubectl logs <pod-name>

# Deploy fixed version
kubectl set image deployment/app app=myorg/app:fixed
```

2. **Resource Limits Too Low:**
```yaml
# Increase limits
resources:
  limits:
    memory: 512Mi  # Was 256Mi
    cpu: 500m      # Was 200m
```

3. **Misconfigured Probes:**
```yaml
# Fix: Increase initial delay
livenessProbe:
  initialDelaySeconds: 60  # Was 10
  periodSeconds: 10
```

### Problem 2: Slow Job Processing

**Symptoms:**
- Jobs piling up in queue
- Processing takes too long
- Workers idle

**Diagnosis:**

```bash
# Check queue depth (BullMQ)
redis-cli LLEN bull:my-queue:wait

# Check worker status
kubectl get pods -l role=worker

# Check worker logs
kubectl logs -l role=worker --tail=100

# Check resource usage
kubectl top pods -l role=worker
```

**Solutions:**

1. **Scale Up Workers:**
```bash
kubectl scale deployment worker --replicas=10
```

2. **Increase Concurrency:**
```javascript
// worker.js
const worker = new Worker('my-queue', processJob, {
  concurrency: 10,  // Was 5
});
```

3. **Optimize Job Processing:**
```javascript
// Batch database operations
async function processJobs(jobs) {
  // Bad: Individual queries
  for (const job of jobs) {
    await db.insert(job.data);
  }

  // Good: Batch insert
  await db.insertMany(jobs.map(j => j.data));
}
```

### Problem 3: Out of Memory (OOM)

**Symptoms:**
- Pods killed with OOMKilled status
- Application slow before crash
- Memory usage growing unbounded

**Diagnosis:**

```bash
# Check if pod was OOM killed
kubectl describe pod <pod-name> | grep -i oom

# Check memory usage history
kubectl top pod <pod-name>

# Analyze heap dump (Node.js)
kubectl exec <pod-name> -- node --heapsnapshot-signal=SIGUSR2 &
kubectl exec <pod-name> -- kill -USR2 <pid>
```

**Solutions:**

1. **Memory Leak:**
```javascript
// Find and fix memory leak
// Common causes:
// - Global caches growing unbounded
// - Event listeners not removed
// - Unclosed database connections

// Fix: Add memory limits
const cache = new LRU({ max: 1000 });  // Limit cache size
```

2. **Increase Memory Limit:**
```yaml
resources:
  limits:
    memory: 2Gi  # Was 1Gi
```

3. **Enable Memory Profiling:**
```javascript
// Node.js: heap profiling
const v8 = require('v8');
const fs = require('fs');

setInterval(() => {
  const heapSnapshot = v8.writeHeapSnapshot();
  console.log('Heap snapshot written to', heapSnapshot);
}, 3600000);  // Every hour
```

### Problem 4: Auto-Scaling Not Working

**Symptoms:**
- HPA shows "unknown" for metrics
- Pods not scaling despite high load
- Scaling slower than expected

**Diagnosis:**

```bash
# Check HPA status
kubectl get hpa

# Check metrics server
kubectl top nodes
kubectl top pods

# View HPA events
kubectl describe hpa <hpa-name>

# Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system
```

**Solutions:**

1. **Install Metrics Server (if missing):**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

2. **Fix Resource Requests (required for HPA):**
```yaml
# HPA needs resource requests defined
resources:
  requests:  # Must be set!
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

3. **Adjust Scaling Thresholds:**
```yaml
# Lower threshold for faster scaling
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      averageUtilization: 50  # Was 70
```

---

## Best Practices

### DO ✅

1. **Set Resource Requests and Limits:**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

2. **Use Health Checks:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

3. **Implement Graceful Shutdown:**
```javascript
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');

  // Stop accepting new requests
  server.close();

  // Finish processing current jobs
  await worker.close();

  // Close database connections
  await db.close();

  process.exit(0);
});
```

4. **Use Anti-Affinity for High Availability:**
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: critical-app
      topologyKey: kubernetes.io/hostname
```

5. **Monitor Everything:**
```go
// Instrument your code
httpRequestsTotal.WithLabelValues(method, path, status).Inc()
httpRequestDuration.WithLabelValues(method, path).Observe(duration)
```

6. **Use Job Queues for Long Tasks:**
```javascript
// Don't block HTTP requests
app.post('/process', async (req, res) => {
  const job = await queue.add('process', req.body);
  res.json({ jobId: job.id });  // Return immediately
});
```

7. **Implement Retry Logic:**
```javascript
const job = await queue.add('task', data, {
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 1000,
  },
});
```

8. **Use Namespaces for Isolation:**
```bash
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development
```

9. **Version Your Container Images:**
```yaml
# Good: Specific version
image: myorg/app:v1.2.3

# Bad: Latest tag
image: myorg/app:latest
```

10. **Set Up Auto-Scaling:**
```yaml
# Both horizontal and vertical
- HorizontalPodAutoscaler (scale replicas)
- VerticalPodAutoscaler (resize pods)
- Cluster Autoscaler (add nodes)
```

### DON'T ❌

1. **Don't Run as Root:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

2. **Don't Use Latest Tag:**
```yaml
# Bad
image: nginx:latest

# Good
image: nginx:1.25.3
```

3. **Don't Forget Resource Limits:**
```yaml
# This can consume all node resources!
resources: {}  # ❌ No limits

# Set limits
resources:  # ✅
  limits:
    cpu: 500m
    memory: 512Mi
```

4. **Don't Block Event Loop (Node.js):**
```javascript
// Bad: CPU-intensive sync operation
app.get('/calculate', (req, res) => {
  const result = heavyComputation();  // Blocks!
  res.json({ result });
});

// Good: Use worker thread or job queue
app.get('/calculate', async (req, res) => {
  const job = await queue.add('calculate', req.body);
  res.json({ jobId: job.id });
});
```

5. **Don't Ignore Failed Jobs:**
```javascript
worker.on('failed', (job, err) => {
  // Bad: Ignore
  // Good: Log, alert, store in DLQ
  console.error(`Job ${job.id} failed:`, err);
  alerting.notify(`Job failure: ${err.message}`);
});
```

6. **Don't Deploy Without Testing:**
```bash
# Test deployment in staging first
kubectl apply -f deployment.yaml --namespace=staging

# Verify it works
curl https://staging.example.com/health

# Then deploy to production
kubectl apply -f deployment.yaml --namespace=production
```

7. **Don't Skip Monitoring:**
```yaml
# Always add Prometheus annotations
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

8. **Don't Use Synchronous Job Processing:**
```javascript
// Bad: Process job synchronously
app.post('/upload', async (req, res) => {
  await processVideo(req.file);  // Takes 10 minutes!
  res.json({ success: true });
});

// Good: Queue for background processing
app.post('/upload', async (req, res) => {
  const job = await queue.add('process-video', { file: req.file });
  res.json({ jobId: job.id, status: 'queued' });
});
```

9. **Don't Forget Timeouts:**
```javascript
// Set timeouts for all operations
const result = await doTask({ timeout: 30000 });  // 30 seconds
```

10. **Don't Over-Provision:**
```yaml
# Bad: Way more than needed
resources:
  requests:
    cpu: 8000m
    memory: 16Gi

# Good: Right-sized
resources:
  requests:
    cpu: 200m
    memory: 256Mi
```

---

## Further Reading

**Books:**
- "Kubernetes in Action" by Marko Lukša
- "Site Reliability Engineering" by Google
- "Designing Data-Intensive Applications" by Martin Kleppmann

**Documentation:**
- Kubernetes: https://kubernetes.io/docs/
- BullMQ: https://docs.bullmq.io/
- Temporal: https://docs.temporal.io/

**Online Resources:**
- CNCF Landscape: https://landscape.cncf.io/
- Kubernetes Patterns: https://k8spatterns.io/
- 12-Factor App: https://12factor.net/

---

## Summary

In this chapter, we built a scalable compute infrastructure:

1. **Kubernetes** for container orchestration and auto-scaling
2. **BullMQ** for reliable job queue processing
3. **Temporal** for durable workflow execution
4. **Auto-scaling strategies** (HPA, VPA, cluster autoscaler)
5. **Cost optimization** techniques
6. **Monitoring** with Prometheus and Grafana

**Key Takeaways:**
- Containers + orchestration enable massive scale
- Job queues decouple work from requests
- Workflows provide durability and retry logic
- Auto-scaling adapts to demand automatically
- Right-sizing resources saves money
- Monitor everything for visibility

**Next Chapter Preview:**

Chapter 9 explores **The Skeletal System (Infrastructure as Code)** - how to define and manage your entire infrastructure with code, enabling reproducibility, version control, and automation.

Ready to build infrastructure that's as easy to manage as writing code? Let's go! 🦴🚀

---

## Apex Compute Practices

Achieve hyperscale-grade muscle memory with these patterns:

- **Kubernetes Foundation** — Run managed control planes (EKS/GKE/AKS) or upstream clusters with Cluster API, and let Karpenter or Cluster Autoscaler match compute supply to demand within seconds.
- **Event-Driven Scaling** — Combine KEDA for workload-based triggers, Argo Workflows for batch DAGs, and Temporal/Cadence for stateful, durable workflows that survive process restarts.
- **Specialized Workloads** — Use NVIDIA GPU Operator, Intel Device Plugins, or Habana SynapseAI for accelerators; schedule ML and batch pipelines with Kubeflow, Ray, or Volcano for gang-scheduling awareness.
- **Edge & Hybrid Execution** — Extend workloads to the edge with k3s or MicroK8s plus Fleet or OpenYurt, and burst into the cloud via Virtual Kubelet or Azure Container Apps when local resources peak.
- **Performance & Cost Feedback** — Instrument with Kepler/eBPF for energy telemetry, feed data into Cast AI or StormForge for rightsizing, and enforce budgets using AWS Compute Optimizer or GCP Recommender integrations.
- **Operational Safety Nets** — Layer on Kyverno/OPA policies for resource quotas, use PriorityClasses + PodDisruptionBudgets to protect critical services, and rehearse chaos via Litmus or ChaosMesh to verify resilience.

This stack delivers elastic, efficient compute that flexes automatically as your organism responds to demand.

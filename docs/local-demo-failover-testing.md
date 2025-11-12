# Local Demo: Testing K8GB Failover

This guide demonstrates how to test K8GB's failover capabilities in your local k3d environment.

## Prerequisites

You should have already run:
```bash
K8GB_LOCAL_VERSION=test make deploy-full-local-setup
```

This creates two k3d clusters (`k3d-test-gslb1` and `k3d-test-gslb2`) with K8GB from your local development branch and test applications deployed.

**Important:** The `K8GB_LOCAL_VERSION=test` variable ensures your local code changes are deployed, not the released version.

### Start Continuous Monitoring

In a separate terminal tab, run:
```bash
make demo
```

**What to observe:**
- The demo script will start showing active tag responses (e.g., `200`)
- Continuous log output like:
  ```
  [Wed Nov 12 05:26:51 UTC 2025] ...
    "message": "eu",
  ```
- The `"message"` field indicates which cluster is actively serving traffic:
  - `"eu"` = cluster 1 (k3d-test-gslb1)
  - `"us"` = cluster 2 (k3d-test-gslb2)
- Keep this running throughout the demo to monitor failover behavior in real-time

## Demo Flow

### 1. Verify Initial Setup

Check the test Gslb resources and ingresses:

```bash
# View ingress resources in the test-gslb namespace
kubectl -n test-gslb get ing

# View Gslb resources
kubectl -n test-gslb get gslb

# Inspect the failover-ingress Gslb configuration
kubectl -n test-gslb get gslb failover-ingress -o yaml
```

**What to observe:**
- The `failover-ingress` Gslb resource should be present
- Status should show healthy endpoints from both clusters
- The `strategy` field should be set to `failover`
- The `make demo` terminal should show `200` responses with `"message": "eu"` (primary cluster)

### 2. Check Running Deployments

View the deployments in the primary cluster:

```bash
kubectl -n test-gslb get deploy --context=k3d-test-gslb1
```

**What to observe:**
- `frontend-podinfo` deployment should be running with 1 replica
- All pods should be in Ready state

### 3. Simulate Primary Cluster Failure

Scale down the frontend application in cluster 1 to simulate a failure:

```bash
kubectl -n test-gslb scale deploy/frontend-podinfo --replicas=0 --context=k3d-test-gslb1
```

**What happens:**
- The deployment scales to 0 replicas
- K8GB health checks detect the unhealthy endpoint
- Traffic should failover to cluster 2
- **Watch the `make demo` output**: the message should change from `"eu"` to `"us"`

### 4. Verify Failover Behavior

Check the Gslb status after scaling down:

```bash
# Check from default context
kubectl -n test-gslb get gslb failover-ingress -o yaml

# Check from cluster 1 context
kubectl -n test-gslb get gslb failover-ingress -o yaml --context=k3d-test-gslb1
```

**What to observe:**
- The `status.healthyRecords` should reflect only cluster 2 endpoints
- The `status.serviceHealth` should show cluster 1 as unhealthy
- DNS records should point to cluster 2
- The `make demo` output should show `"message": "us"` (cluster 2 is now active)

### 5. Restore Primary Cluster

Scale the deployment back up to restore normal operations:

```bash
kubectl -n test-gslb scale deploy/frontend-podinfo --replicas=1 --context=k3d-test-gslb1
```

**What happens:**
- The deployment scales back to 1 replica
- K8GB health checks detect the restored endpoint
- Traffic fails back to cluster 1 (primary)
- **Watch the `make demo` output**: the message should change back from `"us"` to `"eu"`

### 6. Verify Recovery

Check the Gslb status again:

```bash
kubectl -n test-gslb get gslb failover-ingress -o yaml
```

**What to observe:**
- Both clusters should show as healthy again
- DNS records should return to normal failover configuration
- The primary cluster should be serving traffic
- The `make demo` output should show `"message": "eu"` (cluster 1 active again)

## Understanding K8GB Failover

The failover strategy in K8GB works as follows:

1. **Health Monitoring**: K8GB continuously monitors endpoint health via Kubernetes readiness probes
2. **Primary/Secondary**: In failover mode, one cluster acts as primary, others as secondary
3. **Automatic Failover**: When primary becomes unhealthy, traffic automatically routes to secondary
4. **Automatic Failback**: When primary recovers, traffic returns to primary cluster

## Troubleshooting

### View K8GB Controller Logs

```bash
kubectl -n k8gb logs -l app.kubernetes.io/name=k8gb --context=k3d-test-gslb1
```

### Check CoreDNS Status

```bash
kubectl -n k8gb get pods -l app.kubernetes.io/name=coredns --context=k3d-test-gslb1
```

### Verify External DNS Records

```bash
kubectl -n k8gb get dnsendpoint --context=k3d-test-gslb1
```

## Next Steps

- Try the round-robin strategy by modifying the Gslb spec
- Add custom health check annotations to test different failure scenarios
- Monitor metrics in the deployed Prometheus instance
- Experiment with the geoip strategy (requires additional configuration)

For more information, see the [K8GB documentation](https://www.k8gb.io/docs/).

# ContribFest Exercise: Improving K8GB Observability with Logging

## Overview

This exercise guides you through improving observability in K8GB by adding structured debug logging to the Ingress server resolution process. You'll learn how to add meaningful logs that help operators troubleshoot issues in production.

**Time Required:** 30-45 minutes

**Skills Learned:**
- Understanding structured logging with zerolog
- Reading and modifying Kubernetes controllers
- Running unit tests
- Testing changes locally with multi-cluster k3d setup

## Background

### What is K8GB?

K8GB (Kubernetes Global Balancer) distributes traffic across geographically dispersed Kubernetes clusters. When a Gslb (Global Service Load Balancer) resource is created, K8GB:

1. Resolves referenced Ingress resources
2. Extracts server configurations (hosts and services)
3. Creates DNS records for global load balancing

### Why Logging Matters

When debugging issues in production, operators need visibility into:
- How many Ingress rules were found
- Which hosts and services were discovered
- Whether any rules were skipped and why

Currently, the `GetServers()` method in `controllers/refresolver/ingress/ingress.go` only logs warnings for malformed services but doesn't log successful operations. This makes troubleshooting difficult.

## The Problem

### Current State

The `GetServers()` method (lines 151-176 in `controllers/refresolver/ingress/ingress.go`) processes Ingress rules to extract server configurations:

```go
func (rr *ReferenceResolver) GetServers() ([]*k8gbv1beta1.Server, error) {
	servers := []*k8gbv1beta1.Server{}

	for _, rule := range rr.ingress.Spec.Rules {
		server := &k8gbv1beta1.Server{
			Host:     rule.Host,
			Services: []*k8gbv1beta1.NamespacedName{},
		}
		for _, path := range rule.HTTP.Paths {
			if path.Backend.Service == nil || path.Backend.Service.Name == "" {
				log.Warn().
					Str("ingress", rr.ingress.Name).
					Msg("Malformed service definition")
				continue
			}

			server.Services = append(server.Services, &k8gbv1beta1.NamespacedName{
				Name:      path.Backend.Service.Name,
				Namespace: rr.ingress.Namespace,
			})
		}
		servers = append(servers, server)
	}

	return servers, nil
}
```

**Issues:**
- No logging when the method starts (hard to track method execution)
- No logging for successfully processed servers
- No summary of results (how many servers/services found)
- Makes debugging production issues difficult

### Desired State

We want to add structured debug logging that provides visibility into:
1. **Entry point**: Log when starting to process Ingress rules
2. **Progress**: Log each server/host as it's discovered
3. **Summary**: Log the final count of servers and services found

## The Solution

### Step 1: Understand the File Structure

Navigate to the file we'll be modifying:

```bash
cd /path/to/k8gb
cat controllers/refresolver/ingress/ingress.go | head -40
```

Notice:
- Line 37: `var log = logging.Logger()` - The logger is already available
- The file uses [zerolog](https://github.com/rs/zerolog) for structured logging
- Existing logs use methods like `.Info()`, `.Warn()`, `.Debug()`, `.Err()`

### Step 2: Review Zerolog Basics

K8GB uses zerolog, a fast structured logger. Key patterns:

```go
// Info level - important operational messages
log.Info().
    Str("key", "value").
    Int("count", 42).
    Msg("Something happened")

// Debug level - detailed troubleshooting information
log.Debug().
    Str("ingress", ing.Name).
    Int("rules", len(rules)).
    Msg("Processing ingress rules")

// Warn level - problems that don't stop operation
log.Warn().
    Str("host", host).
    Msg("No services found for host")
```

**Important:** Debug logs are only shown when `LOG_LEVEL=debug` environment variable is set.

### Step 3: Add Debug Logging

Open `controllers/refresolver/ingress/ingress.go` and modify the `GetServers()` method:

**Add logging at the start** (after line 152):

```go
func (rr *ReferenceResolver) GetServers() ([]*k8gbv1beta1.Server, error) {
	servers := []*k8gbv1beta1.Server{}

	// Add this block:
	log.Debug().
		Str("ingress", rr.ingress.Name).
		Str("namespace", rr.ingress.Namespace).
		Int("rules_count", len(rr.ingress.Spec.Rules)).
		Msg("Starting server resolution from ingress rules")

	for _, rule := range rr.ingress.Spec.Rules {
		// ... rest of the code
```

**Add logging for each server** (after line 158, inside the loop):

```go
	for _, rule := range rr.ingress.Spec.Rules {
		server := &k8gbv1beta1.Server{
			Host:     rule.Host,
			Services: []*k8gbv1beta1.NamespacedName{},
		}

		// Add this block:
		log.Debug().
			Str("host", rule.Host).
			Str("ingress", rr.ingress.Name).
			Msg("Processing ingress rule for host")

		for _, path := range rule.HTTP.Paths {
			// ... rest of the loop
```

**Add logging when service is successfully added** (after line 170, after the append):

```go
			server.Services = append(server.Services, &k8gbv1beta1.NamespacedName{
				Name:      path.Backend.Service.Name,
				Namespace: rr.ingress.Namespace,
			})

			// Add this block:
			log.Debug().
				Str("service", path.Backend.Service.Name).
				Str("namespace", rr.ingress.Namespace).
				Str("host", rule.Host).
				Msg("Added service to server configuration")
		}
		servers = append(servers, server)
	}
```

**Add summary logging at the end** (before the return statement):

```go
		servers = append(servers, server)
	}

	// Add this block:
	totalServices := 0
	for _, srv := range servers {
		totalServices += len(srv.Services)
	}
	log.Info().
		Str("ingress", rr.ingress.Name).
		Int("servers_count", len(servers)).
		Int("services_count", totalServices).
		Msg("Completed server resolution from ingress")

	return servers, nil
}
```

### Complete Modified Method

Here's what the complete method should look like:

```go
func (rr *ReferenceResolver) GetServers() ([]*k8gbv1beta1.Server, error) {
	servers := []*k8gbv1beta1.Server{}

	log.Debug().
		Str("ingress", rr.ingress.Name).
		Str("namespace", rr.ingress.Namespace).
		Int("rules_count", len(rr.ingress.Spec.Rules)).
		Msg("Starting server resolution from ingress rules")

	for _, rule := range rr.ingress.Spec.Rules {
		server := &k8gbv1beta1.Server{
			Host:     rule.Host,
			Services: []*k8gbv1beta1.NamespacedName{},
		}

		log.Debug().
			Str("host", rule.Host).
			Str("ingress", rr.ingress.Name).
			Msg("Processing ingress rule for host")

		for _, path := range rule.HTTP.Paths {
			if path.Backend.Service == nil || path.Backend.Service.Name == "" {
				log.Warn().
					Str("ingress", rr.ingress.Name).
					Msg("Malformed service definition")
				continue
			}

			server.Services = append(server.Services, &k8gbv1beta1.NamespacedName{
				Name:      path.Backend.Service.Name,
				Namespace: rr.ingress.Namespace,
			})

			log.Debug().
				Str("service", path.Backend.Service.Name).
				Str("namespace", rr.ingress.Namespace).
				Str("host", rule.Host).
				Msg("Added service to server configuration")
		}
		servers = append(servers, server)
	}

	totalServices := 0
	for _, srv := range servers {
		totalServices += len(srv.Services)
	}
	log.Info().
		Str("ingress", rr.ingress.Name).
		Int("servers_count", len(servers)).
		Int("services_count", totalServices).
		Msg("Completed server resolution from ingress")

	return servers, nil
}
```

## Testing Your Changes

### Step 1: Run Unit Tests

First, ensure your changes don't break existing functionality:

```bash
# Run all tests
make test

# Run only the ingress refresolver tests
go test ./controllers/refresolver/ingress/... -v

# Run a specific test
go test ./controllers/refresolver/ingress/... -v -run TestGetServers
```

Expected output:
```
=== RUN   TestGetServers
=== RUN   TestGetServers/single_server
=== RUN   TestGetServers/multiple_servers
--- PASS: TestGetServers (0.00s)
    --- PASS: TestGetServers/single_server (0.00s)
    --- PASS: TestGetServers/multiple_servers (0.00s)
PASS
ok      github.com/k8gb-io/k8gb/controllers/refresolver/ingress
```

### Step 2: Test Locally with k3d

Now let's see the logs in action with a real multi-cluster setup!

**Deploy the full local environment:**

```bash
# This creates 2 k3d clusters with k8gb installed
make deploy-full-local-setup
```

This will:
- Create 2 k3d clusters (cluster-1, cluster-2)
- Deploy k8gb operator to both
- Deploy test applications (podinfo)
- Configure DNS and load balancing

**Already have the local setup running?**

If you previously ran `make deploy-full-local-setup` and just want to update k8gb with your logging changes:

```bash
# Quick upgrade - builds new images and upgrades all clusters
make upgrade-candidate
```

This will:
- Build a new docker image with your logging changes
- Import the image to both k3d clusters
- Run `helm upgrade` on both clusters
- Keep your existing test apps and configuration

**Watch the operator logs:**

```bash
# In one terminal, watch cluster-1 logs
kubectl logs -n k8gb -l app.kubernetes.io/name=k8gb -f --context=k3d-test-gslb1

# In another terminal, watch cluster-2 logs
kubectl logs -n k8gb -l app.kubernetes.io/name=k8gb -f --context=k3d-test-gslb2
```

**Check existing Gslb resources:**

The local setup already has several Gslb resources running. Let's see them:

```bash
kubectl get gslb -A
```

You should see:
```
NAMESPACE         NAME                 STRATEGY     GEOTAG
test-gslb-istio   failover-istio       failover     us
test-gslb-istio   roundrobin-istio     roundRobin   us
test-gslb         failover-ingress     failover     us
test-gslb         roundrobin-ingress   roundRobin   us
```

**Look for your new log messages:**

The operator continuously reconciles these Gslbs. In the logs you're watching, you should now see your logging improvements:

```json
{"level":"debug","ingress":"roundrobin-test-gslb","namespace":"test-gslb","rules_count":3,"time":"2025-11-11T04:00:00Z","message":"Starting server resolution from ingress rules"}
{"level":"debug","host":"roundrobin.cloud.example.com","ingress":"roundrobin-test-gslb","time":"2025-11-11T04:00:00Z","message":"Processing ingress rule for host"}
{"level":"debug","service":"frontend-podinfo","namespace":"test-gslb","host":"roundrobin.cloud.example.com","time":"2025-11-11T04:00:00Z","message":"Added service to server configuration"}
{"level":"info","ingress":"roundrobin-test-gslb","servers_count":3,"services_count":3,"time":"2025-11-11T04:00:00Z","message":"Completed server resolution from ingress"}
```

Notice the `servers_count: 3` because roundrobin-ingress has multiple hosts (roundrobin, notfound, unhealthy)!

**Enable debug logging:**

If you don't see debug logs, the log level needs to be changed:

```bash
# Update the k8gb deployment to enable debug logging
kubectl set env deployment/k8gb -n k8gb LOG_LEVEL=debug --context=k3d-test-gslb1
kubectl set env deployment/k8gb -n k8gb LOG_LEVEL=debug --context=k3d-test-gslb2

# Wait for pods to restart
kubectl rollout status deployment/k8gb -n k8gb --context=k3d-test-gslb1
```

### Step 3: Test with Different Scenarios

**Single server scenario:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single-host
  namespace: test-gslb
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

Expected log output:
- `rules_count: 1`
- `servers_count: 1`
- `services_count: 1`

**Multiple servers scenario:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host
  namespace: test-gslb
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: service3
            port:
              number: 8080
```

Expected log output:
- `rules_count: 2`
- `servers_count: 2`
- `services_count: 3`

### Step 4: Clean Up

When done testing:

```bash
# Destroy the local clusters
make destroy-full-local-setup
```

## Hands-On Exercises

### Exercise 1: Analyze Log Output

**Goal:** Understand how logging helps with troubleshooting

1. Deploy the local setup: `make deploy-full-local-setup`
2. Watch the logs in JSON format
3. Create a Gslb resource
4. Answer these questions:
   - How many ingress rules were processed?
   - What hosts were discovered?
   - How many services are configured per host?
   - How long did the reconciliation take? (check timestamps)

### Exercise 2: Add Error Logging

**Goal:** Improve error visibility

Currently, when an Ingress rule has no paths, we don't log anything. Add a warning:

```go
for _, rule := range rr.ingress.Spec.Rules {
	// Add this check:
	if rule.HTTP == nil || len(rule.HTTP.Paths) == 0 {
		log.Warn().
			Str("host", rule.Host).
			Str("ingress", rr.ingress.Name).
			Msg("Ingress rule has no HTTP paths defined")
		continue
	}

	// ... rest of the code
}
```

Test it by creating an Ingress with an empty rule.

### Exercise 3: Add Metrics

**Goal:** Combine logging with metrics

K8GB uses Prometheus for metrics. Add a metric counter for processed servers:

1. Look at `controllers/providers/metrics/prometheus.go` to understand the metrics system
2. Add a new counter for `ingress_servers_processed_total`
3. Increment it in the `GetServers()` method
4. Test that the metric appears in Prometheus

### Exercise 4: Log Context Propagation

**Goal:** Add request tracing

Modify the method signature to accept a context:

```go
func (rr *ReferenceResolver) GetServers(ctx context.Context) ([]*k8gbv1beta1.Server, error) {
```

Use the context to:
1. Extract trace IDs from OpenTelemetry
2. Add them to log messages
3. Enable distributed tracing across clusters

Hint: Look at `controllers/tracing/tracing.go` for examples.

## Key Takeaways

1. **Structured Logging:** Use key-value pairs (`.Str()`, `.Int()`) instead of formatted strings for easier parsing and filtering

2. **Log Levels:**
   - `Debug`: Detailed information for troubleshooting (disabled by default)
   - `Info`: Important operational events (always shown)
   - `Warn`: Problems that don't stop operation
   - `Error`: Errors that cause failures

3. **Context Matters:** Include relevant context in every log (ingress name, namespace, host, etc.)

4. **Performance:** Zerolog is fast, but still be mindful in tight loops

5. **Testing:** Always verify logs work with both unit tests and e2e testing

6. **Production Value:** Good logging is crucial for operating distributed systems

## Common Pitfalls and Solutions

### Pitfall 1: Too Much Logging

**Problem:** Logging every iteration in a tight loop creates noise

```go
// BAD: Logs inside service loop
for i, svc := range services {
	log.Debug().Int("index", i).Msg("Processing service")
	// ... process service
}
```

**Solution:** Log summaries instead

```go
// GOOD: Log summary after loop
log.Debug().Int("processed", len(services)).Msg("Processed all services")
```

### Pitfall 2: Missing Context

**Problem:** Logs without enough context are hard to correlate

```go
// BAD: No context about which resource
log.Info().Msg("Found 3 servers")
```

**Solution:** Always include identifying information

```go
// GOOD: Clear context
log.Info().
	Str("ingress", ing.Name).
	Str("namespace", ing.Namespace).
	Int("servers_count", 3).
	Msg("Completed server resolution")
```

### Pitfall 3: Using String Formatting

**Problem:** Formatted strings lose structure

```go
// BAD: Unstructured log
log.Info().Msg(fmt.Sprintf("Found %d servers in %s", count, name))
```

**Solution:** Use structured fields

```go
// GOOD: Structured and parseable
log.Info().
	Int("servers_count", count).
	Str("ingress", name).
	Msg("Found servers")
```

### Pitfall 4: Not Testing Logs

**Problem:** Logs aren't visible during local testing

**Solution:** Set environment variable

```bash
# Enable debug logs
export LOG_LEVEL=debug

# Run tests
go test ./controllers/refresolver/ingress/... -v
```

## Additional Resources

- [Zerolog Documentation](https://github.com/rs/zerolog)
- [K8GB Architecture](https://www.k8gb.io/docs/architecture.html)
- [K8GB Local Development](https://www.k8gb.io/docs/local.html)
- [Structured Logging Best Practices](https://www.honeycomb.io/blog/structured-logging-and-your-team)
- [K8GB Contributing Guide](../CONTRIBUTING.md)

## Next Steps

After completing this exercise:

1. Look for other methods in K8GB that could benefit from better logging
2. Consider adding logging to error paths in other controllers
3. Explore adding distributed tracing with OpenTelemetry
4. Submit a pull request with your improvements!

## Bonus Challenge: Create a PR

If you've successfully completed the exercise:

1. Create a branch: `git checkout -b improve-ingress-resolver-logging`
2. Commit your changes with a clear message:
   ```bash
   git add controllers/refresolver/ingress/ingress.go
   git commit -m "feat: add debug logging to ingress server resolution

   Adds structured logging to GetServers() method to improve
   observability when troubleshooting ingress resolution issues.

   - Logs entry with ingress details and rule count
   - Logs each processed host
   - Logs each added service
   - Logs summary with total servers and services count

   This helps operators debug issues with ingress configuration
   and understand how K8GB is processing their resources."
   ```
3. Push and create a PR: `git push origin improve-ingress-resolver-logging`
4. In the PR description, explain:
   - What problem does this solve?
   - How did you test it?
   - Include example log output

## Questions for Discussion

1. When should you use Debug vs Info vs Warn log levels?
2. What information is most valuable in production logs?
3. How do structured logs help with observability tools like Grafana Loki?
4. What are the tradeoffs between verbose logging and performance?
5. How can logs help with debugging distributed system issues?
6. What other areas of K8GB would benefit from better logging?

---

**Need Help?** Ask in the K8GB Slack channel or open a discussion on GitHub!

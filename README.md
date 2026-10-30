# Day 20 - Health, Resources & Autoscaling

DevOps Winter Arc | Day 20/90 | Week 3: Kubernetes

## Objective

Make an application self-checking, resource-aware, and able to scale from
1 to 5 Pods under load, using probes, CPU/memory requests and limits,
metrics-server, and a HorizontalPodAutoscaler (HPA).

## Key concepts

| Concept | Meaning |
|---|---|
| Startup probe | Gives a slow-starting app time to initialize before other probes act |
| Readiness probe | Decides whether a Pod should receive traffic. Failure removes it from Service endpoints but does not restart it |
| Liveness probe | Detects a stuck container. Repeated failure makes the kubelet restart it |
| Request | Used for scheduling and as the base for CPU utilization calculations |
| Limit | Upper cap. Excess CPU is throttled, excess memory can cause OOMKilled |
| QoS class | Guaranteed, Burstable, or BestEffort, based on requests and limits |
| metrics-server | Collects CPU/memory metrics from kubelets for `kubectl top` and HPA |
| HPA | Adjusts replica count from metrics and a target, within min/max bounds |

## Files

- `deployment.yaml`: `php-apache` Deployment with resources and three probes
- `evidence/`: screenshots

## Setup and commands

### 1. Metrics

```bash
kubectl get nodes
kubectl top nodes
kubectl top pods
```

metrics-server installed: **[installed by me]**

If installed for kind:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl rollout status deployment/metrics-server -n kube-system
```

`--kubelet-insecure-tls` is needed on kind because the kubelet certificate is
not trusted by metrics-server. It is acceptable for a local lab only.

`kubectl top nodes` result: **[paste output]**



### 2. Resource-aware Deployment

```bash
kubectl create namespace day20-health
kubectl apply -f deployment.yaml
kubectl rollout status deploy/php-apache -n day20-health
kubectl describe pod -n day20-health -l app=php-apache
```

Settings in `deployment.yaml`:

| Setting | Value |
|---|---|
| CPU request / limit | 200m / 500m |
| Memory request / limit | 64Mi / 128Mi |
| Startup probe | GET `/` on 80, period 2s, failureThreshold 30 (up to about 60s) |
| Readiness probe | GET `/` on 80, initial delay 5s, period 5s |
| Liveness probe | GET `/` on 80, initial delay 15s, period 10s |

Evidence: `evidence/[describe-pod.png]`

### 3. Service for the load test

The load generator calls the Service DNS name, so the Deployment needs a Service:

```bash
kubectl expose deployment php-apache -n day20-health --port=80
kubectl get svc,endpointslices -n day20-health
```

### 4. QoS and resource usage

```bash
kubectl get pod -n day20-health -l app=php-apache -o jsonpath='{.items[0].status.qosClass}'; echo
kubectl top pods -n day20-health
```

- QoS class observed: **[Burstable]**
- CPU / memory observed at idle: **[101m / 11 mi]**

It is Burstable because requests are set but requests and limits are not equal
for every resource.



### 5. HPA

```bash
kubectl autoscale deployment php-apache -n day20-health \
  --cpu-percent=50 --min=1 --max=5
kubectl get hpa -n day20-health
kubectl describe hpa php-apache -n day20-health
```

| HPA setting | Value |
|---|---|
| Target | 50% CPU utilization, relative to the CPU request |
| Min replicas | 1 |
| Max replicas | 5 |

With a 200m request, 50% means HPA aims for about 100m CPU per Pod.



### 6. Load test and scaling

```bash
kubectl run load-generator -n day20-health \
  --image=busybox:1.36 --restart=Never -it --rm -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache.day20-health.svc.cluster.local; done"
```

In other terminals:

```bash
kubectl get hpa -n day20-health -w
kubectl get deployments,pods -n day20-health -w
kubectl top pods -n day20-health
```

Observations:

- CPU target before load: **[ 155%/50%]**
- CPU target under load: **[ 134% / 50% ]**
- Replicas over time: **[ 1 -> ... -> N, never above 5]**
- After stopping the load: **[describe whether and when replicas scaled down]**

Scaling is not instant. HPA reads metrics at intervals, and scale-down waits
for a stabilization window (about 5 minutes by default) to avoid flapping.


### 7. Safe probe failure test

I temporarily changed only the readiness probe path to `/nonexistent`:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n day20-health -w
kubectl describe pod -n day20-health -l app=php-apache
kubectl get events -n day20-health --sort-by=.lastTimestamp
```

- Pod status observed: **[e.g. Running but READY 0/1]**
- Event message: **[paste the Readiness probe failed line]**
- RESTARTS column: **[did it change?]**

Then I restored the path to `/` and confirmed the Pods became Ready again.
The liveness probe was left unchanged to avoid repeated restarts.

Evidence: `evidence/[probe-failure.png]`

What each probe does:

- **Readiness failure** affects traffic eligibility. The Pod is removed from
  Service endpoints but is not restarted.
- **Liveness failure** (repeated) makes the kubelet restart the container.
- **Startup probes** protect slow-starting apps by holding off the other
  probes until the app has started.

## Troubleshooting / lessons learned

- **[Write one real problem you hit, the command that revealed it, and the fix. For example HPA showing `<unknown>`, `kubectl top` not ready yet, or a missing Service.]**
- HPA utilization is measured against CPU requests, so requests are required.
- Scaling and scale-down take time. Watch with `-w` and be patient.
- Do not leave a broken liveness probe running, because it causes restarts.

| Symptom | What to check |
|---|---|
| `kubectl top` fails | metrics-server Pods, logs, node metrics |
| HPA shows unknown | metrics-server running, CPU requests set, wait for metrics |
| Replicas do not increase | HPA current/target, load, events, maxReplicas |
| Pod keeps restarting | Events, Last State, probe path/port, logs |
| OOMKilled | Memory usage and limits |

## Interview questions

**Q1. Readiness vs liveness?**
Readiness decides if a Pod receives traffic. Liveness detects a stuck
container and can trigger a restart.

**Q2. Why use a startup probe?**
It gives a slow-starting app time to initialize before liveness and readiness
checks act.

**Q3. What happens when readiness fails?**
The Pod becomes not ready and is removed from Service endpoints. Readiness
failure alone does not restart it.

**Q4. What happens when liveness repeatedly fails?**
The kubelet restarts the container according to the probe and restart policy.

**Q5. CPU request vs limit?**
The request is used for scheduling and utilization calculations. The limit
caps CPU, and excess use is throttled.

**Q6. What can memory limit exceedance cause?**
The container can be terminated with OOMKilled. Memory limits should reflect
measured needs.

**Q7. Why does HPA need CPU requests?**
CPU utilization is measured relative to requested CPU. Without requests,
utilization may be undefined.

**Q8. HPA vs manual scaling?**
Manual scaling sets replicas explicitly. HPA adjusts them continuously from
metrics within configured bounds.

**Q9. What does metrics-server do?**
It collects CPU and memory metrics from kubelets and exposes them to tools
such as `kubectl top` and HPA.

**Q10. Explain QoS classes.**
Guaranteed: requests equal limits for CPU and memory in every container.
Burstable: some requests or limits are set but the Guaranteed rules are not
met. BestEffort: none are set.

**Q11. At 100% CPU, target 50%, with 2 replicas, what is the rough desired count?**
2 x 100 / 50 = 4, before min/max bounds and other controller behavior.

**Q12. Why might HPA not scale an app that feels slow?**
CPU may not be high, metrics may be missing, requests may be absent, max
replicas may be reached, or the bottleneck may be elsewhere.

**Q13. Should liveness fail whenever a database dependency is unreachable?**
Usually not by default. Transient dependency failures can cause unnecessary
restarts. Design checks around the app's actual failure modes.

**Q14. Does autoscaling guarantee better performance?**
No. Node capacity, memory, downstream services, or non-CPU bottlenecks can
still constrain performance.

## Security notes

- No kubeconfig files, tokens, or credentials are committed.
- `--kubelet-insecure-tls` was used for a local kind lab only and should not
  be used on a production cluster.

## Cleanup

```bash
kubectl delete namespace day20-health
```

metrics-server was **[removed]**, since it may be used in
later Winter Arc days.

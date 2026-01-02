# Falco

Runtime security monitoring system that detects unexpected behavior in your Kubernetes cluster.

## What is Falco?

Falco is a security watchdog for your cluster. It monitors system calls and Kubernetes events in real-time, alerting you when something suspicious happens - like a container trying to access sensitive files or spawn an unexpected shell.

## What it does

- **Runtime threat detection**: Monitors running containers for malicious activity
- **System call monitoring**: Watches kernel-level events
- **Kubernetes audit**: Monitors K8s API server events
- **Anomaly detection**: Detects unusual behavior patterns
- **Alerting**: Sends notifications when threats are detected
- **Compliance**: Helps meet security compliance requirements

## Why you need it

Kubernetes can run untrusted or compromised containers. Falco detects:
- ✅ Shells spawned in containers (potential compromise)
- ✅ Sensitive file access (/etc/passwd, SSH keys)
- ✅ Privilege escalation attempts
- ✅ Unexpected network connections
- ✅ Container escape attempts
- ✅ Crypto mining activity
- ✅ Suspicious K8s API calls

## How it works

1. Falco runs as a DaemonSet (one pod per node)
2. It loads a kernel module or uses eBPF to monitor system calls
3. It applies rules to detect suspicious patterns
4. When a rule matches, Falco generates an alert
5. Alerts are sent to configured outputs (logs, Slack, etc.)

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
driver:
  kind: ebpf  # or module
  
falco:
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/rules.d
    
  json_output: true
  log_level: info
  
  outputs:
    rate: 1
    max_burst: 1000
```

## Falco Rules

Rules define what to detect. Default rules cover common threats.

### Example Rules

**Shell in container**:
```yaml
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    container and 
    proc.name in (bash, sh, zsh)
  output: "Shell spawned in container (user=%user.name container=%container.name)"
  priority: WARNING
```

**Sensitive file access**:
```yaml
- rule: Read sensitive file
  desc: Attempt to read sensitive file
  condition: >
    open_read and
    fd.name in (/etc/shadow, /etc/passwd)
  output: "Sensitive file read (user=%user.name file=%fd.name)"
  priority: WARNING
```

**Privilege escalation**:
```yaml
- rule: Privilege escalation
  desc: Process attempted privilege escalation
  condition: >
    spawned_process and
    proc.is_setuid_setgid
  output: "Privilege escalation detected (user=%user.name process=%proc.name)"
  priority: CRITICAL
```

## Alert Priorities

- **EMERGENCY**: Critical security event, immediate action required
- **ALERT**: Serious security threat
- **CRITICAL**: Potential compromise
- **ERROR**: Suspicious activity
- **WARNING**: Unusual but may be legitimate
- **NOTICE**: Informational security event
- **INFORMATIONAL**: Normal security-relevant event
- **DEBUG**: Debug information

## Deployment

Deployed by ArgoCD as a DaemonSet (runs on all nodes).

Requires privileged access to monitor kernel events.

## Viewing Alerts

### Via Logs

```bash
kubectl logs -n sec -l app=falco
```

### Via Grafana

Configure Loki to collect Falco logs, then query in Grafana:

```logql
{namespace="sec", app="falco"} | json | priority="CRITICAL"
```

### Via External Systems

Configure outputs in values.yaml:
- **Slack**: Send to Slack channel
- **Webhook**: HTTP endpoint
- **gRPC**: Falco Sidekick for advanced routing

## Common Alerts

### Normal Operations (May trigger alerts)

Some normal operations can trigger alerts:
- **Package managers**: apt, yum, apk (modify system files)
- **Init containers**: May run shells during setup
- **Debugging**: kubectl exec creates shells

**Solution**: Create exceptions for known-good behavior.

### Real Threats

- **Unexpected shells**: Shells in production containers
- **Crypto mining**: High CPU + network to mining pools
- **Data exfiltration**: Unusual network connections
- **Container escapes**: Attempts to access host filesystem

## Custom Rules

Add custom rules for your environment:

```yaml
customRules:
  my-rules.yaml: |-
    - rule: Unauthorized API Access
      desc: Detect API access from non-approved services
      condition: >
        k8s_audit and
        ka.verb in (create, delete) and
        not ka.user.name in (system:serviceaccount:kube-system:*)
      output: "Unauthorized K8s API call (user=%ka.user.name verb=%ka.verb)"
      priority: WARNING
```

## Exceptions

Whitelist known-good behavior:

```yaml
- list: allowed_containers
  items: [prometheus, grafana]

- rule: Shell in container
  condition: >
    spawned_process and 
    container and 
    proc.name in (bash, sh) and
    not container.name in (allowed_containers)
```

## Monitoring Falco

Check Falco is running:
```bash
kubectl get pods -n sec -l app=falco
kubectl logs -n sec -l app=falco -f
```

Falco metrics (if enabled):
```promql
falco_events_total
falco_drops_total
```

## Performance Impact

Falco monitoring has minimal overhead:
- CPU: ~2-5% per node
- Memory: ~100-200MB per pod
- Network: Negligible

If performance is critical:
- Use eBPF instead of kernel module (lower overhead)
- Reduce rule complexity
- Filter out noisy rules

## Troubleshooting

**Falco pod failing**:
- Check kernel module/eBPF loading
- Verify privileged access
- Check kernel compatibility

**No alerts appearing**:
- Verify rules are loaded
- Check output configuration
- Trigger a test alert

**Too many alerts**:
- Add exceptions for known-good behavior
- Increase alert thresholds
- Review rule priorities

**High CPU usage**:
- Switch to eBPF driver
- Reduce rule complexity
- Disable noisy rules

## Integration with Other Tools

### Falco Sidekick
Routes Falco alerts to multiple destinations:
- Slack, Teams, Discord
- Elasticsearch, Loki
- S3, GCS
- AWS Lambda, Cloud Functions

### Grafana Dashboard
Create dashboards showing:
- Alert counts by priority
- Most triggered rules
- Affected containers/pods
- Alert trends over time

## Security Best Practices

- **Review alerts regularly**: Don't ignore Falco alerts
- **Tune rules**: Add exceptions for legitimate behavior
- **Automate responses**: Integrate with incident response
- **Test rules**: Verify rules catch real threats
- **Update regularly**: New rules for new threats
- **Least privilege**: Use Falco to enforce least privilege

## Testing Falco

Trigger test alerts:

```bash
# Spawn shell in container (should alert)
kubectl exec -it <pod-name> -- /bin/sh

# Read sensitive file (should alert)
kubectl exec <pod-name> -- cat /etc/shadow

# Check Falco logs for alerts
kubectl logs -n sec -l app=falco | grep CRITICAL
```

## Compliance

Falco helps with compliance:
- **PCI DSS**: File integrity monitoring, privilege escalation detection
- **HIPAA**: Audit logging, access control monitoring
- **SOC 2**: Security monitoring, anomaly detection
- **CIS Benchmarks**: Runtime security controls

## Resources

- Falco rules repo: https://github.com/falcosecurity/rules
- Documentation: https://falco.org/docs/
- Rules syntax: https://falco.org/docs/rules/
- Falco Sidekick: https://github.com/falcosecurity/falcosidekick

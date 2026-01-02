# Calibre Web Automated (CWA)

Web-based eBook library management and reader interface.

## Configuration

### Container Image

[crocodilestick/calibre-web-automated](https://hub.docker.com/r/crocodilestick/calibre-web-automated)

### Authentication

Credentials from Vault:
```yaml
user:
  username: <path:secret/data/media/calibre#user>
  password: <path:secret/data/media/calibre#password>
```

### Storage

NFS mount for Calibre library:
```yaml
nfs: <path:secret/data/infra#nfs>
```

## Resources

- **Node Selector**: `size: s` (small nodes)
- Other resources: Not explicitly limited

## Access

`https://cwa.yourdomain.com`

## References

- [Calibre-Web-Automated GitHub](https://github.com/crocodilestick/Calibre-Web-Automated)

## Troubleshooting

**Can't access CWA**: Check ingress, verify pod is running

**Books not appearing**: Verify library path, check file permissions

**Conversion failing**: Check Calibre backend logs, ensure conversion tools are installed

**Email not working**: Verify SMTP settings, check firewall/port blocking

**Slow performance**: Large libraries can slow down, consider database optimization

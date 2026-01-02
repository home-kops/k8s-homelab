# CWA Downloader

Automated eBook downloader with VPN sidecar for secure downloads.

## Configuration

### Container Images

**Main container**: [calibre-web-automated-book-downloader](https://github.com/calibrain/calibre-web-automated-book-downloader)

**VPN sidecar**: [qmcgaw/gluetun](https://github.com/qdm12/gluetun)

### VPN Sidecar

All download traffic routes through the Gluetun VPN container. Credentials from Vault:
```yaml
vpn:
  user: <path:secret/data/p2p/vpn#user>
  password: <path:secret/data/p2p/vpn#password>
```

Provides network isolation and kill switch if VPN drops.

### Storage

NFS mount for Calibre library:
```yaml
nfs: <path:secret/data/infra#nfs>
```

## Resources

- **Node Selector**: `size: s` (small nodes)
- Other resources: Not explicitly limited

## Access

**IngressRoute**: `https://cwa-downloader.{{ .Values.domain }}` (port 8084)

Domain from Vault: `<path:secret/data/infra#domain>`

## Dependencies

- **CWA**: Downloads books into library that CWA serves

## References

- [CWA Book Downloader GitHub](https://github.com/calibrain/calibre-web-automated-book-downloader)
- [Gluetun VPN Client](https://github.com/qdm12/gluetun)

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
deployment:
  image:
    repository: crocodilestick/cwa-downloader
    tag: latest
    
  env:
    - name: CWA_URL
      value: "http://cwa:8083"
    - name: DOWNLOAD_SCHEDULE
      value: "0 2 * * *"  # Daily at 2 AM
      
  volumes:
    - name: downloads
      persistentVolumeClaim:
        claimName: cwa-downloads
    - name: library
      persistentVolumeClaim:
        claimName: cwa-library  # Shared with CWA
```

### Environment Variables

- **CWA_URL**: URL to Calibre Web Automated instance
- **DOWNLOAD_SCHEDULE**: Cron expression for download schedule
- **SOURCE_CONFIG**: Configuration file for download sources
- **CALIBRE_PATH**: Path to shared Calibre library

## Download Sources

Configure sources in a config file (stored as ConfigMap or Secret):

```yaml
sources:
  - name: "Fantasy Series"
    type: rss
    url: "https://example.com/fantasy-books.rss"
    filter:
      authors:
        - "Brandon Sanderson"
        - "Patrick Rothfuss"
      
  - name: "Daily Free Books"
    type: newsletter
    email: "deals@bookbub.com"
    filter:
      price: free
      
  - name: "Goodreads List"
    type: goodreads
    list_id: "12345"
```

## Integration with CWA

CWA Downloader and CWA share the same Calibre library:

```
cwa-library/
  ├── metadata.db        # Shared database
  ├── Author 1/
  │   └── Book 1/
  │       ├── Book 1.epub
  │       └── cover.jpg
  └── Author 2/
      └── Book 2/
```

When CWA Downloader adds a book:
1. Downloads the eBook file
2. Adds entry to `metadata.db`
3. CWA sees the new entry
4. CWA makes it available in the web interface

## Scheduling

Default schedule: Daily at 2 AM

Modify schedule in `values.yaml`:
```yaml
env:
  - name: DOWNLOAD_SCHEDULE
    value: "0 2 * * *"  # Cron format: minute hour day month weekday
```

Examples:
- `0 2 * * *`: Daily at 2 AM
- `0 */6 * * *`: Every 6 hours
- `0 2 * * 1`: Every Monday at 2 AM

## Storage Requirements

- **Downloads cache**: ~5-10GB for temporary files
- **Library**: Shared with CWA

## Deployment

Deployed by ArgoCD along with CWA.

## Legal Considerations

**Important**: Only download eBooks from legal sources:
- ✅ Free promotional eBooks
- ✅ Public domain books
- ✅ Books you purchased
- ✅ DRM-free books from legitimate retailers
- ❌ Pirated content
- ❌ Books with DRM from unauthorized sources

Always respect copyright laws and author rights.

## Monitoring

Check downloader status:
```bash
kubectl get pods -n media -l app=cwa-downloader
kubectl logs -n media -l app=cwa-downloader
```

Logs show:
- Download attempts
- Successfully imported books
- Errors or failures
- Duplicate detections

## Troubleshooting

**No books downloading**: Check source configuration, verify network access

**Downloads not appearing in CWA**: Verify shared library volume, check file permissions

**Duplicates downloaded**: Improve deduplication filters, check ISBN matching

**High storage usage**: Configure cleanup of download cache

**Schedule not working**: Verify cron expression, check pod timezone

## Example Workflow

1. Subscribe to BookBub daily deals email
2. Configure CWA Downloader to monitor that email
3. Set filters for genres/authors you like
4. Downloader runs daily at 2 AM
5. Free books automatically added to library
6. Read them in CWA or on your device

## Security Considerations

- **Source credentials**: Store email/API credentials in Vault
- **Network access**: May need external network access
- **File permissions**: Ensure shared library is writable

## Limitations

- Requires properly formatted sources
- Some sources may have rate limits
- DRM-protected books cannot be processed
- Metadata quality depends on source

## Alternatives

If CWA Downloader doesn't meet your needs:
- Manual downloads and uploads to CWA
- Calibre desktop with email fetching
- Custom scripts using Calibre CLI tools
- Third-party tools like Readarr (for book automation)

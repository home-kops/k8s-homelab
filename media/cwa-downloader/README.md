# CWA Downloader

Automated eBook downloader companion service for Calibre Web Automated.

## What is CWA Downloader?

CWA Downloader is a service that automatically downloads eBooks from configured sources and imports them into your Calibre Web Automated library.

## What it does

- **Automated downloads**: Fetches eBooks from configured sources
- **Automatic import**: Imports downloaded books into CWA
- **Format handling**: Manages different eBook formats
- **Scheduling**: Runs downloads on a schedule
- **Deduplication**: Avoids downloading books you already have

## How it works

1. CWA Downloader runs on a schedule (e.g., daily)
2. It checks configured sources for new eBooks
3. Downloads matching books based on your criteria
4. Imports them into the Calibre library
5. CWA picks up the new books automatically
6. Books appear in your library with metadata

## Use Cases

- **Series completion**: Automatically download new books in a series
- **Author following**: Get new releases from favorite authors
- **Free eBooks**: Daily free book promotions
- **Newsletter integration**: Download books from deal newsletters
- **RSS feeds**: Monitor RSS feeds for eBook announcements

## Configuration

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

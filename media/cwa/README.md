# Calibre Web Automated (CWA)

Web-based eBook library manager with automatic format conversion.

## What is CWA?

Calibre Web Automated is a web interface for managing your eBook collection. It's built on Calibre (the popular desktop eBook manager) but accessible through a browser, with added automation for format conversion.

## What it does

- **eBook library management**: Organize books by author, series, tags, etc.
- **Format conversion**: Automatically convert between formats (EPUB, MOBI, AZW3, PDF)
- **Web reading**: Read books directly in your browser
- **Metadata editing**: Edit book information, covers, descriptions
- **Send to device**: Email books to Kindle or other devices
- **OPDS support**: Browse library from compatible eBook readers
- **User management**: Multiple users with different libraries

## Features

- **Multiple formats**: Supports 20+ eBook formats
- **Automatic conversion**: Converts books on upload or on-demand
- **Metadata search**: Fetch book info from online sources
- **Cover downloads**: Automatically find and download book covers
- **Reading integration**: Built-in eBook reader for browser
- **Email integration**: Send books directly to Kindle via email
- **Custom columns**: Add custom metadata fields

## How to use it

### Accessing CWA

Access through your configured ingress URL (e.g., `https://calibre.yourdomain.com`)

### Initial Setup

1. Login with default credentials (check `values.yaml`)
2. Configure Calibre library path
3. Set up format conversion preferences
4. Configure email for Kindle sending (optional)
5. Create user accounts if needed

### Adding Books

1. Upload eBooks through web interface
2. CWA automatically:
   - Fetches metadata (title, author, description)
   - Downloads cover art
   - Converts to configured formats
   - Adds to library

### Sending to Kindle

Configure email settings to send books to Kindle:
1. Add your Kindle email in user settings
2. Configure SMTP settings in admin panel
3. Click "Send to Kindle" on any book

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
deployment:
  image:
    repository: crocodilestick/calibre-web-automated
    tag: latest
    
  env:
    - name: PUID
      value: "1000"
    - name: PGID  
      value: "1000"
      
  volumes:
    - name: config
      persistentVolumeClaim:
        claimName: cwa-config
    - name: library
      persistentVolumeClaim:
        claimName: cwa-library
```

### Environment Variables

- **PUID/PGID**: User/group ID for file permissions
- **TZ**: Timezone for scheduled tasks
- **AUTO_CONVERT**: Enable automatic format conversion
- **CALIBRE_PATH**: Path to Calibre library

## Storage Requirements

- **Config**: ~500MB for CWA configuration
- **Library**: Depends on collection size (typically 10-100GB)
- **Conversion cache**: ~5GB for temporary conversion files

## Format Conversion

CWA can convert between formats:
- **EPUB** ↔ **MOBI** ↔ **AZW3** ↔ **PDF**
- **DOCX** → **EPUB**
- And many more...

Configure conversion preferences:
- Automatic conversion on upload
- Preferred output formats
- Conversion quality settings

## Integration with CWA Downloader

The [CWA Downloader](../cwa-downloader/README.md) companion service automatically downloads and imports eBooks into CWA.

## Deployment

Deployed by ArgoCD. Configuration in this directory.

## Ingress Configuration

Example IngressRoute in [templates/ingressroute.yaml](./templates/ingressroute.yaml):

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: cwa
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`calibre.example.com`)
      kind: Rule
      services:
        - name: cwa
          port: 8083
  tls:
    certResolver: letsencrypt
```

## OPDS Support

OPDS (Open Publication Distribution System) allows eBook readers to browse your library:

OPDS URL: `https://calibre.example.com/opds`

Compatible with:
- KOReader (e-ink devices)
- FBReader
- Marvin (iOS)
- Many other eBook readers

## Security Considerations

- **Authentication**: Enable user accounts
- **HTTPS**: Required for secure access
- **Email credentials**: Store SMTP credentials securely (use Vault)
- **User permissions**: Limit what users can delete or modify

## Backup

Important data to backup:
- Calibre library database (`metadata.db`)
- eBook files
- Cover images
- CWA configuration

## Troubleshooting

**Can't access CWA**: Check ingress, verify pod is running

**Books not appearing**: Verify library path, check file permissions

**Conversion failing**: Check Calibre backend logs, ensure conversion tools are installed

**Email not working**: Verify SMTP settings, check firewall/port blocking

**Slow performance**: Large libraries can slow down, consider database optimization

## Monitoring

Check CWA status:
```bash
kubectl get pods -n media -l app=cwa
kubectl logs -n media -l app=cwa
```

## Alternative to Calibre Desktop

CWA replaces the need for the Calibre desktop application for most tasks:
- ✅ Web access from any device
- ✅ No desktop app required
- ✅ Multi-user support
- ✅ Automated workflows
- ❌ Some advanced Calibre features not available

For advanced editing or troubleshooting, you can still use Calibre desktop on the library.

## Resources

- Calibre Web GitHub: https://github.com/janeczku/calibre-web
- Calibre documentation: https://manual.calibre-ebook.com/
- OPDS spec: https://opds.io/

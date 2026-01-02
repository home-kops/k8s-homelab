# Media

This module contains applications for managing and streaming your personal media library.

## What's included?

### [Jellyfin](./jellyfin/README.md)
A free media streaming server similar to Plex or Netflix, but for your personal content.

**What it does**: Streams your movies, TV shows, music, and photos to any device with a web browser or Jellyfin client app.

### [Calibre Web Automated (CWA)](./cwa/README.md)
A web-based eBook library manager with automatic conversion capabilities.

**What it does**: Manages your eBook collection, converts between formats (EPUB, MOBI, PDF), and provides a web interface for reading.

### [CWA Downloader](./cwa-downloader/README.md)
Companion service for Calibre Web Automated that handles automated eBook downloads.

**What it does**: Automatically downloads and imports eBooks into your Calibre library based on configured sources.

## How they work together

These applications operate independently but serve a unified purpose of managing your home media:

- **Jellyfin** handles video and audio streaming
- **CWA** and **CWA Downloader** handle your eBook library

Each application exposes a web interface accessible through your Traefik ingress controller.

## Deployment

These services are managed by ArgoCD. To deploy or update:

1. Modify the `values.yaml` file in the respective component directory
2. Commit and push your changes
3. ArgoCD automatically syncs the changes to your cluster

All services require persistent storage (provided by the NFS provisioner) to store media files and metadata.

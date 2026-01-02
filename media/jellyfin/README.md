# Jellyfin

Free media streaming server for your personal movies, TV shows, music, and photos.

## What is Jellyfin?

Jellyfin is like running your own Netflix or Spotify. It organizes your media library, transcodes video formats, and streams content to any device with a web browser or Jellyfin app.

## What it does

- **Media management**: Organizes movies, TV shows, music, and photos
- **Metadata fetching**: Automatically downloads posters, descriptions, and info
- **Transcoding**: Converts media formats for different devices on-the-fly
- **Multi-device streaming**: Watch on phones, tablets, smart TVs, web browsers
- **User management**: Create accounts for family members with different permissions
- **Resume playback**: Continue watching from where you left off
- **Remote access**: Stream your media anywhere (with proper security)

## Features

- **No subscriptions**: Completely free, no paid tiers
- **No ads**: Pure media streaming experience
- **Format support**: Plays almost any video/audio format
- **Subtitle support**: Automatic subtitle download and customization
- **Live TV & DVR**: With additional hardware (TV tuner)
- **Plugins**: Extend functionality with community plugins

## How to use it

### Accessing Jellyfin

Access through your configured ingress URL (e.g., `https://jellyfin.yourdomain.com`)

### Initial Setup

1. Create an admin account on first launch
2. Add media libraries (Movies, TV Shows, Music, etc.)
3. Point each library to your media storage location
4. Jellyfin scans and organizes your media
5. Create user accounts for family/friends

### Adding Media

Media should be organized following Jellyfin's naming conventions:

```
Movies/
  ├── Movie Name (2020)/
  │   └── Movie Name (2020).mkv
  └── Another Movie (2021)/
      └── Another Movie (2021).mp4

TV Shows/
  └── Show Name/
      ├── Season 01/
      │   ├── Show Name - S01E01.mkv
      │   └── Show Name - S01E02.mkv
      └── Season 02/
          └── Show Name - S02E01.mkv
```

## Configuration

Main configuration file: [values.yaml](./values.yaml)

Key settings:
```yaml
deployment:
  image:
    repository: jellyfin/jellyfin
    tag: latest
  replicas: 1
  
  volumes:
    - name: media
      nfs:
        server: your-nfs-server
        path: /mnt/media
    - name: config
      persistentVolumeClaim:
        claimName: jellyfin-config
```

### Hardware Acceleration

For better transcoding performance, enable hardware acceleration:

```yaml
securityContext:
  privileged: true  # Required for GPU access

# Add device mounts for GPU
volumeMounts:
  - name: gpu
    mountPath: /dev/dri
```

## Storage Requirements

- **Config**: ~1-2GB for Jellyfin configuration and metadata
- **Media**: Depends on your library size (typically hundreds of GB to TBs)
- **Transcoding**: Temporary storage for transcoded files

## Client Applications

- **Web**: Access directly through browser
- **Android/iOS**: Official Jellyfin apps
- **Android TV**: App available on Play Store
- **Roku**: Jellyfin channel
- **Smart TVs**: Apps for Samsung, LG, etc.
- **Desktop**: Windows, Mac, Linux applications

## Transcoding

Jellyfin transcodes media when:
- Device doesn't support the original format
- Network bandwidth is limited
- You manually select a lower quality

Transcoding is CPU/GPU intensive. For best performance:
- Enable hardware acceleration (Intel QuickSync, NVIDIA, AMD)
- Store original media in widely-compatible formats (H.264/AAC)
- Upgrade server hardware if experiencing lag

## Deployment

Deployed by ArgoCD. Configuration in this directory.

## Ingress Configuration

Example IngressRoute in [templates/ingressroute.yaml](./templates/ingressroute.yaml):

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: jellyfin
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`jellyfin.example.com`)
      kind: Rule
      services:
        - name: jellyfin
          port: 8096
  tls:
    certResolver: letsencrypt
```

## Security Considerations

- **Authentication**: Enable user authentication, don't allow anonymous access
- **HTTPS**: Always use HTTPS for remote access
- **User permissions**: Limit what users can delete or modify
- **Network security**: Consider VPN for remote access instead of exposing publicly
- **Regular updates**: Keep Jellyfin updated for security patches

## Troubleshooting

**Can't access Jellyfin**: Check ingress configuration, verify pod is running

**Media not appearing**: Check file permissions, verify media paths, trigger library scan

**Buffering/lag**: Enable hardware acceleration, check network bandwidth, reduce quality

**Transcoding fails**: Check Jellyfin logs, verify hardware acceleration is working

**High CPU usage**: Transcoding is intensive, enable hardware acceleration or reduce concurrent streams

## Monitoring

Check Jellyfin status:
```bash
kubectl get pods -n media -l app=jellyfin
kubectl logs -n media -l app=jellyfin
```

Monitor transcoding and streaming in Jellyfin's web dashboard.

## Resources

- Official docs: https://jellyfin.org/docs/
- Naming guide: https://jellyfin.org/docs/general/server/media/movies.html
- Hardware acceleration: https://jellyfin.org/docs/general/administration/hardware-acceleration.html

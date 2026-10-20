# selfhost

## Filebrowser Quantum

```
services:
  filebrowser:
    image: ghcr.io/gtsteffaniak/filebrowser:latest
    container_name: quantum-prod
    user: "1001:1001"
    ports:
      - 8900:80
    restart: unless-stopped
    # user: filebrowser
    volumes:
      - ./data:/home/filebrowser/data
      - ./files:/files
      # - /other/dir:/dir # Add other sources
    environment:
      - "FILEBROWSER_CONFIG=data/config.yaml" # using our config file at ./data/config.yaml
```

Prod structure

```
filebrowser-quantum/
├── .env
├── compose.yaml
└── files/
└── data/
    ├── config.yaml
    ├── filebrowser.sqlite
    └── tmp/
```

```
# compose.yaml
services:
  filebrowser:
    image: ghcr.io/gtsteffaniak/filebrowser:latest
    container_name: quantum-prod
    user: "1000:1000"
    ports:
      - 8900:80
    restart: unless-stopped
    # user: filebrowser
    volumes:
      - ./data:/home/filebrowser/data
      - ./files:/files
      # - /other/dir:/dir # Add other sources
    environment:
      - "FILEBROWSER_CONFIG=data/config.yaml"
```

```
# config.yaml
http:
  baseURL: "/fb"
  externalUrl: "https://your-machine.tailnet.ts.net/fb"
  trustProxyHeaders: true

server:
  database:
    path: "data/filebrowser.sqlite"
  cacheDir: "data/tmp"
  sources:
    - path: "/files"
      name: Home
      config:
        defaultUserScope: "/users/" # new users will get created in /users/<username>
        defaultEnabled: true # all users get this source on create; v2.0.1+ also merges for existing u>
        createUserDir: true # a user "bill" will see files from /files/users/bill
    - path: "/home/filebrowser" # mount the docker home folder for convenience
      name: Backend
    # Add your sources here.
  #externalUrl: 'https://<YOUR_IP>:8900' # if you plan to share externally, share links will be genera>
  maxArchiveSize: 50 # maxiumum pre-archive size users are allowed to download at once.
auth:
  tokenExpirationHours: 2
  methods:
    password:
      enabled: true
      minLength: 5
      signup: true
```

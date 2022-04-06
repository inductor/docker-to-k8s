## 0. Docker のおさらい

ここに Docker Compose のファイルがあります

```yaml
version: '3'

services:
  minecraft:
    image: itzg/minecraft-server:java17
    container_name: minecraft
    stdin_open: true
    tty: true
    ports:
      - "25565:25565"
      - "8123" # Dynmap
    volumes:
      - "./data:/data"
    environment:
      EULA: "TRUE"
      MEMORY: "2G"
      TYPE: "SPIGOT"
      VERSION: "1.18.1"
      MODS: "https://github.com/webbukkit/dynmap/releases/download/v3.3-beta-2/Dynmap-3.3-beta-2-spigot.jar"
    restart: always
  nginx:
    image: nginx:mainline
    container_name: nginx
    stdin_open: true
    tty: true
    volumes:
      - "./conf.d:/etc/nginx/conf.d"
    ports:
      - "80:80"
    restart: always
```

何も難しくないですね。Dynmap を 落としてきて動かすだけの簡単な Spigot サーバーと、それにアクセスさせるための Nginx が動いています。

Docker Compose を使うと、この定義を置いといて `docker compose up -d` とかやるだけでアプリケーションが立ち上がってくれるとても便利なツールです。

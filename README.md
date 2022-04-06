# Docker が分かる人のための Kubernetes 入門

このリポジトリは、Docker は分かるけど Kubernetes は分からん、という人が Kubernetes に入門するために作られました。

本編を進めるにあたっては、以下のような環境を用意してください。

- Docker 20.10 (Desktop でも Linux でも)
- bash (zsh でもいいですが互換性は保証しません) on Linux / macOS / WSL2

また、前提として以下の知識はあるものとします。

- `docker build` 及び `docker run` の基本的な使い方が分かる
- `docker-compose` を、Volume なども含めて一通り使うことができる
- Web 3層構造と聞いてなんのことだか分かる
- 基本的な TCP/IP 通信について話が理解できる

0. [Docker のおさらい](./00-docker)
1. [Docker Compose から Kubernetes Manifest を作ってみよう](./01-docker-and-k8s)

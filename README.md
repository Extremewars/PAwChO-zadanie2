# PAwChO-zadanie2

[![Github build status](https://github.com/Extremewars/PAwChO-zadanie2/actions/workflows/workflow.yml/badge.svg)](https://github.com/Extremewars/PAwChO-zadanie2/actions/workflows/workflow.yml)
[![GitHub release](https://img.shields.io/github/release/Extremewars/PAwChO-zadanie2.svg)](https://github.com/Extremewars/PAwChO-zadanie2/releases)

## Pobranie najnowszego obrazu

### Przez Docker Hub

[Repozytorium Docker Hub - zadanie2](https://hub.docker.com/repository/docker/extremical/zadanie2/general)
```bash
docker pull extremical/zadanie2:latest
```
Wersja v1.0.0  
```bash
docker pull extremical/zadanie2:v1.0.0
```

### Przez GHCR

[Pakiet GHCR - zadanie2](https://github.com/Extremewars/PAwChO-zadanie2/pkgs/container/zadanie2).
Ten pakiet jest również podpięty pod to repozytorium  
```bash
docker pull ghcr.io/extremewars/zadanie2:latest
```
Wersja v1.0.0  
```bash
docker pull ghcr.io/extremewars/zadanie2:1.0.0
```

## Rodzaj wersjonowania

Do wersjonowania tagów repozytorium oraz tagów obrazów wykorzysstano wersjonowanie semantyczne: [Wersjonowanie semantyczne 2.0.0](https://semver.org/lang/pl/spec/v2.0.0.html)  
  
> Dla numeru wersji MAJOR.MINOR.PATCH, zwiększaj:  
wersję MAJOR, gdy dokonujesz zmian niekompatybilnych z API,  
wersję MINOR, gdy dodajesz nową funkcjonalność, która jest kompatybilna z poprzednimi wersjami,  
wersję PATCH, gdy naprawiasz błąd nie zrywając kompatybilności z poprzednimi wersjami.  

W przypadku tego repozytorium będą stosowane głównie bardzo małe poprawki, więc wersjonowanie/tagi będą wyglądały następująco:  
```v1.0.x```
gdzie x - numer poprawki/commita

## Zarządzanie danymi cache

Dane cache są przechowywane w oddzielnym repozytorium Docker Hub:
[zadanie2-cache](https://hub.docker.com/repository/docker/extremical/zadanie2-cache/general)

Przy publikacji obrazu na repozytoria ponownie korzystamy z cache'u w ostatnim kroku workflowa:
```yml
-
  name: Build and push Docker image
  if: success()
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    platforms: linux/amd64,linux/arm64
    push: true
    cache-from: |
      type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest
    cache-to: |
      type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest,mode=max
    tags: ${{ steps.meta.outputs.tags }}
```

## Testy CVE

Testowanie bezpieczeństwa jest podzielone na dwa etapy oddzielnie dla dwóch architektur.  
Najpierw jest ładowany obraz bez publikowania za pomocą flag:
```yml
push: false
load: true
```

Załadowanie obrazu:
```yml
- 
  name: Build an arm64 image for Trivy check
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    platforms: linux/arm64
    push: false
    load: true
    cache-from: |
      type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest
    cache-to: |
      type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest,mode=max
    tags: zadanie2:arm64
```
Do sprawdzenia obrazu zastosowano narzędzie Trivy zaimplementowane jako GitHub Action: [Trivy Action](https://github.com/aquasecurity/trivy-action)  
Sprawdzenie obrazu:
```yml
-
  name: Run Trivy scanner for arm64 image
  uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: image
    image-ref: zadanie2:arm64
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    vuln-type: 'os,library'
    severity: 'CRITICAL,HIGH'
```
Narzędzie Trivy sprawdza pod kątem podatności CRITICAL i HIGH. Jeśli natknie się na zagrożenie to zwróci kod 1, 
co będzie sprawdzane w ostatnim kroku publikowania obrazu we fladze ```if: success()```.

## Źródłowy workflow

```yml
name: Pushing image to repositories (GHCR and Dockerhub)

on:
  workflow_dispatch:
  push:
    tags:
    - 'v*'

jobs:
  ci_step:
    name: Build, tag and push Docker image to GHCR and Dockerhub
    runs-on: ubuntu-latest
      
    steps:
      -
        name: Check out the source_repo
        uses: actions/checkout@v4
      
      - 
        name: Docker metadata definitions
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ghcr.io/${{ github.repository_owner }}/zadanie2
            docker.io/${{ vars.DOCKERHUB_USERNAME }}/zadanie2
          flavor: latest=false
          tags: |
            type=sha,priority=100,prefix=sha-,format=short
            type=semver,priority=200,pattern={{version}}   

      - 
        name: QEMU set-up
        uses: docker/setup-qemu-action@v3

      - 
        name: Buildx set-up
        uses: docker/setup-buildx-action@v3

      - 
        name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GIT_TOKEN }}

      - 
        name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - 
        name: Build an arm64 image for Trivy check
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/arm64
          push: false
          load: true
          cache-from: |
            type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest
          cache-to: |
            type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest,mode=max
          tags: zadanie2:arm64
      -
        name: Run Trivy scanner for arm64 image
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: image
          image-ref: zadanie2:arm64
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

      - 
        name: Build an amd64 image for Trivy check
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64
          push: false
          load: true
          cache-from: |
            type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest
          cache-to: |
            type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest,mode=max
          tags: zadanie2:amd64
      
      -
        name: Run Trivy scanner for amd64 image
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: image
          image-ref: zadanie2:amd64
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

      -
        name: Build and push Docker image
        if: success()
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64
          push: true
          cache-from: |
            type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest
          cache-to: |
            type=registry,ref=${{ vars.DOCKERHUB_USERNAME }}/zadanie2-cache:latest,mode=max
          tags: ${{ steps.meta.outputs.tags }}
```

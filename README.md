# source-repo

## Zadanie 2 - Laboratorium 10

### Zawartość pliku index.html:

```HTML
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Zadanie 2</title>
</head>
<body style="width: 100%; height: 100vh;">
    <div style="height: 100%; display: flex; flex-direction: column; justify-content: center; align-items: center;">
        <h1>Zadanie 2</h2>
        <br>
        <br>
        <h3>Informacje</h3>
        <h4><b>Imie i nazwisko:</b> Szymon Mikołajczuk</h4>
        <h4>Wersja aplikacji:</h4>
        <p id="appVersion"></p>
        <h3>Dockerfile:</h3>
        <code>
            FROM nginx:alpine <br>
            WORKDIR /usr/share/nginx/html <br>
            COPY ./index.html . <br>
            EXPOSE 80 <br>
            CMD ["nginx", "-g", "daemon off;"] <br>
        </code>
    </div>
</body>
</html>
```

### Zawartość pliku dockerfile obrazu aplikacji:
```Dockerfile
FROM nginx:alpine 
WORKDIR /usr/share/nginx/html
COPY ./index.html .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Zawartość pliku zad2lab10.yaml
```yaml
name: Docker CI

on:
  workflow_dispatch:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  dockerCI:
    name: dockerCI
    runs-on: ubuntu-latest
    steps:
      - name: Set Run ID to Variable
        run: echo "NEW_VERSION=${{ github.run_id }}" >> $GITHUB_ENV

      - name: Check out repo
        uses: actions/checkout@v4
      
      - name: Qemu installation
        uses: docker/setup-qemu-action@v3
        
      - name: Installation of Buildx image building engine
        uses: docker/setup-buildx-action@v3

      - name: Replace version in HTML
        run: sed -i 's|<p id="appVersion">.*<\/p>|<p id="appVersion">'"${NEW_VERSION}"'<\/p>|' index.html

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./
          push: true
          tags: szymonmik/zad2-img:${{ env.NEW_VERSION }}
          platforms: linux/amd64,linux/arm64
          
  kubernetesCI:
    name: kubernetesCI
    needs: dockerCI
    runs-on: ubuntu-latest
    steps:
      - name: Set Run ID to Variable
        run: echo "NEW_VERSION=${{ github.run_id }}" >> $GITHUB_ENV
      - name: Check out config-repo
        uses: actions/checkout@master
        with:
          repository: szymonmik/config-repo
          token: ${{ secrets.PAT_GITHUB }}
          
      - name: Edit deployment
        run: |
          git config --global user.name "ci"
          git config --global user.email "ci@ci.ci"
          yq -i '.spec.template.spec.containers[0].image |= "szymonmik/zad2-img:" + strenv(NEW_VERSION)' deployment.yaml
          yq -i '.spec.template.spec.containers[0].env[0].value = strenv(NEW_VERSION)' deployment.yaml
          git add deployment.yaml
          git commit -m "update deployment"
          git push origin main
```

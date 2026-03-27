

docker driver costuma carregar localmente por padrão;

docker-container não faz isso automaticamente, então vc precisa escolher entre
````
--push
--load
````

````
wsl docker buildx build -f ./src/Usuarios.Api/Dockerfile -t ghcr.io/felipementel/deploy-dotnet-model:1.0 --push .
````
````
wsl docker buildx build -f ./src/Usuarios.Api/Dockerfile -t ghcr.io/felipementel/deploy-dotnet-model:2.0 --load .

wsl docker push ghcr.io/felipementel/deploy-dotnet-model:2.0
````

# Azure Container Registry

````
az acr build -f .\src\Usuarios.Api\Dockerfile --registry canaldeploy --image felipementel/deploy-dotnet-model:3.0 .
````

# OCI
[Docker OCI](https://docs.docker.com/build/exporters/oci-docker/)

## Docker-container [ type=docker ]

### Exportar para Arquivo Tarball
````
wsl docker buildx build -f ./src/Usuarios.Api/Dockerfile -t ghcr.io/felipementel/deploy-dotnet-model:4.0 --output=type=docker,dest=./deploy-dotnet-model.tar .
````

````
wsl docker buildx build -f ./src/Usuarios.Api/Dockerfile -t ghcr.io/felipementel/deploy-dotnet-model:4.0 --output=type=docker,dest=./deploy-dotnet-model.tar,name=ghcr.io/felipementel/deploy-dotnet-model:4.0,compression-level=22,force-compression=true .
````

### Importar para o Daemon Local
````
wsl docker load -i ./deploy-dotnet-model.tar
````
## OCI com skopeo [ type=oci ]

### 0. instalar
````
wsl sudo apt-get update && wsl sudo apt-get install -y skopeo
````

### 1. Criar um builder com driver docker-container
````
wsl docker buildx create --name mybuilder --driver docker-container --use
````

### 2. Agora o export OCI funciona
````
wsl docker buildx build -f ./src/Usuarios.Api/Dockerfile -t ghcr.io/felipementel/deploy-dotnet-model:4.0 --output=type=oci,dest=/tmp/deploy-dotnet-model.tar .
````

### 3. Importar para o daemon local via skopeo [ Gera erro no windows ] 
````
wsl skopeo copy oci-archive:/tmp/deploy-dotnet-model.tar docker-daemon:ghcr.io/felipementel/deploy-dotnet-model:4.0
````
### 4. (Opcional) Depois de usar, remover o builder
````
wsl docker buildx rm mybuilder
````

## OCI com crane [ type=oci ]

````
wsl bash -c "curl -sL https://github.com/google/go-containerregistry/releases/latest/download/go-containerregistry_Linux_x86_64.tar.gz | sudo tar -xzf - -C /usr/local/bin crane"
````

````
wsl docker buildx build -f ./src/Usuarios.Api/Dockerfile -t ghcr.io/felipementel/deploy-dotnet-model:5.0 --output=type=oci,dest=./deploy-dotnet-model.tar .
````

````
wsl sudo ctr -n moby images import ./deploy-dotnet-model.tar
````

````
wsl crane push ./deploy-dotnet-model.tar ghcr.io/felipementel/deploy-dotnet-model:4.0
````

````
wsl docker image ls # Nada vai acontecer
````
---
O ctr importou para o image store do containerd, mas o docker image ls consulta o image store do Docker (legado). São stores separados.

Para a imagem aparecer em docker image ls, você precisa habilitar o containerd image store no Docker Desktop:

Settings > General > "Use containerd for pulling and storing images" (marque e reinicie o Docker Desktop)

Com isso, ambos (ctr e docker) passam a usar o mesmo store, e a imagem aparecerá em docker image ls.

Sem isso, a imagem existe no containerd mas o Docker não a enxerga. Você pode confirmar que ela está lá com:

Resumo: sem containerd image store ativado, não há como fazer o OCI tar virar uma imagem visível no docker image ls sem converter o formato. A forma mais prática continua sendo:

````
wsl sudo ctr -n moby images ls
````

### Para analisar o ambiente local
````
wsl docker system df
````

### Para limpar o ambiente
````
wsl docker rm -f $(wsl docker ps -aq) 2>$null; wsl docker system prune -a -f --volumes
````

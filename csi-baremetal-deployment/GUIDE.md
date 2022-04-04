# Стартовое задание Ферапонтов Михаил
## Установка CSI-baremetal на kind

### Приготовления:
### Требования
Желательно наличие Ubuntu (использование других дистрибутивов Linux очень ~~больно~~ нежелательно)
1. docker
2. helm
3. kubectl
4. lvm2
5. kind
6. go
7. 50GB свободного места

### Установка docker
```
 sudo apt-get update
 sudo apt-get install ca-certificates curl gnupg lsb-release

 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

 echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Действия после установки докера, чтобы использовать его как non-root user
```
 sudo groupadd docker
 sudo usermod -aG docker $USER
```
### Установка helm
Установка по [гайду](https://helm.sh/docs/intro/install/)
Чтобы мы могли использовать отовсюду:
```
 sudo install helm /usr/local/bin/helm
```

### Установка kubectl
Установка по [гайду](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
Чтобы мы могли пользоваться отовсюду:
```
 sudo install kubectl /usr/local/bin/kubectl
```

### Установка lvm2
```
sudo apt-get install lvm2
```

### Установка kind
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.12.0/kind-linux-amd64
chmod +x ./kind
```
Чтобы пользоваться отовсюду:
```
sudo install kind /usr/local/bin/kind
```

### Установка go
Скачиваем tar архив с [официального сайта go](https://go.dev/doc/install)
Теперь разархивируем
```
tar -C $HOME -xzvf <наш архив>
```
Чтобы пользоваться go отовсюду меняем содержимое файла $HOME/.profile, в конце добавляем:
```
#set PATH so it includes user's go bin if it exists
if [ -d "$HOME/go/bin" ]; then
    PATH="$HOME/go/bin:$PATH"
fi

#set PATH so it includes user's go plugins bin if it exists
if [ -d "$HOME/go/bin/bin" ]; then
    PATH="$HOME/go/bin/bin:$PATH"
fi
```
### Установка
Клонируем в рабочий репозиторий [csi-baremetal](https://github.com/dell/csi-baremetal) [csi-baremetal-operator](https://github.com/dell/csi-baremetal-operator)

Запускаем в рабочем репозитории скрипт build.sh:
```
#!/bin/bash
export REGISTRY=<your_docker_hub>
export CSI_BAREMETAL_DIR=<full_path_to_csi_baremetal_src>
export CSI_BAREMETAL_OPERATOR_DIR=<full_path_to_csi_baremetal_operator_src>

cd ${CSI_BAREMETAL_DIR}
export CSI_VERSION=`make version`

# Get dependencies
make dependency

# Compile proto files
make install-compile-proto

# Generate CRD
make install-controller-gen
make generate-deepcopy

# Clean previous artefacts
make clean

# Build binary
make build
make DRIVE_MANAGER_TYPE=loopbackmgr build-drivemgr

# Build docker images
make download-grpc-health-probe
make images REGISTRY=${REGISTRY}
make DRIVE_MANAGER_TYPE=loopbackmgr image-drivemgr REGISTRY=${REGISTRY}

cd ${CSI_BAREMETAL_OPERATOR_DIR}
export CSI_OPERATOR_VERSION=`make version`

# Build docker image
make docker-build REGISTRY=${REGISTRY}
```
В рабочем репозитории запускаем скрипт kind.sh:
```
#!/bin/bash
export REGISTRY=<your_docker_hub>
export CSI_BAREMETAL_DIR=<full_path_to_csi_baremetal_src>
export CSI_BAREMETAL_OPERATOR_DIR=<full_path_to_csi_baremetal_operator_src>

cd ${CSI_BAREMETAL_DIR}
export CSI_VERSION=`make version`
cd ${CSI_BAREMETAL_OPERATOR_DIR}
export CSI_OPERATOR_VERSION=`make version

cd ${CSI_BAREMETAL_DIR}

# Build custom kind binary
make kind-build

# Create kind cluster
make kind-create-cluster

# Check cluster
kubectl cluster-info --context kind-kind

# If you use another path for kubeconfig or don't set KUBECONFIG env
kind get kubeconfig > <path_to_kubeconfig>

# Prepare sidecars 
make deps-docker-pull
make deps-docker-tag

# Retag CSI images and load them to kind
make kind-tag-images TAG=${CSI_VERSION} REGISTRY=${REGISTRY}
make kind-load-images TAG=${CSI_VERSION} REGISTRY=${REGISTRY}
make load-operator-image OPERATOR_VERSION=${CSI_OPERATOR_VERSION} REGISTRY=${REGISTRY}

cd ${CSI_BAREMETAL_OPERATOR_DIR}

# Install Operator
helm install csi-baremetal-operator ./charts/csi-baremetal-operator/ \
    --set image.tag=${CSI_OPERATOR_VERSION} \
    --set image.pullPolicy=IfNotPresent

#Install Deployment
helm install csi-baremetal ./charts/csi-baremetal-deployment/ \
    --set image.tag=${CSI_VERSION} \
    --set image.pullPolicy=IfNotPresent \
    --set scheduler.patcher.enable=true \
    --set driver.drivemgr.type=loopbackmgr \
    --set driver.drivemgr.deployConfig=true \
    --set driver.log.level=debug \
    --set scheduler.log.level=debug \
    --set nodeController.log.level=debug
```
Редактируем файл csi-baremetal/test/app/nginx.yaml изменя количество replicas на 3.

### Validation
```# Install test app
cd ${CSI_BAREMETAL_DIR}
kubectl apply -f test/app/nginx.yaml

# Check all pods are Running and Ready
kubectl get pods

# And all PVCs are Bound
kubectl get pvc

```

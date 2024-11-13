# Развертывание Deckhouse на узле
Подробная инструкция от Elma365 [ELMA365 On-Premises / Подготовка инфраструктуры](https://elma365.com/ru/help/platform/kubernetes-deckhouse-air-gap.html)
## Создать ключ SSH
Создать ключ SSH для подключения к кластеру Deckhouse'ом
```sh
ssh-keygen
```
скопировать ключ на удаленную машину можно командой:
```sh
ssh-copy-id <username>@<master_ip>
```

## Подготовка узла к Deckhouse

Необходимо переименовать хост если имя начинается с заглавной буквы. 

```sh
sudo hostnamectl set-hostname <new_hostname>
```

## Запуск установщика Deckhouse

Получить __config.yml__ по ссылке: [Deckhouse Kubernetes Platform на bare metal](https://deckhouse.ru/products/kubernetes-platform/gs/bm/step2.html), указав шаблон для DNS-имен кластера, например: %s.dev.example.com

внести необходимые изменения в config.yml:
- __podSubnetCIDR__ – адресное пространство подов кластера
- __serviceSubnetCIDR__ – адресное пространство сети сервисов кластера
- __kubernetesVersion__ – версия кластера
- __releaseChannel__ – канал обновлений (___Stable___)
- __publicDomainTemplate__ – проверить шаблон доменного имени. Например, Grafana для шаблона %s.example.com будет доступна, как grafana.example.com
- __podNetworkMode__ – режим работы модуля cni-flannel, допустимые значения ___VXLAN___ (если сервера имеют связность L3) или ___HostGW___ (для L2-сетей)
- __internalNetworkCIDRs__ – CIDR-адреса сетей внутренних подсетей кластера (используется для связи компонентов Kubernetes (kube-apiserver, kubelet и т. д.) между собой)

Запустить установщик на __рабочей машине__:

```sh
docker run --pull=always -it -v "$PWD/config.yml:/config.yml" -v "$HOME/.ssh/:/tmp/.ssh/" registry.deckhouse.ru/deckhouse/ce/install:stable bash
```

## Установка Deckhouse на узел

Внутри контейнера установщика выполнить команду:

```sh
dhctl bootstrap --ssh-user=<username> --ssh-host=<master_ip> --ssh-agent-private-keys=/tmp/.ssh/id_rsa \
--config=/config.yml \
--ask-become-pass
```


<username> — В параметре --ssh-user укажите имя пользователя, от которого генерировался SSH-ключ для установки;
<master_ip> — IP адрес master-узла подготовленного на этапе подготовка инфраструктуры.

Процесс установки может занять 15-30 минут при хорошем соединении.

## Настройка Deckhouse на узле

Подключиться к мастер-узлу

### Сниять с узла ограничения taint:
```sh
kubectl patch nodegroup master --type json -p '[{"op": "remove", "path": "/spec/nodeTemplate/taints"}]'
```
(Подразумевается что кластер состоит из одного узла, таким образом разрешаем компонентам Deckhouse работать на master-узле)
### Увеличить количества подов на master-ноде:
```sh
kubectl patch nodegroup master --type=merge -p '{"spec":{"kubelet":{"maxPods":200}}}'
```

### Добавить Local Path Provisioner
создать файл local-path-provisioner.yaml со следующим содержимым:
```yaml
apiVersion: deckhouse.io/v1alpha1
kind: LocalPathProvisioner
metadata:
  name: localpath-deckhouse-system
spec:
  nodeGroups:
  - master
  path: "/opt/local-path-provisioner"
  reclaimPolicy: "Delete"
```

Создать папку _/opt/local-path-provisioner_ и применить файл _local-path-provisioner.yaml_ в Kubernetes:
```sh
kubectl apply -f local-path-provisioner.yaml
```
Созданный _LocalPathProvisioner_ установить как storageclass по умолчанию (default-class):
```sh
kubectl patch storageclass localpath-deckhouse-system -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
### Добавить Ingress Nginx Controller

создать файл _ingress-nginx-controller.yaml_ со следующим содержимым:
```yaml
# секция, описывающая параметры nginx ingress controller
# используемая версия API Deckhouse
apiVersion: deckhouse.io/v1
kind: IngressNginxController
metadata:
  name: nginx
spec:
  # имя Ingress-класса для обслуживания Ingress NGINX controller
  ingressClass: nginx
  # версия Ingress‑контроллера (в примере используется версия 1.9 с Kubernetes 1.23+)
  controllerVersion: "1.9"
  # способ поступления трафика из внешнего мира
  inlet: HostPort
  hostPort:
    httpPort: 80
    httpsPort: 443
  # описывает, на каких узлах будет находиться компонент
  # при необходимости можно изменить
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
  - operator: Exists
```
применить файл _ingress-nginx-controller.yaml_:
```sh
kubectl create -f ingress-nginx-controller.yaml
```
### Добавление пользователя для доступа к веб-интерфейсу кластера

создать файл _user.yaml_:
```yaml
apiVersion: deckhouse.io/v1
kind: ClusterAuthorizationRule
metadata:
  name: admin
spec:
  # список учётных записей Kubernetes RBAC
  subjects:
  - kind: User
    name: admin@jft.team
  # предустановленный шаблон уровня доступа
  accessLevel: SuperAdmin
  # разрешить пользователю делать kubectl port-forward
  portForwarding: true
---
# секция, описывающая параметры статического пользователя
# используемая версия API Deckhouse
apiVersion: deckhouse.io/v1
kind: User
metadata:
  name: admin
spec:
  # e-mail пользователя
  email: admin@jft.team
  # это хэш пароля H793dsf-Sd, сгенерированного сейчас
  # сгенерируйте свой или используйте этот, но только для тестирования
  # echo "H793dsf-Sd" | htpasswd -BinC 10 "" | cut -d: -f2
  # при необходимости можно изменить
  password: '$2y$10$qcuHRI8l..WK0NRjP0Av2.fzPLT1kk5TmCmZS4gpFIbNxUzYeyawC'
```
применить файл _user.yaml_:
```sh
kubectl create -f user.yaml
```
Подробнее в документации на deckhouse.ru: [Модуль user-authz: Custom Resources](https://deckhouse.ru/products/kubernetes-platform/documentation/v1/modules/140-user-authz/cr.html#clusterauthorizationrule)

# Установка Helm
Со страницы релизов [Helm](https://github.com/helm/helm/releases) скачать релиз, респаковать и установить:
```sh
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
tar -zxvf helm-v3.16.2-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

# Подготовка БД

Чарт elma365-dbs: 
[dl.elma365.com/onPremise/latest/elma365-dbs-latest.tar.gz](dl.elma365.com/onPremise/latest/elma365-dbs-latest.tar.gz)


```sh
sudo helm upgrade --install elma365-dbs ./elma365-dbs -f values-dbs-test.yaml -n elma365-dbs-test --create-namespace
```

убедитья что база подготовлена:

```sh
kubectl get events -n elma365-dbs-test

kubectl logs -n elma365-dbs-test -l app.kubernetes.io/instance=elma365-dbs
```

### удаление чарта

```sh
helm uninstall elma365-dbs -n <namespace>
kubectl delete namespace <namespace>
```


# Установка Elma365

```sh
kubectl create ns elma365

kubectl label namespace elma365 security.deckhouse.io/pod-policy=privileged --overwrite
```

## Подготовка secret с сертификатом для работы по HTTPS

Копирование файла по SSH:
```sh
scp -i ~/.ssh/id_rsa path/to/cert/file username@master_ip:path/to/cert/file
scp -i ~/.ssh/id_rsa path/to/key/file username@master_ip:path/to/key/file
```

В namespace с целевым приложением создать secret типа tls с наименованием elma365-onpremise-tls:
```sh
kubectl create secret tls elma365-onpremise-tls \
--cert=path/to/cert/file \
--key=path/to/key/file [-n namespace]
```

| Параметр | Описание |
| --- | --- |
| _--cert_ | Путь к файлу с сертификатом |
| _--key_ | Путь к файлу с приватным ключом |
| _-n_ | Имя namespace |


Подробнее: [ELMA365 справка по подготовке secret с сертификатом для работы по HTTPS](https://elma365.com/ru/help/platform/preparation-secret-with-certificate-https.html)

### также смотреть __возможные проблемы__

## скачать чарт необходимой версии

Описаное в [ELMA365 справке](https://elma365.com/ru/help/platform/links-for-install-elma365.html#links-for-install-helm)

Определённая минорная версия, например 2024.8.10:
https://dl.elma365.com/onPremise/2024/8/10/elma365-2024.8.10.tar.gz

```sh
wget https://dl.elma365.com/onPremise/2024/8/10/elma365-2024.8.10.tar.gz
tar -zxf elma365-2024.8.10.tar.gz
```

## установка чарта

```sh
helm upgrade --install elma365 ./elma365 -f values-elma365.yaml --timeout=30m --wait -n elma365
```
## Возможные проблемы

Если при установке обнаружено сообщение:
```
Warning   FailedCreate       replicaset/worker-bd95fb58f                               Error creating: admission webhook "admission-policy-engine.deckhouse.io" denied the request: [d8-pod-security-baseline-deny-default] Privileged container is not allowed: vm-overcommit-memory, securityContext: {"privileged": true, "runAsUser": 0}
```

то следует установить соответствующую аннотацию:
```sh
kubectl annotate ns <elma-namespace> pod-security.kubernetes.io/enforce=privileged --overwrite
```

## удаление чарта

```sh
helm uninstall elma365 -n <namespace>
kubectl delete namespace <namespace>
```

# Установка Onlyoffice
```sh
kubectl create namespace onlyoffice
helm show values elma365/onlyoffice > values-onlyoffice.yaml
```
Создать секрет с сертификатом для пространства имен onlyoffice.
В параметре onlyoffice.ingress.enabled должно быть значение __true__.

Установка:
```sh
helm upgrade --install onlyoffice elma365/onlyoffice -f values-onlyoffice.yaml -n onlyoffice 
```


# Резервное копирование и восстановление баз данных: утилита Elma365-Backupper

Подробнее: [ELMA365 справка по резервному копированию и восстановлению баз данных](https://elma365.com/ru/help/platform/database-backup-and-recovery.html)

## установка утилиты
```sh
sudo apt install -y apt-transport-https ca-certificates curl
```
импортировать ключи:
```sh
sudo curl -fsSL https://repo.elma365.tech/deb/elma365-keyring.gpg | gpg --dearmor > /etc/apt/trusted.gpg.d/elma365-keyring.gpg
```
добавить репозиторий ELMA365:
```sh
echo "deb [arch=amd64] https://repo.elma365.tech/deb $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/elma365.list
sudo apt update
```
установка утилиты:
```sh
sudo apt install elma365-backupper
```


## восстановление из дампа

```sh
elma365-backupper restore all --config /opt/elma365/backupper/etc/<config-file> --backup-path /path/to/backup --cleanup-databases
```
## Как посмотреть версию Deckhouse?

```sh
kubectl get deckhousereleases
```

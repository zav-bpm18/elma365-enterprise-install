## Запуск установщика Deckhouse

Запустите установщик на __персональном компьютере__.

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

## Подготовка secret с сертификатом для работы по HTTPS

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
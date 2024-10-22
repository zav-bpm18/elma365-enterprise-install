
Запустите установщик на __персональном компьютере__.

```sh
docker run --pull=always -it -v "$PWD/config.yml:/config.yml" -v "$HOME/.ssh/:/tmp/.ssh/" registry.deckhouse.ru/deckhouse/ce/install:stable bash
```
# Настройка облака для проведения нагрузочного тестирования и создание агента

_Примечание._ В приведенных ниже фрагментах кода предполагается, что параметр `folder-id` уже задан в конфигурации `yc`:

```bash
yc config set folder-id 'my_folder_id'
```

В противном случае можно передать аргумент `--folder-id my_folder_id` в командах `yc`.

## Подготовьте инфраструктуру

Необходимо настроить соответствующую инфраструктуру для агентов нагрузочного тестирования и целевых объектов, если она еще не настроена.

### Сервисные аккаунты

Для облачного агента потребуется сервисный аккаунт с ролью `loadtesting.generatorClient`. Один сервисный аккаунт можно использовать для нескольких агентов в одном каталоге.

Создайте сервисный аккаунт:

```bash

$ YC_LT_SERVICE_ACCOUNT_ID=$(yc iam service-account create --name sa-loadagent --format json | jq -r ".id")

```

Назначьте сервисному аккаунту роль:

```bash

$ FOLDER_ID=$(yc config get folder-id)
$ yc resource-manager folder add-access-binding $FOLDER_ID \
  --service-account-id $YC_LT_SERVICE_ACCOUNT_ID \
  --role loadtesting.generatorClient

```

### Сеть агента

Агенты нагрузочного тестирования должны иметь доступ к Yandex Cloud API Gateway из своей подсети.
По умолчанию в каждом каталоге создается одна сеть VPC с подсетями для всех зон.

#### Пример конфигурации сети с NAT Gateway и маршрутом к Yandex Cloud API Gateway

Подробную информацию см. в [статье о настройке NAT-шлюза](https://cloud.yandex.ru/ru/docs/vpc/operations/create-nat-gateway#console_1).

Убедитесь, что в каталоге настроена сеть:

```bash

$ YC_VPC_NETWORK_ID=$(yc vpc network list --format json | jq -r ".[0] | .id")

$ echo $YC_VPC_NETWORK_ID
enp29occlmph********

```

Создайте NAT-шлюз и настройте маршрут для доступа в интернет:

```bash

$ YC_NAT_GATEWAY_ID=$(yc vpc gateway create --name load-agents-gateway --format json | jq -r ".id")

$ yc vpc route-table create \
   --name=load-agents-route-table \
   --network-id=$YC_VPC_NETWORK_ID \
   --route destination=0.0.0.0/0,gateway-id=$YC_NAT_GATEWAY_ID

```

### Группы безопасности

Обычно агент нагрузочного тестирования и целевой объект развертываются в разных сетях. В таком случае необходимо настроить группы безопасности для разрешения трафика от агента к целевому объекту и от агентов к API Gateway.

Создайте новую группу безопасности для агента нагрузочного тестирования:

```bash

$ AGENT_SG_ID=$(yc vpc security-group create --format json \
  --name sg-load-testing-agents \
  --rule "direction=egress,protocol=tcp,v4-cidrs=[0.0.0.0/0],from-port=0,to-port=65535" \
  --network-id $YC_VPC_NETWORK_ID | jq -r ".id")

```

Создайте новую группу безопасности для целевого объекта тестирования с правилом, разрешающим весь трафик, поступающий от агентов:

```bash

$ yc vpc security-group create \
  --name sg-load-testing-targets \
  --rule "direction=ingress,protocol=any,security-group-id=$AGENT_SG_ID,from-port=0,to-port=65535" \
  --network-id $YC_VPC_NETWORK_ID

```

## Создайте облачного агента

Для начала выберите зону для развертывания агента. Обычно это та же зона, в которой развернут целевой объект.

```bash

$ LT_ZONE_ID=ru-central1-a

$ yc vpc network list-subnets $YC_VPC_NETWORK_ID --format json | jq ".[] | select(.zone_id == \"$LT_ZONE_ID\") | {\"id\":.id,\"name\":.name}"
{
  "id": "e9bt0v**************",
  "name": "lt-net-ru-central1-a"
}

$ SUBNET_ZONE_A_ID=e9bt0v**************

```

Теперь можно приступать к созданию нового агента:

```bash
$ yc loadtesting agent create \
  --name my-agent \
  --labels origin=default,label-key=label-value \
  --zone $LT_ZONE_ID \
  --network-interface subnet-id=$SUBNET_ZONE_A_ID,security-group-ids=$AGENT_SG_ID \
  --cores 2 \
  --memory 2G \
  --service-account-id $YC_LT_SERVICE_ACCOUNT_ID
```

См. [бенчмарк для агентов](https://cloud.yandex.ru/en/docs/load-testing/concepts/agent#benchmark), чтобы правильно настроить CPU и RAM агентов под ваши задачи.


## Доступ к агенту по SSH

Для подключения к ВМ агента по SSH настройте правило группы безопасности для входящего трафика на порт 22:

```bash

# use your v4-cidrs to limit ip addresses allowed to connect
$ yc vpc security-group update-rules $AGENT_SG_ID \
  --add-rule "direction=ingress,port=22,protocol=tcp,v4-cidrs=[0.0.0.0/24]"

```

При создании нового агента укажите в метаданных публичный SSH-ключ:

```bash

$ SSH_PUB_FILE_PATH=/path/to/ssh/key.pub
$ cat $SSH_PUB_FILE_PATH  # ensure file path is correct

$ SSH_USERNAME=agent-root
$ cat > agent-metadata-user-data.yaml <<EOF
#cloud-config
ssh_pwauth: 'no'
users:
- groups: sudo
  name: $SSH_USERNAME
  shell: /bin/bash
  ssh-authorized-keys:
  - $(cat $SSH_PUB_FILE_PATH)
  sudo:
  - ALL=(ALL) NOPASSWD:ALL
EOF


$ yc loadtesting agent create \
  --name my-agent \
  --zone $LT_ZONE_ID \
  --network-interface subnet-id=$SUBNET_ZONE_A_ID,security-group-ids=$AGENT_SG_ID \
  --cores 2 \
  --memory 2G \
  --service-account-id $YC_LT_SERVICE_ACCOUNT_ID \
  --metadata-from-file user-data=agent-metadata-user-data.yaml

```

Параметр метаданных `user-data` принимает [конфигурации `cloud-init`](https://cloudinit.readthedocs.io/en/latest/). Настройте ВМ агента с использованием дополнительного ПО.

Если у вас еще нет SSH-ключа, создайте его:

```bash

$ ssh-keygen -t ed25519

Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/username/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/username/.ssh/id_ed25519
Your public key has been saved in /home/username/.ssh/id_ed25519.pub

$ SSH_PUB_FILE_PATH=/home/username/.ssh/id_ed25519.pub

```

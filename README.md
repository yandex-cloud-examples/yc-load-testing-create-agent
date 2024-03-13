# Configure your cloud for load testing and create first agent

NOTE: In the snippets below, it is assumed that folder-id setting is already set in `yc` config:

```bash
yc config set folder-id 'my_folder_id'
```

if not, you may always pass `--folder-id my_folder_id` argument to `yc` commands.

## 1. Prepare infrastructure

You need to configure proper infrastructure for load testing agents and testing targets if you haven`t done it before.

### 1.1. Service account

For cloud agent you will need a service account with role `loadtesting.generatorClient`. One service account may be used with multiple agents within a single folder.

Create new service account:

```bash

$ YC_LT_SERVICE_ACCOUNT_ID=$(yc iam service-account create --name sa-loadagent --format json | jq -r ".id")

```

Add role to service account:

```bash

$ FOLDER_ID=$(yc config get folder-id)
$ yc resource-manager folder add-access-binding $FOLDER_ID \
  --service-account-id $YC_LT_SERVICE_ACCOUNT_ID \
  --role loadtesting.generatorClient

```

### 1.2. Agent network

For load testing agents to operate, it requires to have an access to Yandex Cloud API Gateway from agent subnet.
By default single VPC network with subnets for each zone is present in each folder.

#### Example of network configuration with NAT Gateway and route to Yandex Cloud API Gateway

See [NAT Gateway docs](https://cloud.yandex.ru/en/docs/vpc/operations/create-nat-gateway#console_1) for more details.

Ensure you have a network in the folder:

```bash

$ YC_VPC_NETWORK_ID=$(yc vpc network list --format json | jq -r ".[0] | .id")

$ echo $YC_VPC_NETWORK_ID
enp29occlmph********

```

Create NAT Gateway and route to Internet

```bash

$ YC_NAT_GATEWAY_ID=$(yc vpc gateway create --name load-agents-gateway --format json | jq -r ".id")

$ yc vpc route-table create \
   --name=load-agents-route-table \
   --network-id=$YC_VPC_NETWORK_ID \
   --route destination=0.0.0.0/0,gateway-id=$YC_NAT_GATEWAY_ID

```

### 1.3. Security groups

Typically, the load testing agent and test target are deployed in separate networks. If you do this, configure security groups to allow traffic from agent to target and from agents to API Gateway.

Create new security group for load testing agent:

```bash

$ AGENT_SG_ID=$(yc vpc security-group create --format json \
  --name sg-load-testing-agents \
  --rule "direction=egress,protocol=tcp,v4-cidrs=[0.0.0.0/0],from-port=0,to-port=65535" \
  --network-id $YC_VPC_NETWORK_ID | jq -r ".id")

```

Create new security group for test target with rule to allow all traffic from agents:

```bash

$ yc vpc security-group create \
  --name sg-load-testing-targets \
  --rule "direction=ingress,protocol=any,security-group-id=$AGENT_SG_ID,from-port=0,to-port=65535" \
  --network-id $YC_VPC_NETWORK_ID

```

## 2. Create Cloud Agent

First you need to pick a zone where agent should be provisioned. In general, you want the same zone where your target is deployed.

```bash

$ LT_ZONE_ID=ru-central1-a

$ yc vpc network list-subnets $YC_VPC_NETWORK_ID --format json | jq ".[] | select(.zone_id == \"$LT_ZONE_ID\") | {\"id\":.id,\"name\":.name}"
{
  "id": "e9bt0v**************",
  "name": "lt-net-ru-central1-a"
}

$ SUBNET_ZONE_A_ID=e9bt0v**************

```

Finally we can create new agent.

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

See [agents benchmark](https://cloud.yandex.ru/en/docs/load-testing/concepts/agent#benchmark) to adjust agents CPU and RAM for your needs.


## 3. SSH access to your agent

To access the load testing agent VM via ssh, you need a security group rule for incoming traffic on port 22:

```bash

# use your v4-cidrs to limit ip addresses allowed to connect
$ yc vpc security-group update-rules $AGENT_SG_ID \
  --add-rule "direction=ingress,port=22,protocol=tcp,v4-cidrs=[0.0.0.0/24]"

```

Also you need to specify your public ssh key in metadata when creating new agent:

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

The `user-data` metadata option accepts [cloud-init configs](https://cloudinit.readthedocs.io/en/latest/). Feel free to customize your agent VM with additional software.

If you don't have ssh key yet, create one:

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

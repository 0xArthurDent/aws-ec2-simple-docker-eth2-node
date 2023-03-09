# aws-ec2-simple-docker-eth2-node

An example of a simple eth2 node running on docker using Geth as execution client and Prysm as a Consensus client.

## Prerequisites

- Provision an AWS EC2 instance - an m5.xlarge instance was selected running Ubuntu LTS 22.04 x64. This will serve as the docker host to run the containers needed.
- a 1TB EBS volume, was used as a persisent data storage
- In terms of network security group the following rules were configured:

### Networking Inbound Rules

|Protocol|Port|Source| Comment |
|:---:|:---:|:---:|:---:|
| SSH | 22 | *admin ip*| Admin desktop |
| TCP | 30303 | 0.0.0.0/0 | Geth  |
| UDP | 30303 | 0.0.0.0/0 | Geth  |
| TCP | 13000 | 0.0.0.0/0 | Prysm |
| UDP | 12000 | 0.0.0.0/0 | Prysm |

### Networking Outbound Rules

|Protocol|Port|Destination| Comment |
|:---:|:---:|:---:|:---:|
| All | All | 0.0.0.0/0 | All traffic |

## Server Host Config

### General

```bash
sudo apt update -y
sudo apt upgrade -y

sudo apt -y install chrony

sudo apt-get clean
sudo apt-get autoclean
sudo apt-get autoremove
```

### Mounting EBS volume for persistent storage

```bash
#verify volume label is nvme1n1 - otherwise edit as necessary
lsblk

#Docker persistent Datastore
sudo mkdir /mnt/docker-datastore/
sudo mkfs -t ext4 /dev/nvme1n1
sudo mount -t ext4 /dev/nvme1n1 /mnt/docker-datastore

#Persist volumes mounting on reboots
echo "$(sudo blkid -o export /dev/nvme1n1 | grep ^UUID=) /mnt/docker-datastore ext4    defaults,noatime       0       1" | sudo tee -a /etc/fstab
```

### Create directories for container data

```bash
sudo mkdir /mnt/docker-datastore/geth
sudo mkdir /mnt/docker-datastore/geth/.ethereum
sudo mkdir /mnt/docker-datastore/prysm
```

### Set Swappines to 1

```bash
sudo vim /etc/sysctl.conf

#Scroll to the bottom of this file and add

vm.swappiness=1

#Close and save. Then load the new value with

sudo sysctl -p
```

### Installing Docker

```bash
sudo apt  -get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    tree \
    sysstat

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io


#Check docker version and docker service is running
docker -v
sudo service docker status

#Include user (change accoridngly) on docker set and execute command to avoid having to use sudo. restart vm after running command
sudo usermod -a -G docker Your_user
exit

```

### Optional: Install CTop

```bash
echo "deb http://packages.azlux.fr/debian/ buster main" | sudo tee /etc/apt/sources.list.d/azlux.list
wget -qO - https://azlux.fr/repo.gpg.key | sudo apt-key add -
sudo apt update
sudo apt install -y docker-ctop
```

### Optional: Install Sysstat

```bash
sudo apt -y install sysstat
```

## Prepare for node sync

Before running the containers, edit the docker-compose.yml and insert a wallet address for the consensus container (see line: - --suggested-fee-recipient=0x********************* )

Download the gensis file from [Prysm github](https://github.com/eth-clients/merge-testnets/tree/main/sepolia) and place it in the same directory as specified in the docker-compose.yml  (see line: - --genesis-state=/data/genesis.ssz )

```bash
sudo wget -P /mnt/docker-datastore/prysm/genesis.ssz https://github.com/eth-clients/merge-testnets/blob/main/sepolia/genesis.ssz
```

## Run containers

Start containers

```bash
docker compose up
```

Check that node has synced. example command to execute command within docker container

```bash
docker exec geth-sepolia geth --exec 'eth.blockNumber' attach ipc:/root/.ethereum/sepolia/geth.ipc
```

### Optional: > Run Ctop

```bash
docker run --rm -ti \
  --name=ctop \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  quay.io/vektorlab/ctop:latest
```

### Optional: > Run IOStat

```bash
iostat -xt 10
```


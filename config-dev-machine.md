# Create Dev Machine

1. Resource Group

```
  az group create --name funkydev --location centralus
```

2. Developer Machine

```
# Create SSH keys (we use files, feel free to put them on your ssh-agent)

  ssh-keygen -t rsa -b 2048 -C "funkydev@funky.com" -f ./funky

# Create VM
az vm create --name funkydev00 --resource-group funkydev --public-ip-address-dns-name funkydev00 --image ubuntults --data-disk-sizes 1024 --size Standard_DS4_v2 --ssh-key-value ./funky.pub
```

## notes on developer machine

1. We are using a fairly large machine because building Kubernetes is a CPU intensive process.
2. We added another data disk because docker builds/drops will eat the OS disk quickly. 

# Configure Developer Machine

1. Connect 

```
  ssh -i ./funky funkydev00.centralus.cloudapp.azure.com
```

2. Format the data disk 

```
# Find the data disk i(Since it is a new machine it will probably take /dev/sdc slot)
lsblk 

# Format it
 sudo mkfs.ext4 /dev/sdc

# Create mount point
sudo mkdir /mnt/d
sudo chown $(whoami):$(whoami) /mnt/d
sudo chmod +rw /mnt/d

# Configure auto mount (modify /dev/sdc if your disk is not attached there)
cat << EOF | sudo tee /etc/systemd/system/mnt-d.mount
[Unit]
Description=Mount Data Disk

[Mount]
What=/dev/sdc
Where=/mnt/d
Type=ext4

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable mnt-d.mount

# manually start the mount point
sudo systemctl start mnt-d.mount

```

3. Install Docker & Build Essentials 

```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y docker.io 

sudo apt-get install -y build-essential 
```

4. Configure Docker to use the data disk

```
mkdir /mnt/d/docker-pwd
sudo mkdir /etc/systemd/system/docker.service.d 

cat << EOF | sudo tee /etc/systemd/system/docker.service.d/graph.conf
[Service]
ExecStart=
ExecStart=/usr/bin/docker daemon -H fd:// --graph="/mnt/d/docker-pwd"
EOF

# Restart the service (allow docker sometime before starting again, do not trust systemd to do it)
sudo systemctl daemon-reload
sudo systemctl stop docker.service && sleep 3 && sudo systemctl start docker.service 
```

5. Configure Docker Hub Login

```

#Login to docker (follow the prompts)
sudo docker login

#as you build you will generate docker images, and automatically pushed to your repo

```

6. Configure Docker to work without sudo 

```
sudo usermod -a -G docker ${USER}
# relogin after this command 

# you may encounter premission issues on conf files, if so do the following
sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
sudo chmod g+rwx "/home/$USER/.docker" -R
```

7. Install Go

```
$(cd /tmp && wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz)
sudo tar -C /usr/local -xzf /tmp/go1.8.3.linux-amd64.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin" | sudo tee /etc/profile
echo "export PATH=$PATH:~/go/bin" | tee ~/.bashrc
source /etc/profile
source ~/.bashrc
```


# Prepare for Kubernetes 

1. Fork Kubernetes Repo

2. Clone Kubernetes Repo

```
cd ~
mkdir -p go/src/k8s.io/

# clone it 
git clone https://github.com/kubernetes/kubernetes.git

# add your fork as a remote 
git remote add fork <<your fork here>>

```

3. Create a test Build

```
cd ~/go/src/k8s.io/kubernetes
export REGISTRY=<<YOUR DOCKER USERNAME>>
export VERSION=<<your dev image name>> # this will be <<register>>:hyperkube-amd64:<<VERSION>>
./hack/dev-push-hyperkube.sh
# the output is a hyperkube image on docker hub with <<REGISTRY>>:hyperkube-amd64:<<VERSION>>
# takes a good 2+ mins to complete.
```

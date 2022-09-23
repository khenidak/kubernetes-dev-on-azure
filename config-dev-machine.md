
# Create Dev Machine

1. Resource Group
```
  az group create --name funkydev --location centralus
```

#### Notes on subscriptions

If you have more than one subscription, then please use the following command to figure out if its default subscription:
```
  az account show
```
To get a list of subscriptions use:
```
  az account list
```
and finally, to set the right subscription as default:
```
  az account set -s <subscription id>
```

2. Developer Machine

```
# Create SSH keys (we use files, feel free to put them on your ssh-agent)

  ssh-keygen -t rsa -b 2048 -C "funkydev@funky.com" -f ./funky

# Create VM
az vm create --name funkydev00 --resource-group funkydev --public-ip-address-dns-name funkydev00 --image ubuntults --data-disk-sizes 1024 --size Standard_DS4_v2 --ssh-key-value ./funky.pub
```

#### Notes on developer machine

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

# Create mount point (Don't use /mnt since it's the temporary storage on Azure VM)
sudo mkdir -p /mount/d

# Find the UUID of the storage device, since /dev/sdc may change drive letters on reboot
sudo blkid | grep UUID=

# Configure auto mount (modify the disk device to include the UUID from above)
cat << EOF | sudo tee /etc/systemd/system/mount-d.mount
[Unit]
Description=Mount Data Disk

[Mount]
What=/dev/disk/by-uuid/<UUID>
Where=/mount/d
Type=ext4

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable mount-d.mount

# manually start the mount point
sudo systemctl start mount-d.mount

# After mounting the disk, we need to change the owner:group of the directory:
sudo chown $(whoami):$(whoami) /mount/d
sudo chmod +rw /mount/d
```

3. Install Docker & Build Essentials 

```
sudo apt-get update
sudo apt-get upgrade -y
```

Install docker using the steps from https://docs.docker.com/install/linux/docker-ce/ubuntu/

```
sudo apt-get install -y build-essential 
```

4. Configure Docker to use the data disk

```
mkdir /mount/d/docker-pwd
sudo mkdir /etc/systemd/system/docker.service.d 

cat << EOF | sudo tee /etc/systemd/system/docker.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --data-root="/mount/d/docker-pwd"
EOF

# Restart the service (allow docker sometime before starting again, do not trust systemd to do it)
sudo systemctl daemon-reload

sudo systemctl restart docker.service
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
$(cd /tmp && wget https://storage.googleapis.com/golang/go1.10.linux-amd64.tar.gz)
sudo tar -C /usr/local -xzf /tmp/go1.10.linux-amd64.tar.gz
echo "export PATH=\$PATH:/usr/local/go/bin" | sudo tee --append /etc/profile
echo "export PATH=\$PATH:~/go/bin" | tee --append ~/.bashrc
source /etc/profile
source ~/.bashrc
```


# Prepare for Kubernetes 

1. Fork the Kubernetes repo in GitHub.

2. Clone your fork of the Kubernetes repo

```
cd ~
mkdir -p go/src/k8s.io/

# clone your fork as origin
cd ~/go/src/k8s.io
git clone https://github.com/<your_github_username>/kubernetes.git
```

3. Set upstream to the main Kubernetes repo.

```
cd kubernetes
# add upstream repo 
git remote add upstream https://github.com/kubernetes/kubernetes.git
```

> NOTE
> You don't have to use origin/upstream as described here, but it's a common naming
> convention for GitHub workflows.

4. Create a test build.

```
cd ~/go/src/k8s.io/kubernetes
export REGISTRY=<<YOUR DOCKER USERNAME>>
export VERSION=<<your dev image name>>  # will become version part of image tag
./hack/dev-push-hyperkube.sh
# the output is a hyperkube image on docker hub with <<REGISTRY>>:hyperkube-amd64:<<VERSION>>
# takes a good 2+ mins to complete.
```

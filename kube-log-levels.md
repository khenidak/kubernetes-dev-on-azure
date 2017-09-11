# Changing Kubernetes log levels

ACS Engine defaults to log level 2 (or 4 for apiserver).  When debugging it's often useful to change this to show more detail.  To do this:

* `ssh` into the master VM
  * E.g. `ssh -i .ssh/id_rsa azureuser@mycluster.westus.cloudapp.azure.com`
* Change to the `/etc/kubernetes/manifests` directory.
  * This will contain files named `kube-xyz.yaml` which are the config files for the various Kubernetes components, e.g. apiserver, controller-manager...
* Edit the config file for each component where you want to see more detailed logs.
  * The file will include a line of the form `- "--v=2"` (under `spec/containers/command`).
  * Change the number to the desired log level.
  * Save the file.
* Once you save the file, kubelet will restart the relevant component with the new spec.

## Helper script

You can create a helper script to automate the process of ssh-ing in and editing the config file.  Here's an example.

```
#! /bin/bash

if [ $# -ne 4 ]; then
  echo "./loglevel.sh n cluster region {addon-manager|apiserver|controller-manager|scheduler}"
  exit 1
fi

lvl=$1
clus=$2
rgn=$3
role=$4

ssh -i .ssh/id_rsa azureuser@${clus}.${rgn}.cloudapp.azure.com "sudo sed -i \"s|- \\\"--v=[0-9]\\+\\\"|- \\\"--v=${lvl}}\\\"|g\" /etc/kubernetes/manifests/kube-${role}.yaml"
```

(Note: replace the SSH key file with the one containing your own private key.)

Example usage: `./loglevel.sh 5 mycluster australiasoutheast apiserver`

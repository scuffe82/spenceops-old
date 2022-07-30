# spenceops

## Requirments
1+ Ubuntu 22.04 servers
Some sort of storage solution. I use both the Synology iSCSI CSI Driver and the NFS Sub-directory provider
DNS for deployments that require ingress through traefik
A git repo to push flux configs to
Ansible installed
kubectl installed
Other tools below have some install examples, see the tools homepage for additional install details for other OS's

## A note on security
* The .gitignore file will exclude known secrets from being pushed into your repo.
* Use SOPS to encrypt secrets that need to be pushed into the repo
* TODO: Pre-commit hooks to check for secrets/keys

## SOPS/AGE Setup

Install Sops and Age for your OS:

* OSX/Homebrew:
```
brew install sops age
```

* Ubuntu:
```
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.7.1/sops-v3.7.1.linux -o /usr/local/bin/sops && sudo chmod +x /usr/local/bin/sops

curl -LO https://github.com/FiloSottile/age/releases/download/v1.0.0/age-v1.0.0-linux-amd64.tar.gz && sudo tar -C /usr/local/bin --strip-components 1 -xf age-v1.0.0-linux-amd64.tar.gz && rm age-v1.0.0-linux-amd64.tar.gz
```

## Install Task

* OSX/Homebrew:
```
brew install go-task/tap/go-task
```

* Ubuntu/Linux:
```
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d
```

## Generate Age key for SOPS
Run the following command to generate a public/private key pair for sops with age
```
age-keygen -o ./config/sops/keys.txt
```

Setup the sops config by:

* Copy ./config/sops/sops-config.yaml.exmple to ./.sops.yaml
* Add your public Age key into the file on the last line <YOUR PUBLIC AGE KEY>
* Copy ./config/sops/sops-secret.yaml.example tp ./config/sops/sops-secret.yaml
* Base64 encode your age key file by running
```
cat ./config/sops/keys.txt | base64
```
* Take the output vaule and replace <YOUR keys.txt BASE64 VAULE> in the ./config/sops/sops-secret.yaml file
  
## Setup Flux SSH keys
* Copy ./provisioners/flux/clusters/flux-system/flux-secret.yaml.example to ./provisioners/flux/clusters/flux-system/flux-secret.yaml
* Base64 encode your SSH identity both id_rsa and id_rsa.pub and replace the vaules in the file
* If you are using a git provider other than public github replace the known_hosts: vaule with the SSH identity of your git provider. For example to scan github.com and get the proper key you would run
```
ssh-keyscan -t ecdsa-sha2-nistp256 github.com
```
The -t flag needs to be set to whatever the supported cipher of your host is

## Ansible inventory and variables
Update the Ansible inventory to your server list. The current setup is for 1+ nodes where the master is untainted and will run workloads.
* ./providers/ansible/inventory/hosts.yaml contains the server list.
* Add all servers in the All section
* Add the server that you want to be the master to the master section
* Add all other servers to the workers section
* In the ./providers/ansible/inventory/host_vars directory create a file for each of your servers containing servername: YOURNAME
* In the ./provisioners/ansible/inventory/group_vars/all file edit the parameters for your environment.

## Ansible user setup
Generate ssh-key for the ansible user in ./provisioners/ansible/ansible_user_setup/files named ansible-user and ansible-user.pub then run

```
ansible-playbook ./provisioners/ansible/ansible_user_setup/setup_ansible_user.yaml --ask-pass --ask-become-pass
``` 

This will create the ansible user on the inventory servers with the ssh key you created above. The user is allowed to sudo without auth so it can setup the system automatically. 


## Now to use all this stuff
Before you deploy the cluster you need to push everything to your git repo. Once uploaded update the ./provisioners/flux/clusters/flux-system/gotk-sync.yaml file with the URL of your git repo.
* NOTE: Flux does not like git URL's that have a ":" in the URL so if your using a provider that puts one in there (like bitbucket) try replacing it with a "/" and see if you can still clone your repo.

Run the following commands to setup the cluster
* ``` task ansible:list ``` to check that your ansible inventory is configured for your hosts
* ``` task ansible:ping ``` to make sure ansible can talk to all your servers
* ``` task ansible:prep-ubuntu ``` to install all the packages and make OS tweaks for k8s
* ``` task ansible:k8s-install ``` to install k8s and setup flux
* ``` task ansible: disable-kube-proxy ``` This disables the kube-proxy daemon set from running since Calico is deployed in eBFP mode

At this point the cluster is bootstraping with all of the included apps. Once its up there will be a kubeconfig file located in ./config/CLUSTER-NAME/kubeconfig to access the cluster with.

## Notes
* The charts and docs directories are not needed for the project and in the repo for a helm repo and for github pages.
* This is a work in progress/living project that i mess with in my spare time.
* The .in and .out file are for autoenv in zsh, if you're running OSX you will need to export the .in file to make sure the parameter is set for ansible.
# spenceops
Ansible user setup
generate ssh-key for the ansible user in ./provisioners/ansible/ansible_user_setup/files named ansible-user
Run
```
ansible-playbook ./provisioners/ansible/ansible_user_setup/setup_ansible_user.yaml --ask-pass --ask-become-pass
``` 
This will create the ansible user on the inventory servers with the ssh key you created above. The user is allowed to sudo without auth so it can setup the system automatically. 


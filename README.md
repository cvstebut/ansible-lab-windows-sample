# Manage Windows 10 Client with Ansible - WinRM & Domain account using CredSSP

## Links
- [Installation of OpenSSH For Windows Server 2019 and Windows 10](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)
- [docs.ansible - Windows SSH Setup](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html#windows-ssh-setup)
- [chocolatey.org - openssh beta](https://chocolatey.org/packages/openssh/8.1.0-beta)
- [docs.ansible - Windows WinRm setup](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html)

### Encryption in Ansible using ansible-vault
- [duffney.io - Secure Group Variables with Ansible Vault](https://duffney.io/secure-group-variables-with-ansible-vault/)
- [stackoverflow - Encrypt a single variable](https://stackoverflow.com/a/44241343)
- [docs.ansible - Running a Playbook with Vault](https://docs.ansible.com/ansible/2.5/user_guide/playbooks_vault.html)
- [github ansible issues - Use lookups in group var files with `set_fact`](https://github.com/ansible/ansible/issues/52290)


password file `~/.ci/ansible/vault-password-minikube` containing the vault password in clear text. 

Stored in a secure location apart from git repo!

ansible.cfg
```ini
[defaults]
roles_path=../roles/internal:../roles/external
inventory=../inventory/cvs004
vault_password_file=~/.ci/ansible/vault-password-minikube
```


## Installation and Configuration on Windows 10

### Configure winrm

`ConfigureRemotingForAnsible.ps1`

```powershell
./ConfigureRemotingForAnsible.ps1 -verbose
```

### Enable CredSSP

```powershell
Enable-WSManCredSSP -Role Server -Force
```

## Ansible Configuration on Linux Control Host

`./plays/ansible.cfg`
```ini
[defaults]
roles_path=../roles/internal:../roles/external
inventory=../inventory/cvs004
vault_password_file=~/.ci/ansible/vault-password-minikube
```

`./inventory/cvs004/hosts`
```yaml
cvs004 ansible_host=192.168.2.121

[windows]
cvs004 ansible_host=192.168.2.121
```

`./inventory/cvs004/group_vars/windows/windows_cleartext.yml`
```yaml
# domain user e.g.: user@mydomain.de
ansible_user: "{{ vault_ansible_user }}"
ansible_connection: winrm
ansible_winrm_transport: credssp
ansible_winrm_server_cert_validation: ignore
ansible_password: "{{ vault_ansible_password }}"
```

Playbook `./plays/sample.yml`

```yaml
---
- hosts: cvs004
  vars:
    MINIKUBE_PROFILE: minikube02
  tasks:

  - name: check for existence of minikube cluster {{ MINIKUBE_PROFILE}}
    win_command: minikube status -p {{ MINIKUBE_PROFILE}}
    register: minikube_out
    changed_when: false
    failed_when: false

  - name: minikube_out
    debug:
      msg: "{{ minikube_out.stdout }}"

  - name: minikube_out stdout_lines[0]
    debug:
      msg: "{{ minikube_out.stdout_lines[0] }}"

  - name: Should print if {{ MINIKUBE_PROFILE}} is present
    debug:
      msg: "Found: {{ MINIKUBE_PROFILE }}"
    when: minikube_out.stdout_lines[0] == MINIKUBE_PROFILE

  - name: Should print if {{ MINIKUBE_PROFILE}} is not present
    debug:
      msg: "Not Found: {{ MINIKUBE_PROFILE }}"
    when: minikube_out.stdout_lines[0] != MINIKUBE_PROFILE    
```

```console
./plays$ ansible-playbook sample.yml 
```

```log

PLAY [cvs004] ***************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************
ok: [cvs004]

TASK [check for existence of minikube cluster minikube02] *******************************************************************************
ok: [cvs004]

TASK [minikube_out] *********************************************************************************************************************
ok: [cvs004] => {
    "msg": "minikube02\ntype: Control Plane\nhost: Running\nkubelet: Running\napiserver: Running\nkubeconfig: Configured\n\n"
}

TASK [minikube_out stdout_lines[0]] *****************************************************************************************************
ok: [cvs004] => {
    "msg": "minikube02"
}

TASK [Should print if minikube02 is present] ********************************************************************************************
ok: [cvs004] => {
    "msg": "Found: minikube02"
}

TASK [Should print if minikube02 is not present] ****************************************************************************************
skipping: [cvs004]

PLAY RECAP ******************************************************************************************************************************
cvs004                     : ok=5    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```
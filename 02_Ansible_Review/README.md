# Basic Overview of Ansible

## First Commands

Edit (or create) `/etc/ansible/hosts` and put one or more remote systems in it. Use `localhost` as a starting point:
```
localhost
```

This is the inventory file which describes the list of hosts that Ansible will interact with. In order for Ansible to connect to this host, we must enable passwordless SSH. To do this, ensure that your public SSH key is in `~/.ssh/authorized_keys`. If it is not, add it:
```bash
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

To enable passwordless SSH:
```bash
$ ssh-agent bash
$ ssh-add ~/.ssh/id_rsa
```

Now test that Ansible is properly configured:
```bash
$ ansible all -m ping
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Now that we have confirmed Ansible is properly configured, let's look at Ansible Playbooks.

## Working with Ansible Playbooks

Playbooks are Ansible’s configuration, deployment, and orchestration language. Inside of playbooks a developer can declare a set of tasks or Ansible Roles to be executed on launching the playbook.

For simplicity, let's start with declaring the tasks directly in the playbook. Start by creating and editing `playbook.yaml` with a debug statement to output information from our host:
```yaml
- hosts: localhost
  tasks:
    - debug:
        msg: "System {{ inventory_hostname }} has uuid {{ ansible_product_uuid }}"
```

Run the playbook:
```bash
$ ansible-playbook playbook.yaml
---- (omitted output) ----
TASK [debug] ****************************
ok: [localhost] => {
    "msg": "System localhost has uuid NA"
}
```

At a basic level, Playbooks can be used to manage configurations and deployments to remote machines. Putting all of our orchestration logic inside of a single playbook file can be cumbersome, which is why it is recommended to separate our logic into multiple Ansible Roles. Let's move our tasks inside of a role.

## Creating an Ansible Role

Ansible Roles are a way of automatically loading certain vars_files, tasks, and handlers based on a known file structure. Grouping content by roles also allows easy sharing of roles with others. Roles are far more consumable than playbooks.

Now that we have seen how tasks are executed inside of playbooks, let's move the same logic inside of an Ansible Role. Begin by creating a new role using `ansible-galaxy`:
```
$ ansible-galaxy init example-role
- example-role was created successfully
```

This generates a skeleton Role which can be easily modified by editing the tasks inside of `example-role/tasks/main.yml`. For a simple example, let's modify the tasks file to print out information about our host as we did in the previous example:
```yaml
- debug:
    msg: "System {{ inventory_hostname }} has uuid {{ ansible_product_uuid }}"
```

## Consuming Ansible Roles from within Ansible Playbook

We can now consume this role inside of `playbook.yaml` instead of declaring the tasks inside of our playbook. Modify `playbook.yaml` to match the following:
```yaml
- hosts: localhost
  roles:
    - example-role
```

Since `example-role` exists as a directory in the same path as `playbook.yaml`, we can execute the role from our playbook. If we were calling this playbook from a different directory we would have to ensure that the role exists in the default Ansible Roles path or provide the fully qualified path inside of our playbook.

Run the playbook:
```bash
$ ansible-playbook playbook.yaml
---- (omitted output) ----
TASK [debug] ****************************
ok: [localhost] => {
    "msg": "System localhost has uuid NA"
}
```

You should observe that the output remains the same.

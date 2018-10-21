# Ansible Kubernetes Modules

## Getting started with the k8s Ansible modules

Since we are interested in using Ansible for the lifecycle management of our
application on Kubernetes, it is beneficial for a developer to get a good grasp
of the [k8s Ansible module][k8s_ansible_module]. This Ansible module allows a
developer to either leverage their existing Kubernetes resource files (written
in YaML) or express the lifecycle management in native Ansible. One of the
biggest benefits of using Ansible in conjunction with existing Kubernetes
resource files is the ability to use Jinja templating so that you can customize
deployments with the simplicity of a few variables in Ansible.

The easiest way to get started is to install the modules on your local machine
and test them using a playbook.

### Installing the k8s Ansible modules

To install the k8s Ansible modules, one must first install Ansible 2.6+. On
Fedora/Centos:
```bash
$ sudo dnf install ansible
```

In addition to Ansible, a user must install the [Openshift Restclient
Python][openshift_restclient_python] package. This can be installed from pip:
```bash
$ pip install openshift
```

### Running the k8s Ansible modules locally

Modify `example-role/tasks/main.yml` with desired Ansible logic. For this example
we will create and delete a namespace with the switch of a variable:
```yaml
---
- name: set test namespace to {{ state }}
  k8s:
    api_version: v1
    kind: Namespace
    state: "{{ state }}"
  ignore_errors: true
```
**note**: Setting `ignore_errors: true` is done so that deleting a nonexistent
project doesn't error out.

Modify `example-role/defaults/main.yml` to set `state` to `present` by default.
```yaml
---
state: present
```

Run the playbook:
```bash
$ ansible-playbook playbook.yaml
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'


PLAY [localhost] ***************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [localhost]

Task [example-role : set test namespace to present]
changed: [localhost]

PLAY RECAP *********************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```

Check that the namespace was created:
```bash
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    28d
kube-public   Active    28d
kube-system   Active    28d
test          Active    3s
```

Rerun the playbook setting `state` to `absent`:
```bash
$ ansible-playbook playbook.yml --extra-vars state=absent
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'


PLAY [localhost] ***************************************************************************

TASK [Gathering Facts] *********************************************************************
ok: [localhost]

Task [example-role : set test namespace to absent]
changed: [localhost]

PLAY RECAP *********************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```

Check that the namespace was deleted:
```bash
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    28d
kube-public   Active    28d
kube-system   Active    28d
```

## Leveraging existing Kubernetes Resource Files inside of Ansible

It may be easier to leverage existing Kubernetes Resource files to get started.
Take the [nginx deployment
example](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)
available in the Kubernetes docs. Starting with a clean Ansible Role
`example-role`, copy `nginx-deployment.yaml` into `example-role/files`:
```bash
$ cat example-role/files/nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
```
Modify `example-role/tasks/main.yml`:
```yaml
---
- k8s:
    state: present
    definition: "{{ lookup('file', 'nginx-deployment.yaml') }}"
    namespace: default
```

Run the playbook and observe 3 nginx replicas created from the above
deployment.

[k8s_ansible_module]:https://docs.ansible.com/ansible/2.6/modules/k8s_module.html
[openshift_restclient_python]:https://github.com/openshift/openshift-restclient-python

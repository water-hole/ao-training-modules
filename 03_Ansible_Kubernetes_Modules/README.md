# Ansible Kubernetes Modules

Using Ansible to manage workloads on Kubernetes is easy and natural, even if
you are new to Ansible.

## Goals

By the end of this module, you will understand:
* How to create and remove resources in Kubernetes
* How to reuse an existing Kubernetes manifest file with Ansible

**Note**: This guide assumes [minikube][minikube_tool] has been installed and
is running a local kubernetes cluster.

## Getting started with the k8s Ansible modules

Since we are interested in using Ansible for the lifecycle management of our
application on Kubernetes, it is beneficial for a developer to get a good grasp
of the [k8s Ansible module][k8s_ansible_module]. This Ansible module allows a
developer to either leverage their existing Kubernetes resource files (written
in YaML) or express the lifecycle management in native Ansible. One of the
biggest benefits of using Ansible in conjunction with existing Kubernetes
resource files is the ability to use Jinja templating so that you can customize
deployments with the simplicity of a few variables in Ansible.

### Running the k8s Ansible modules locally

Modify `example-role/tasks/main.yml` with desired Ansible logic. For this example
we will create and delete a namespace with the switch of a variable:
```yaml
---
- name: set test namespace to {{ state }}
  k8s:
    api_version: v1
    kind: Namespace
    name: test
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
$ ansible-playbook -i inventory playbook.yaml

PLAY [localhost] **********************************************************

TASK [Gathering Facts] ****************************************************
ok: [localhost]

Task [example-role : set test namespace to present]
changed: [localhost]

PLAY RECAP ****************************************************************
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
$ ansible-playbook -i inventory playbook.yaml --extra-vars state=absent

PLAY [localhost] **********************************************************

TASK [Gathering Facts] ****************************************************
ok: [localhost]

Task [example-role : set test namespace to absent]
changed: [localhost]

PLAY RECAP ****************************************************************
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

Now, we can leverage existing Kubernetes Resource files. Let us take the [nginx deployment
example](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)
from the Kubernetes docs.

Copy `nginx-deployment.yaml` into `example-role/files`:

```bash
$ curl https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/controllers/nginx-deployment.yaml -o example-role/files/nginx-deployment.yaml
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

Update `example-role/tasks/main.yml`:
```yaml
---
- name: set test namespace to {{ state }}
  k8s:
    api_version: v1
    kind: Namespace
    name: test
    state: "{{ state }}"

- name: set nginx deployment to {{ state }}
  k8s:
    state: "{{ state }}"
    definition: "{{ lookup('file', 'nginx-deployment.yaml') }}"
    namespace: test
```

Run the playbook:
```bash
$ ansible-playbook -i inventory playbook.yaml

PLAY [localhost] **********************************************************

TASK [Gathering Facts] ****************************************************
ok: [localhost]

TASK [example-role : set test namespace to present] ***********************
changed: [localhost]

TASK [example-role : set nginx deployment to present] *********************
changed: [localhost]

PLAY RECAP ****************************************************************
localhost                  : ok=3    changed=2    unreachable=0    failed=0
```

You can see the `test` namespace created and the `nginx` deployment created in the new
namespace.
```bash
$ kubectl get all -n test
NAME                                    READY     STATUS              RESTARTS   AGE
pod/nginx-deployment-86d59dd769-7flft   0/1       ContainerCreating   0          7s
pod/nginx-deployment-86d59dd769-7ptcg   0/1       ContainerCreating   0          7s
pod/nginx-deployment-86d59dd769-h5qrb   0/1       ContainerCreating   0          7s

NAME                               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3         3         3            0           7s

NAME                                          DESIRED   CURRENT   READY     AGE
replicaset.apps/nginx-deployment-86d59dd769   3         3         0         7s
```

And just like before we can take it all down by simply running our playbook with `state=absent`:
```bash
$ ansible-playbook -i inventory playbook.yaml -e state=absent

PLAY [localhost] **********************************************************

TASK [Gathering Facts] ****************************************************
ok: [localhost]

TASK [example-role : set test namespace to absent] ************************
changed: [localhost]

TASK [example-role : set nginx deployment to absent] **********************
changed: [localhost]

PLAY RECAP ****************************************************************
localhost                  : ok=3    changed=2    unreachable=0    failed=0
```

[minikube_tool]:https://github.com/kubernetes/minikube#installation
[k8s_ansible_module]:https://docs.ansible.com/ansible/2.6/modules/k8s_module.html
[openshift_restclient_python]:https://github.com/openshift/openshift-restclient-python

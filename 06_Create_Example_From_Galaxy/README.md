# Creating Example leveraging existing content from Ansible Galaxy

**Note**: This guide assumes [minikube][minikube_tool] has been installed and
is running a local kubernetes cluster. Guide also assumes access to quay.io for
publishing images to a registry.

This module will show the reader how to create a Memcached Operator by either
installing the Ansible Role from Ansible Galaxy or developing it from scratch.

## Create a new project

To get started, use the CLI to create a new Ansible-based memcached-operator
project:

```sh
$ operator-sdk new memcached-operator --api-version=cache.example.com/v1alpha1 --kind=Memcached --type=ansible
$ cd memcached-operator
```

This creates the memcached-operator project specifically for watching the
Memcached resource with APIVersion `cache.example.com/v1apha1` and Kind
`Memcached`.


## Customize the operator logic

For this example the memcached-operator will execute the following
reconciliation logic for each `Memcached` Custom Resource (CR):
- Create a memcached Deployment if it doesn't exist
- Ensure that the Deployment size is the same as specified by the `Memcached`
CR

## Installing an existing role from Ansible Galaxy

To speed things up, we can reuse a role that has been written inside of our
operator. The role we will use is
[dymurray.memcached_operator_role][memcached_galaxy]. The section below will
also show the reader how to create that Ansible Role from scratch. To get
started, install the Ansible Role inside of the project:
```bash
$ ansible-galaxy install dymurray.memcached_operator_role -p ./roles
$ ls roles/
dymurray.memcached_operator_role Memcached
$ rm -rf ./roles/Memcached # Delete generated scaffolding role
```

This role provides the user with a variable `size` which is an integer to
control the number of replicas to create. You can find the default for this
variable in `roles/dymurray.memcached_operator_role/defaults/main.yml`:
```bash
$ cat roles/dymurray.memcached_operator_role/defaults/main.yml
---
# defaults file for Memcached
size: 1
```

The reader can also take note of the tasks file which uses the Kubernetes
Ansible module to create a deployment of memcached if it does not exist. Again,
see below for a deep dive into each portion of the role.

It is important that we modify the necessary files to ensure that our operator
is using this role instead of the generated scaffolding role. First, lets
modify `watches.yaml`.

### Watches File

By default, the memcached-operator watches `Memcached` resource events as shown
in `watches.yaml` and executes Ansible Role `Memached`. Since we have changed
this role, lets change it to:

```yaml
---
- version: v1alpha1
  group: cache.example.com
  kind: Memcached
  role: /opt/ansible/roles/dymurray.memcached_operator_role
```

## Building the Memcached Ansible Role from scratch

The below guide is how to create
[dymurray.memcached_operator_role][memcached_galaxy] from scratch.

The first thing to do is to modify the generated Ansible role under
`roles/Memcached`. This Ansible Role controls the logic that is executed when a
resource is modified.

### Define the Memcached spec

Users provide input to an operator by placing descriptive data in the `spec`
field of a Custom Resource. In the case of an Ansible Operator, any key value
pairs listed in the Custom Resource `spec` field will be passed into Ansible as
[variables](https://docs.ansible.com/ansible/2.5/user_guide/playbooks_variables.html#passing-variables-on-the-command-line).
It is recommended that you perform some type validation in Ansible on the
variables to ensure that your application is receiving expected input.

Set a default size for Memcached, in case the user doesn't provide it in the
`spec` field, by modifying `roles/Memcached/defaults/main.yml`:
```yaml
size: 1
```

### Defining the Memcached deployment

Now that we have the spec defined, we can define which Ansible tasks are
executed on resource changes. Since this is an Ansible Role, the default
behavior will be to execute the tasks in `roles/Memcached/tasks/main.yml`. We
want Ansible to create a deployment if it does not exist which runs the
`memcached:1.4.36-alpine` image. Ansible 2.5+ supports the [k8s Ansible
Module](https://docs.ansible.com/ansible/2.6/modules/k8s_module.html) which we
will leverage to control the deployment definition.

Modify `roles/Memcached/tasks/main.yml` to look like the following:
```yaml
---
- name: start memcached
  k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ meta.name }}-memcached'
        namespace: '{{ meta.namespace }}'
      spec:
        replicas: "{{ size }}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211

```

It is important to note that we used the `size` variable to control how many
replicas of the Memcached deployment we want. We set the default to `1`, but
any user can create a Custom Resource that overwrites the default.

## Build and run the operator

Before running the operator, Kubernetes needs to know about the new custom
resource definition the operator will be watching.

Deploy the CRD:

```sh
$ kubectl create -f deploy/crds/cache_v1alpha1_memcached_crd.yaml
```

Once this is done, there are two ways to run the operator:

- As a pod inside a Kubernetes cluster
- As a go program outside the cluster using `operator-sdk`

### 1. Run as a pod inside a Kubernetes cluster

Running as a pod inside a Kubernetes cluster is preferred for production use.

Build the memcached-operator image and push it to a registry:
```
$ operator-sdk build quay.io/example/memcached-operator:v0.0.1
$ docker push quay.io/example/memcached-operator:v0.0.1
```

Kubernetes deployment manifests are generated in `deploy/operator.yaml`. The
deployment image in this file needs to be modified from the placeholder
`REPLACE_IMAGE` to the previous built image. To do this run:
```
$ sed -i 's|REPLACE_IMAGE|quay.io/example/memcached-operator:v0.0.1|g' deploy/operator.yaml
```

Deploy the memcached-operator:

```sh
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
$ kubectl create -f deploy/operator.yaml
```

Verify that the memcached-operator is up and running:

```sh
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           1m
```

### 2. Run outside the cluster

This method is preferred during the development cycle to speed up deployment and testing.

**Note**: Ensure that [Ansible Runner][ansible_runner_tool] and [Ansible Runner
HTTP Plugin][ansible_runner_http_plugin] is installed or else you will see
unexpected errors from Ansible Runner when a Custom Resource is created.

It is also important that the `role` path referenced in `watches.yaml` exists
on your machine. Since we are normally used to using a container where the Role
is put on disk for us, we need to manually copy our role to the configured
Ansible Roles path (e.g `/etc/ansible/roles`.

Run the operator locally with the default kubernetes config file present at
`$HOME/.kube/config`:

```sh
$ operator-sdk up local
INFO[0000] Go Version: go1.10
INFO[0000] Go OS/Arch: darwin/amd64
INFO[0000] operator-sdk Version: 0.0.5+git
```

Run the operator locally with a provided kubernetes config file:

```sh
$ operator-sdk up local --kubeconfig=config
INFO[0000] Go Version: go1.10
INFO[0000] Go OS/Arch: darwin/amd64
INFO[0000] operator-sdk Version: 0.0.5+git
```

## Create a Memcached CR

Modify `deploy/cr.yaml` as shown and create a `Memcached` custom resource:

```sh
$ cat deploy/cr.yaml
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 3

$ kubectl apply -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
```

Ensure that the memcached-operator creates the deployment for the CR:

```sh
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           2m
example-memcached        3         3         3            3           1m
```

Check the pods to confirm 3 replicas were created:

```sh
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
example-memcached-6fd7c98d8-7dqdr     1/1       Running   0          1m
example-memcached-6fd7c98d8-g5k7v     1/1       Running   0          1m
example-memcached-6fd7c98d8-m7vn7     1/1       Running   0          1m
memcached-operator-7cc7cfdf86-vvjqk   1/1       Running   0          2m
```

## Update the size

Change the `spec.size` field in the memcached CR from 3 to 4 and apply the
change:

```sh
$ cat deploy/cr.yaml
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 4

$ kubectl apply -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
```

Confirm that the operator changes the deployment size:

```sh
$ kubectl get deployment
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
example-memcached    4         4         4            4           5m
```

### Cleanup

Clean up the resources:

```sh
$ kubectl delete -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
$ kubectl delete -f deploy/operator.yaml
$ kubectl delete -f deploy/role_binding.yaml
$ kubectl delete -f deploy/role.yaml
$ kubectl delete -f deploy/service_account.yaml
$ kubectl delete -f deploy/crds/cache_v1alpha1_memcached_cr.yaml
```

[minikube_tool]:https://github.com/kubernetes/minikube#installation
[ansible_runner_tool]:https://ansible-runner.readthedocs.io/en/latest/install.html
[ansible_runner_http_plugin]:https://github.com/ansible/ansible-runner-http
[memcached_galaxy]:https://galaxy.ansible.com/dymurray/memcached_operator_role

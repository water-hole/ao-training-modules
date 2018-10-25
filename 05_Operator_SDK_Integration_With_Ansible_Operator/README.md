# Operator SDK Integration with Ansible Operator

**Note**: A portion of this guide uses [minikube][minikube_tool] version
v0.25.0+ as the local kubernetes cluster and quay.io for the public registry.

Ansible Operator development and testing is fully supported as a first-class
citizen within the Operator SDK. Operator SDK can be used to create new
Operator projects, test existing Operator projects, build Operator images, and
generate new Custom Resource Definitions for an Operator.

## Creating a new Operator

Use the CLI to create a new Ansible-based Operator project with the `new`
command. `operator-sdk new --type ansible` has two required flags
`--api-version` and `--kind`. These flags are used to generate proper Custom
Resource files and an Ansible Role whose name matches the input for `--kind`.
An example of this command is:

```sh
$ operator-sdk new memcached-operator --api-version=cache.example.com/v1alpha1 --kind=Memcached --type=ansible
$ cd memcached-operator
```

This creates a new memcached-operator project specifically for watching the
Memcached resource with APIVersion `cache.example.com/v1apha1` and Kind
`Memcached`.

### Project Scaffolding Layout

After creating a new operator project using `operator-sdk new --type ansible`,
the project directory has numerous generated folders and files. The following
table describes a basic rundown of each generated file/directory.


| File/Folders   | Purpose                           |
| :---           | :--- |
| deploy | Contains a generic set of kubernetes manifests for deploying this operator on a kubernetes cluster. |
| roles/\<kind\> | Contains an Ansible Role initialized using [Ansible Galaxy](https://docs.ansible.com/ansible/latest/reference_appendices/galaxy.html) |
| build | Contains scripts that the operator-sdk uses for build and initialization. |
| watches.yaml | Contains Group, Version, Kind, and Ansible invocation method. |


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

For the sake of this tutorial, we will run the operator as a pod inside of a
Kubernetes Cluster. If you are interested in learning more about running the
operator using `operator-sdk`, see the section at the bottom of this document.

### Run as a pod inside a Kubernetes cluster

Running as a pod inside a Kubernetes cluster is preferred for production use.

It is recommended to take advantage of `minikube docker-env` so that you can
build Docker images directly inside of the VM. This way the images can be
referenced from within the Kubernetes cluster without the need to push images
to a registry. To do this, run:
```bash
$ eval $(minikube docker-env)
```

Build the memcached-operator image:
```
$ operator-sdk build memcached-operator:v0.0.1
```

Kubernetes deployment manifests are generated in `deploy/operator.yaml`. The
deployment image in this file needs to be modified from the placeholder
`REPLACE_IMAGE` to the previous built image. To do this run:
```
$ sed -i 's|REPLACE_IMAGE|memcached-operator:v0.0.1|g' deploy/operator.yaml
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

## Optional: Run operator outside the cluster

This method is preferred during the development cycle to speed up deployment
and testing.

**Note**: Ensure that [Ansible Runner][ansible_runner_tool] and [Ansible Runner
HTTP Plugin][ansible_runner_http_plugin] is installed or else you will see
unexpected errors from Ansible Runner when a Custom Resource is created.

It is also important that the `role` path referenced in `watches.yaml` exists
on your machine. Since we are normally used to using a container where the Role
is put on disk for us, we need to manually copy our role to the configured
Ansible Roles path (e.g `/etc/ansible/roles`).

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

[minikube_tool]:https://github.com/kubernetes/minikube#installation
[ansible_runner_tool]:https://ansible-runner.readthedocs.io/en/latest/install.html
[ansible_runner_http_plugin]:https://github.com/ansible/ansible-runner-http

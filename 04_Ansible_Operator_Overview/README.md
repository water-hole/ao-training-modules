# Ansible Operator Overview

## Using Ansible inside of an Operator
Now that we have demonstrated using the k8s modules inside of Ansible, we want
to trigger this Ansible logic when a custom resource changes. In the above
example, we want to map a role to a specific Kubernetes resource that the
operator will watch. This is called the Watches file.

### Watches file

The Operator expects a mapping file, which lists each GVK to watch and the
corresponding path to an Ansible role or playbook, to be copied into the
container at a predefined location: /opt/ansible/watches.yaml

Dockerfile example:
```Dockerfile
COPY watches.yaml /opt/ansible/watches.yaml
```

The Watches file format is yaml and is an array of objects. The object has
mandatory fields:

**version**:  The version of the Custom Resource that you will be watching.

**group**:  The group of the Custom Resource that you will be watching.

**kind**:  The kind of the Custom Resource that you will be watching.

**playbook**:  This is the path to the playbook that you have added to the
container. This playbook is expected to be simply a way to call roles. This
field is mutually exclusive with the "role" field.

**role**:  This is the path to the role that you have added to the container.
For example if your roles directory is at `/opt/ansible/roles/` and your role
is named `busybox`, this value will be `/opt/ansible/roles/busybox`. This field
is mutually exclusive with the "playbook" field.

Example specifying a role:

```yaml
---
- version: v1alpha1
  group: foo.example.com
  kind: Foo
  role: /opt/ansible/roles/Foo
```

#### Using playbooks in watches.yaml

By default, `operator-sdk new --type ansible` sets `watches.yaml` to execute a
role directly on a resource event. This works well for new projects, but with a
lot of Ansible code this can be hard to scale if we are putting everything
inside of one role. Using a playbook allows the developer to have more
flexibility in consuming other roles and enabling more customized deployments
of their application. To do this, modify `watches.yaml` to use a playbook
instead of the role:
```yaml
---
- version: v1alpha1
  group: foo.example.com
  kind: Foo
  playbook: /opt/ansible/playbook.yml
```

Modify `tmp/build/Dockerfile` to put `playbook.yml` in `/opt/ansible` in the
container in addition to the role (`/opt/ansible` is the `HOME` environment
variable inside of the Ansible Operator base image):
```Dockerfile
FROM quay.io/water-hole/ansible-operator

COPY roles/ ${HOME}/roles
COPY playbook.yaml ${HOME}/playbook.yaml
COPY watches.yaml ${HOME}/watches.yaml
```

Alternatively, to generate a skeleton project with the above changes, a
developer can also do:
```bash
$ operator-sdk new --type ansible --kind Foo --api-version foo.example.com/v1alpha1 foo-operator --generate-playbook
```

### Custom Resource file

The Custom Resource file format is Kubernetes resource file. The object has
mandatory fields:

**apiVersion**:  The version of the Custom Resource that will be created.

**kind**:  The kind of the Custom Resource that will be created

**metadata**:  Kubernetes specific metadata to be created

**spec**:  This is the key-value list of variables which are passed to Ansible.
This field is optional and will be empty by default.

**annotations**: Kubernetes specific annotations to be appened to the CR. See
the below section for Ansible Operator specifc annotations.

#### Ansible Operator annotations
This is the list of CR annotations which will modify the behavior of the operator:

**ansible.operator-sdk/reconcile-period**: Used to specify the reconciliation
interval for the CR. This value is parsed using the standard Golang package
[time][time_pkg]. Specifically [ParseDuration][time_parse_duration] is used
which will use the default of `s` suffix giving the value in seconds.

Example:
```
apiVersion: "foo.example.com/v1alpha1"
kind: "Foo"
metadata:
  name: "example"
annotations:
  ansible.operator-sdk/reconcile-period: "30s"
```

### Testing an Ansible operator locally

Once a developer is comfortable working with the above workflow, it will be
beneficial to test the logic inside of an operator. To accomplish this, we can
use `operator-sdk up local` from the top-level directory of our project. The
`up local` command reads from `./watches.yaml` and uses `~/.kube/config` to
communicate with a kubernetes cluster just as the `k8s` modules do. This
section assumes the developer has read the [Ansible Operator user
guide][ansible_operator_user_guide] and has the proper dependencies installed.

Since `up local` reads from `./watches.yaml`, there are a couple options
available to the developer. If `role` is left alone (by default
`/opt/ansible/roles/<name>`) the developer must copy the role over to
`/opt/ansible/roles` from the operator directly. This is cumbersome because
changes will not be reflected from the current directory. It is recommended
that the developer instead change the `role` field to point to the current
directory and simply comment out the existing line:
```yaml
- version: v1alpha1
  group: foo.example.com
  kind: Foo
  #  role: /opt/ansible/roles/Foo
  role: /home/user/foo-operator/Foo
```

Create a Custom Resource Definiton (CRD) and proper Role-Based Access Control
(RBAC) defitinions for resource Foo. `operator-sdk` autogenerates these files
inside of the `deploy` folder:
```bash
$ kubectl create -f deploy/crd.yaml
$ kubectl create -f deploy/rbac.yaml
```

Run the `up local` command:
```bash
$ operator-sdk up local
INFO[0000] Go Version: go1.10.3                         
INFO[0000] Go OS/Arch: linux/amd64                      
INFO[0000] operator-sdk Version: 0.0.6+git              
INFO[0000] Starting to serve on 127.0.0.1:8888
         
INFO[0000] Watching foo.example.com/v1alpha1, Foo, default 
```

Now that the operator is watching resource `Foo` for events, the creation of a
Custom Resource will trigger our Ansible Role to be executed. Take a look at
`deploy/cr.yaml`:
```yaml
apiVersion: "foo.example.com/v1alpha1"
kind: "Foo"
metadata:
  name: "example"
```

Since `spec` is not set, Ansible is invoked with no extra variables. The next
section covers how extra variables are passed from a Custom Resource to
Ansible. This is why it is important to set sane defaults for the operator.

Create a Custom Resource instance of Foo with default var `state` set to
`present`:
```bash
$ kubectl create -f deploy/cr.yaml
```

Check that namespace `test` was created:
```bash
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    28d
kube-public   Active    28d
kube-system   Active    28d
test          Active    3s
```

Modify `deploy/cr.yaml` to set `state` to `absent`:
```yaml
apiVersion: "foo.example.com/v1alpha1"
kind: "Foo"
metadata:
  name: "example"
spec:
  state: "absent"
```

Apply the changes to Kubernetes and confirm that the namespace is deleted:
```bash
$ kubectl apply -f deploy/cr.yaml
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    28d
kube-public   Active    28d
kube-system   Active    28d
```

### Testing an Ansible operator on a cluster

Now that a developer is confident in the operator logic, testing the operator
inside of a pod on a Kubernetes cluster is desired. Running as a pod inside a
Kubernetes cluster is preferred for production use.

To build the `foo-operator` image and push it to a registry:
```
$ operator-sdk build quay.io/example/foo-operator:v0.0.1
$ docker push quay.io/example/foo-operator:v0.0.1
```

Kubernetes deployment manifests are generated in `deploy/operator.yaml`. The
deployment image in this file needs to be modified from the placeholder
`REPLACE_IMAGE` to the previous built image. To do this run:
```
$ sed -i 's|REPLACE_IMAGE|quay.io/example/foo-operator:v0.0.1|g' deploy/operator.yaml
```

Deploy the foo-operator:

```sh
$ kubectl create -f deploy/rbac.yaml
$ kubectl create -f deploy/operator.yaml
```

**NOTE**: `deploy/rbac.yaml` creates a `ClusterRoleBinding` and assumes we are
working in namespace `default`. If you are working in a different namespace you
must modify this file before creating it.

Verify that the foo-operator is up and running:

```sh
$ kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
foo-operator       1         1         1            1           1m
```

## Extra vars sent to Ansible
The extravars that are sent to Ansible are predefined and managed by the operator. The `spec` section will pass along the key-value pairs as extra vars. This is equivalent to how above extra vars are passed in to `ansible-playbook`.

For the CR example:
```yaml
apiVersion: "app.example.com/v1alpha1"
kind: "Database"
metadata:
  name: "example"
spec:
  message:"Hello world 2"
  newParameter: "newParam"
```

The structure is:


```json
{ "meta": {
        "name": "<cr-name>",
        "namespace": "<cr-namespace>",
  },
  "message": "Hello world 2",
  "new_parameter": "newParam",
  "_app_example_com_database": {
     <Full CRD>
   },
}
```
`message` and `newParameter` are set in the top level as extra variables and `meta` provides the relevant metadata for the Custom Resource as defined in the operator. The `meta` fields can be access via dot notation in Ansible as so:
```yaml
---
- debug:
    msg: "name: {{ meta.name }}, {{ meta.namespace }}"
```

[ansible_operator_user_guide]:../user-guide.md
[time_pkg]:https://golang.org/pkg/time/
[time_parse_duration]:https://golang.org/pkg/time/#ParseDuration

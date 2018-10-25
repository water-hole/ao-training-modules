# Ansible Operator Overview

This section is not a training exercise, but instead a brief overview of
Ansible Operator. If you are interested in a step-by-step example of developing
an Ansible Operator using Operator SDK, see [Section 5][section_5_link].

The reader is expected to have a basic understanding of the Operator pattern.
Ansible Operator is an Operator which is powered by Ansible. Custom resource
events trigger Ansible tasks as opposed to the traditional approach of handling
these events with Go code.

## Using Ansible inside of an Operator

Now that we have demonstrated using the k8s modules inside of Ansible, we want
to trigger this Ansible logic when a custom resource changes. It is incredibly
important that the roles/playbooks which are executed are deemed idempotent by
the developer as these tasks will be executed frequently to ensure the
application is in its proper state. The Ansible Operator uses a YaML file
called `watches.yaml`, or the Watches file, which holds the mapping between
custom resources and Ansible Roles/Playbooks.

## Watches file

The Operator expects a mapping file, which lists each GVK to watch and the
corresponding path to an Ansible role or playbook, to be copied into the
container at a predefined location: `/opt/ansible/watches.yaml`

Dockerfile example:
```Dockerfile
COPY watches.yaml /opt/ansible/watches.yaml
```

The Watches file is written in YaML and contains an array of objects. The object has
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

Example specifying a playbook:

```yaml
---
- version: v1alpha1
  group: foo.example.com
  kind: Foo
  playbook: /opt/ansible/playbook.yaml
```


### Using playbooks in watches.yaml

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
  playbook: /opt/ansible/playbook.yaml
```

Modify `tmp/build/Dockerfile` to put `playbook.yaml` in `/opt/ansible` in the
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

## Custom Resource file

The Custom Resource file format is a Kubernetes resource file. The object has
mandatory fields:

**apiVersion**:  The version of the Custom Resource that will be created.

**kind**:  The kind of the Custom Resource that will be created

**metadata**:  Kubernetes specific metadata to be created

**spec**:  This is the key-value list of variables which are passed to Ansible.
This field is optional and will be empty by default.

**annotations**: Kubernetes specific annotations to be appened to the CR. See
the below section for Ansible Operator specifc annotations.

### Ansible Operator annotations
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

## How Ansible Operator manages Extra Vars
The extravars that are sent to Ansible are predefined and managed by the
operator. The `spec` section will pass along the key-value pairs as extra vars.
This is equivalent to how above extra vars are passed in to `ansible-playbook`.

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

**Note:** The resulting JSON structure that is passed in as extra vars are autoconverted to snake-case. `newParameter` becomes `new_parameter`.

[section_5_link]:../05_Operator_SDK_Integration_With_Ansible_Operator/README.md
[time_pkg]:https://golang.org/pkg/time/
[time_parse_duration]:https://golang.org/pkg/time/#ParseDuration

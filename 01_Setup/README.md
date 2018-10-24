# Setup

## Prerequisites

1. `ansible` 2.6+
1. `openshift` Python package 0.6.0+
1. `python` 2.7+
1. `operator-sdk` 0.0.7+
1. `minikube` [MiniKube installation guide](https://kubernetes.io/docs/tasks/tools/install-minikube/)
1. `kubectl`

Optional:

1. `ansible-runner` 1.0+
1. `ansible-runner-http` 1.0+

## Installation - RPM
```
$ sudo dnf install ansible python2-openshift
```

## Installation - pip

It is beneficial to install the dependencies in a virtual environment to avoid
installing unneeded files on your local machine. To do this, ensure you have
`virtualenv` installed and run:
```
$ mkdir -p ~/operatordev/<project>
$ virtualenv ~/operatordev/<project>
$ source ~/operatordev/foo/bin/activate
$ pip install openshift ansible
```

Optional dependencies used for running an Operator on your local machine:
```
$ pip install ansible-runner ansible-runner-http
```

## Install the Operator SDK CLI - PLACEHOLDER UNTIL WE HAVE SDK RELEASE

The Operator SDK has a CLI tool that helps you create, build, and deploy a new
operator project.

Checkout the desired release tag and install the SDK CLI tool:

```sh
$ mkdir -p $GOPATH/src/github.com/operator-framework
$ cd $GOPATH/src/github.com/operator-framework
$ git clone https://github.com/operator-framework/operator-sdk
$ cd operator-sdk
$ git checkout master
$ make dep
$ make install
```

This installs the CLI binary `operator-sdk` at `$GOPATH/bin`.

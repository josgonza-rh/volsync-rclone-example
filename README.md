# How to synchronise 2 PVs using VolSync rclone method

Here we will see the necessary requirements and a step-by-step guide on how to synchronise 2 PVs in different namespaces within the same cluster (thanks to the `rclone` method, **extrapolable to namespaces in different clusters**).

## About VolSync

VolSync asynchronously replicates Kubernetes persistent volumes between clusters
using either [rsync](https://rsync.samba.org/) or [rclone](https://rclone.org/).
It also supports creating backups of persistent volumes via [restic](https://restic.net/).

[![Documentation
Status](https://readthedocs.org/projects/volsync/badge/?version=latest)](https://volsync.readthedocs.io/en/latest/?badge=latest)
[![Go Report
Card](https://goreportcard.com/badge/github.com/backube/volsync)](https://goreportcard.com/report/github.com/backube/volsync)
[![codecov](https://codecov.io/gh/backube/volsync/branch/main/graph/badge.svg)](https://codecov.io/gh/backube/volsync)
![maturity](https://img.shields.io/static/v1?label=maturity&message=alpha&color=red)

![Documentation](https://github.com/backube/volsync/workflows/Documentation/badge.svg)
![operator](https://github.com/backube/volsync/workflows/operator/badge.svg)

### Rclone Intro

With this method, VolSync synchronizes data from a `ReplicationSource` to a `ReplicationDestination` using **Rclone** using an intermediary storage system like AWS S3.

The **Rclone** method uses a “push” and “pull” model for the data replication. A schedule or other trigger is used on the source side of the relationship to trigger each replication iteration.

## Requitements

Below are the requirements needed to deploy the example on a Kubernetes/OpenShift cluster:

* A linux Laptop/Bastion/VM/...
* Have the Helm CLI installed (not really mandatory, but we want to follow [VolSync's recommendations](https://volsync.readthedocs.io/en/latest/installation/index.html#kubernetes-openshift))
* Git and clone this repo :)

As we want to synchronise 2 PVs _directly_ (without using snaphots/PiT and so ...), the volume needs to be mounted by both the VolSync data mover and the primary application at the same time. This limits it to **RWX** volumes currently. The workaround as the VolSync team works to extend it to **RWO** (I guess in the Roadmap) is to use the `node selectors` **(*)** mechanism.

> ![NOTE](images/note-icon.png) **NOTE(*)**: This is discussed in the section on the deployment of source and target applications.

## Installation

### Install VolSync operator

> ![WARNING](images/warning-icon.png) **WARNING**: Before proceeding you must be logged as _cluster-admin_ in to the OpenShift cluster.

Deploy the chart in your cluster:

```bash
helm repo add backube https://backube.github.io/helm-charts/
helm install --create-namespace -n volsync-system volsync backube/volsync
```

> ![NOTE](images/note-icon.png) **NOTE**: The Helm chart will deploy the operator in your behalf using the namespace of your choice, in this case, _volsync-system_

### S3 bucket config

As I used AWS as my IaaS provider in order to deploy my OpenShift cluster I took the chance and I decided to use their S3 service also.

The command to create my S3 bucket:

```bash
aws s3api create-bucket --bucket my-volsync-bucket --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1
```

Remember to setup the [example/rclone.conf](example/rclone.conf) file, known as `rclone-secret`, with the correct data (at least, you must change the credentials). This file provides the configuration details to locate and access the intermediary storage system. It is mounted as a secret on the Rclone data mover pod. More details about [`rclone-secret`](https://volsync.readthedocs.io/en/latest/usage/rclone/rclone-secret.html#what-is-rclone-secret).

> ![TIP](images/info-icon.png) **TIP**: a useful tool that could help you to verify the `rclone.conf` config file and the S3 bucket is the [`s3cmd`](https://s3tools.org/s3cmd), a command Line S3 Client or the [AWS client](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html) itself.

### Deploying the MySQL app (source & dest)

> ![NOTE](images/note-icon.png) **NOTE**: You don't need _cluster-admin_ privileges in order to do the following steps, but you need special rights in order to label nodes.

We are going to [placing pods on specific nodes using node selectors](https://docs.openshift.com/container-platform/4.7/nodes/scheduling/nodes-scheduler-node-selectors.html). Concretely, we are going to use `Project node selectors` so when you create a pod in the project, OpenShift adds the node selectors to the pod and schedules the pods on a node with matching labels. If there is a cluster-wide default node selector, a project node selector takes preference.

1. Label the nodes of your choice:

    ```bash
    export NODE_SOURCE=<node_you_want_1>
    export NODE_DEST=<node_you_want_2>
    
    oc label node $NODE_SOURCE volsync=source
    oc label node $NODE_DEST volsync=dest
    ```

2. Deploy the _source_ app:

    ```bash
    oc create ns source
    oc annotate ns source openshift.io/node-selector="volsync=source" --overwrite
    oc create -f example/source-database/ -n source

    ##Check the progress
    oc get pods -n source -w
    ```

    > ![NOTE](images/note-icon.png) **NOTE**: Thanks to the `openshift.io/node-selector` annotation, when you create a pod in the `source` project, the pod is scheduled on the node labeled as `"type=source"`

3. Deploy the _dest_ app:

    ```bash
    oc create ns dest
    oc annotate ns dest openshift.io/node-selector="volsync=dest" --overwrite
    oc create -n dest -f example/destination-database/
    ```

### Synchronise source & dest

**On Source:**

1. Add a new file to sync

    ```bash
    oc cp -n source README.md `oc get pods -n source | grep mysql | awk '{print $1}'`:/var/lib/mysql/README.md
    oc rsync -n source ./images `oc get pods -n source | grep mysql | awk '{print $1}'`:/var/lib/mysql
    ```

2. Config VolSync replication

    ```bash
    oc create secret generic rclone-secret --from-file=rclone.conf=./example/rclone.conf -n source
    oc create -f example/volsync_v1alpha1_replicationsource_rclone.yaml -n source
    ```

3. To verify the replication has completed describe the Replication source

    ```bash
    oc describe ReplicationSource -n source database-source
    ```

    > ![INFO](images/info-icon.png) **INFO**: More info about the [ReplicationSource](https://volsync.readthedocs.io/en/latest/usage/rclone/index.html#source-configuration) object and its attributes.

    From the output, the success of the replication can be seen by the following lines:

    ```bash
    Status:
      Conditions:
        Last Transition Time:  2021-10-13T12:08:32Z
        Message:               Reconcile complete
        Reason:                ReconcileComplete
        Status:                True
        Type:                  Reconciled
      Last Sync Duration:      4.687844461s
      Last Sync Time:          2021-10-13T16:40:04Z
      Next Sync Time:          2021-10-13T16:50:00Z
    ```

**On Dest:**

1. Config VolSync replication

    ```bash
    oc create secret generic rclone-secret --from-file=rclone.conf=./example/rclone.conf -n dest
    oc create -f example/volsync_v1alpha1_replicationdestination_rclone.yaml -n dest
    ```

2. Connect to the mysql pod and list/navigate the `/var/lib/mysql` to verify the _image_ folder exists (and its content)

    ```bash
    oc exec --stdin --tty -n dest `oc get pods -n dest | grep mysql | awk '{print $1}'` -- /bin/bash
    ##after prompt is shown, ex.
    $ ls -ltr /var/lib/mysql

    ##or directly if you want to avoid to use an interactive terminal
    oc exec -n dest `oc get pods -n dest | grep mysql | awk '{print $1}'` -- ls -ltr /var/lib/mysql
    ```

    > ![WARNING](images/warning-icon.png) **WARNING**: whether or not to see the content at the destination will depend on the configuration of the replication, its frequency (in this example, every 5 minutes). You can check the status in the same way as for the _Source_:

    ```bash
    oc describe ReplicationDestination -n dest database-destination
    ```

    > ![INFO](images/info-icon.png) **INFO**: More info about the [ReplicationDestination](https://volsync.readthedocs.io/en/latest/usage/rclone/index.html#destination-configuration) object and its attributes.

    From the output, the success of the replication can be seen by the following lines:

    ```bash
    Status:
      Conditions:
        Last Transition Time:  2021-10-13T13:54:12Z
        Message:               Reconcile complete
        Reason:                ReconcileComplete
        Status:                True
        Type:                  Reconciled
      Last Sync Duration:      4.502579832s
      Last Sync Time:          2021-10-13T17:15:04Z
      Latest Image:
        API Group:     
        Kind:          PersistentVolumeClaim
        Name:          mysql-pv-claim
      Next Sync Time:  2021-10-13T17:20:00Z
    ```

## Helpful links

> ![INFO](images/info-icon.png) **INFO**: If you need more information or a deep dive into the VolSync project, here you can find some helpful links

* [VolSync documentation](https://volsync.readthedocs.io)
* [Changelog](CHANGELOG.md)
* [Contributing guidelines](https://github.com/backube/.github/blob/master/CONTRIBUTING.md)
* [Organization code of conduct](https://github.com/backube/.github/blob/master/CODE_OF_CONDUCT.md)

## Licensing

This project is licensed under the [GNU AGPL 3.0 License](LICENSE).

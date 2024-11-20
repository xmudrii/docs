+++
title = "Using Servlet without KDP"
+++

It is possible to run the servlet component without running a fully fleged KDP installation. All that is necessary is a running [kcp](https://kcp.io) installation.

## Prerequisites

- A running kcp installation.
- A kubeconfig with admin or comparable permissions in a specific kcp workspace.

## APIExport Setup

Before installing the Servlet it is necessary to create an `APIExport` on kcp. This is automatically generated by KDP if you create a `Service` object.

The `APIExport` should be empty, because it is updated later by the Servlet. An example file could look like this:

```yaml
apiVersion: apis.kcp.io/v1alpha1
kind: APIExport
metadata:
  name: test.kubermatic.io
spec: {}
```

Create a file with a similar content (you most likely want to change the name, as that is the API group under which your published resources will be made available) and create it in a kcp workspace of your choice:

```sh
$ kubectl create -f ./apiexport.yaml
apiexport/test.kubermatic.io created
```

## Servlet Installation

The Servlet can be installed into any namespace, but in our example we are going with `kdp-system`.

Now that the `APIExport` is created, switch to the Kubernetes cluster from which you wish to [publish resources](../../service-providers/publish-resources/). You will need to ensure that a kubeconfig with access to the workspace that the `APIExport`  has been created is stored as a `Secret` on this cluster. Make sure that the kubeconfig points to the right workspace (not necessarily the `root` workspace).

This can be done via a command like this:

```sh
$ kubectl create secret generic kcp-kubeconfig -n kdp-system --from-file=kubeconfig=admin.kubeconfig
```

The next step is preparing a `values.yaml` file for the Servlet Helm chart. We need to pass the target `APIExport`, a name for the Servlet itself and a reference to the kubeconfig secret we just created.


```yaml
servlet:
  # Required: the name of the APIExport in the KDP Platform that this Servlet
  # is supposed to serve.
  apiExportName: "test.kubermatic.io"

  # Required: this Servlet's public name, will be shown in the KDP Platform,
  # purely for informational purposes.
  servletName: "unique-test"

  # Required: Name of the Kubernetes Secret that contains a "kubeconfig" key,
  # with the kubeconfig provided by the KDP Platform to access it.
  platformKubeconfig: "kcp-kubeconfig"

  # Create additional RBAC on the service cluster.
  rbac:
    createClusterRole: true
    rules:
      # in order to create APIResourceSchemas
      - apiGroups:
          - apiextensions.k8s.io
        resources:
          - customresourcedefinitions
        verbs:
          - get
          - list
          - watch
      # so copies of remote objects can be placed in their target namespaces
      - apiGroups:
          - ""
        resources:
          - namespaces
        verbs:
          - get
          - list
          - watch
          - create
```

In addition, it is important to create RBAC rules for the resources you want to publish. If you want to publish the `Certificate` resource as created by cert-manager, you will need to append the following ruleset:

```yaml
      # so we can manage certificates
      - apiGroups:
          - cert-manager.io
        resources:
          - certificates
        verbs:
          - '*'
```

Once this `values.yaml` file is prepared, install a recent development build of the Servlet:

```sh
helm install servlet oci://quay.io/kubermatic/helm-charts/kdp-servlet --version 9.9.9-9fc9a430d95f95f4b2210f91ef67b3ec153b5cab -f values.yaml -n kdp-system
```

Two `servlet` Pods should start in the `kdp-system` namespace. If they crash you will need to identify the reason from container logs. A possible issue is that the provided kubeconfig does not have permissions against the target kcp workspace.

## Publish Resources

Once the Servlet Pods are up and running, you should be able to follow [Publishing Resources](../../service-providers/publish-resources/). Be aware that the remaining KDP documentation assumes you are working on a KDP installation, and as such might show you screenshots of the KDP dashboard. Such a UI to manage resources does not exist if you use the servlet without, and you will need to create objects from the command line directly.

## Consume Service

Once resources have been published through the Servlet, they can be consumed on the kcp side (i.e. objects on kcp will be synced back and forth with the service cluster). Follow the [manual steps to consume services](../../platform-users/consuming-services/#manual).

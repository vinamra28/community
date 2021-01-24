---
status: proposed
title: Tekton Component Versioning
creation-date: "2021-02-01"
last-updated: "2021-02-01"
authors:
  - "@vinamra28"
  - "@piyush-garg"
  - "SM43"
---

# TEP-0041: Tejton Component Versioning

<!-- toc -->

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases (optional)](#use-cases-optional)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats (optional)](#notescaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [User Experience (optional)](#user-experience-optional)
  - [Performance (optional)](#performance-optional)
- [Design Details](#design-details)
  - [1. Creating a CRD](#1.-creating-a-crd)
  - [2. Creating a ConfigMap](#2.-creating-a-configmap)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [References (optional)](#references-optional)
<!-- /toc -->

## Summary

This TEP proposes to add versioning for all the components of Tekton in
such a manner that any authenticated user of the cluster on which Pipelines
is installed can get to know version of any Tekton Component. Also with
Tekton CLI, we can get to know the version of pipelines and commands like
install/upgrade/downgrade a resource from catalog only if the newer resource
version is compatible with Pipelines.

## Motivation

Finding version of the tekton components installed is not an easy
thing for all users as few permissions are needed.

In case we fetch version from controller deployment we need:-

- Namespace in which tekton component is installed
- Permissions to view deployment/pod in installed namespace

In case fetching version from CRD we need:-

- Permissions to view CRD on cluster

These permissions may not be provided to all the users and finding
a version is little difficult in some cases. As Tekton Hub CLI won't be
able to find the component version for all users, it is difficult to
provide the benefits of some `tkn hub` commands like install, upgrade
and downgrade which can work efficiently.

### Goals

1. All the users having access to the cluster should be able to view the version
   of Tekton component installed using Tekton CLI.
2. Tekton Hub CLI should be able to install/upgrade resources from catalog on the
   basis of Tekton component installed.

### Non-Goals

1. How to handle versioning of components which were released prior to this
   TEP will not be covered.
2. Maintaining the agreed upon solution in the respective components will not
   be covered.

### Use Cases (optional)

#### Tekton Hub CLI

1. Tekton Hub CLI will provide an install command like `tkn hub install task buildah`.
2. Tekton Hub CLI will provide an update command like `tkn hub upgrade task buildah`.
3. Tekton Hub CLI will provide an check-updates command to list all tasks whose newer
   version is available.

We want to perform above operations using `tkn hub` CLI based on the version
of Tekton Pipelines installed on the cluster and `pipelines.minVersion`
annotation available in catalog like [this](https://github.com/tektoncd/catalog/blob/master/task/buildah/0.2/buildah.yaml#L9).
This feature can later be extended for the other Tekton components such
as Triggers, Dashboard etc.

#### New Tekton User

New Tekton users can get to know the version of Tekton components
installed so that they can create their pipelines based on the features
available in that version.

## Requirements

None

## Proposal

This proposal is to allow the system authenticated users of the cluster
to access the version of the Tekton component installed even if they don't
have admin rights to the cluster.

The two possible solutions are:-

1. Creating a CRD
2. Creating a ConfigMap

Possible design details are present in [Design Details](#design-details)

### Risks and Mitigations

1. Tekton CLI will not be able to access the version information from previously
   released tekton components without this functionality.

2. Tekton CLI needs to handle backward compatibility,

### User Experience (optional)

We need to provide the details in the docs of how to get the
version information.

### Performance (optional)

## Design Details

As discussed in the proposal the possible solutions with their
`pros` and `cons` are:-

### 1. Creating a CRD

Tekton components should create a CRD which is to provide information like
metadata about components.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: info.tekton.dev
  labels:
    app.kubernetes.io/instance: default
    app.kubernetes.io/part-of: tekton-pipelines
    pipeline.tekton.dev/release: "devel"
    version: "devel"
spec:
  group: tekton.dev
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - version
              properties:
                version:
                  type: string
                  description: version of the component
  scope: Cluster
  names:
    plural: info
    singular: info
    kind: Info
    categories:
      - tekton
      - tekton-pipelines
```

And a CR should be added during installation which has the details
about the version of the component or any other info required.

```yaml
apiVersion: tekton.dev/v1
kind: Info
metadata:
  name: pipeline-info
spec:
  version: v0.18.0
```

`ClusterRole` and `ClusterRoleBinding`

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pipelines-info-view-cr
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups: ["tekton.dev"]
    resources: ["info"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pipelines-info-crbinding
subjects:
  - kind: Group
    name: system:authenticated
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pipelines-info-view
```

#### Advantages

1. The CRD can be clusterwide so user need not to know the installed pipeline namespace.

#### Disadvantages

1. Where to store the CRD?

   - Pipelines, Triggers or somewhere else?
   - In each component then how to handle the changes?
     - (Solution) Each component will have to maintain their own CRD so that they
       can provide their respective information if they want to.

### 2. Creating a ConfigMap

Tekton components should create their own ConfigMap which contains
the version information.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pipeline-version
  namespace: tekton-pipelines
data:
  version: v0.19.0
```

`Role` and `RoleBinding`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: info
  namespace: tekton-pipelines
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["pipelines-version"]
    verbs: ["get", "describe"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: info
  namespace: tekton-pipelines
subjects:
  - kind: Group
    name: system:authenticated
    apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: info
```

#### Advantages

1. Other users not having access to the namespace can still have access
   to the particular ConfigMap.

   `kubectl get cm pipelines-version -n tekton-pipelines`

#### Disadvantages

1. User need to know the namespace in which Tekton components are installed
   since the scope is only the namespace.

## Test Plan

Basic test scenarios of getting the version information should be added in all the components.
Other components like CLI can add the test cases based on their usage.

## Design Evaluation

## Drawbacks

## Alternatives

## Upgrade & Migration Strategy (optional)

## References (optional)

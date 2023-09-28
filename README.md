![CrowdStrike CloudTrail Lambda](https://raw.githubusercontent.com/CrowdStrike/Kubernetes-FluentBit-Logging-Falcon-Logscale-Integration/main/docs/assets/cs-logo.png)

[![Twitter URL](https://img.shields.io/twitter/url?label=Follow%20%40CrowdStrike&style=social&url=https%3A%2F%2Ftwitter.com%2FCrowdStrike)](https://twitter.com/CrowdStrike)<br/>

# Kubernetes log forwarding with the Fluent-Bit log shipper

Forward Kubernetes logs and system metrics to Falcon LogScale with the fluent-bit log shipper.

## Overview

This integration provides instructions on how to configure the Fluent-Bit log shipper to forward Kubernetes cluster logs and node metrics to Falcon LogScale for ingestion.

This integration was created as a companion to the LogScale [Kubernetes Logging Package](https://github.com/CrowdStrike/logscale-community-content/tree/dev/Log-Sources/Kubernetes/Kubernetes-FluentBit) which is available on the [CrowdStrike GitHub](https://github.com/CrowdStrike) repository.

*note: The Kubernetes Logging package dashboards rely on specific enrichments added by this integration.*

## Fluent-bit

[Fluent-Bit](https://fluentbit.io/) is a fast and lightweight end-to-end log processing pipeline that collects, filters, and forwards logs and metrics.  It is an [open source](https://docs.fluentbit.io/manual/about/what-is-fluent-bit) sub-project of the [CNCF](https://www.cncf.io/).

Fluent-bit is written in C.  It provides a variety of plugins to input, filter, and output logging telemetry.  It can be run as system level service or as a DaemonSet within Kubernetes.

## Implementation Notes

This integration employs a split horizon setup of fluent-bit in order to capture a complete view of Kubernetes cluster logging telemetry: that is, fluent-bit is run simultaneously as a system service outside of Kubernetes, and as a DaemonSet within Kubernetes.

As a DaemonSet, fluent-bit monitors and forwards well known Kubernetes log sources: container, docker, audit.

As a system service, fluent-bit collects and forwards a variety of node metrics (cpu, network, disk, memory), and Docker events.

The split horizon implementation reflects the current state of fluent-bit namespace contraints among the various fluent-bit input plugins: metrics scraping is currently only possible from the system level where the fluent-bit metrics `[INPUT]` plugins have access to special device files in /proc and /sys.

Metrics capture is not currently possible when running fluent-bit from within a Kubernetes container.

Logs collected by the DaemonSet are automatic enriched with a set of Kubernetes properties: `i.e. "kubernetes": { ... }`

```json
{
    "@timestamp": "2023-XX-XXTXX:XX:XX.XXXZ",
    "log": "[2023/XX/XX XX:XX:XX] [ info] [engine] flush chunk '1-1111111111.111111111.flb' succeeded at retry 2: task_id=7, input=tail.1 > output=es.0 (out_id=0)\n",
    "stream": "stderr",
    "time": "2023-XX-XXTXX:XX:XX.000000000Z",
    "source": "container",
    "hostname": "host.name.com",
    "kubernetes": {
        "pod_name": "fluent-bit-tsv4f",
        "namespace_name": "logging",
        "pod_id": "nnnnnnnn-nnnn-nnnn-nnnn-nnnnnnnnnnnn",
        "labels": {
            "app.kubernetes.io/instance": "fluent-bit",
            "app.kubernetes.io/name": "fluent-bit",
            "controller-revision-hash": "1111111111",
            "pod-template-generation": "2"
        },
        "annotations": {
            "checksum/config": "nnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnn",
            "checksum/luascripts": "nnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnn",
            "cni.projectcalico.org/containerID": "nnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnn",
            "cni.projectcalico.org/podIP": "10.10.10.11/32",
            "cni.projectcalico.org/podIPs": "10.10.10.11/32",
            "kubectl.kubernetes.io/restartedAt": "2023-XX-XXTXX:XX:XX-XX:XX"
        },
        "host": "host.name.com",
        "container_name": "fluent-bit",
        "docker_id": "nnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnn",
        "container_hash": "cr.fluentbit.io/fluent/fluent-bit@sha256:nnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnn",
        "container_image": "cr.fluentbit.io/fluent/fluent-bit:2.0.8"
    }
}
```


## Installation and Setup
### Install the Kubernetes Logging LogScale package
[Kubernetes Logging Package](https://github.com/CrowdStrike/logscale-community-content/tree/dev/Log-Sources/Kubernetes/Kubernetes-FluentBit) 
*note: the `fluentbit` parser is included in the package*


#### Install Fluent-Bit as a System Service
*note: this integration assumes Ubuntu Linux as the target platform*
##### [Apt Install Fluent-Bit](https://docs.fluentbit.io/manual/installation/linux/ubuntu)
`sudo apt-get install fluent-bit`

##### Configure Fluent-Bit
Prepare the `/etc/fluent-bit/fluent-bit.conf` file.  
Use the `fluent-bit.conf` template file included with this integration to source the `[INPUT]`, `[FILTER]`, and `[OUTPUT]` stanzas.
Update the `[OUTPUT]` stanza with your LogScale ingest credentials.
*note: a fluent-bit timestamp is added to some metrics events that lack native timestamping.*
##### Start Fluent-Bit Service
`sudo systemctl start fluent-bit`

Verify that events are being ingested to the repository.

#### Install Fluent-Bit as a Kubernetes DaemonSet
#####Create a Logging namespace
example: `kubectl create namespace logging`
#####Prepare the Helm Chart
Follow the instructions in the [guide to installing Fluent-Bit via Helm](https://fluentbit.io/blog/2020/12/29/5-minute-guide-to-deploying-fluent-bit-on-kubernetes/) to install the base Fluent-Bit DaemonSet.

###### Configure the Fluent-Bit Helm Chart
###### ConfigMap
Use the included `fluent-bit.conf` template file to provide the stanzas for `[INPUT]`, `[FILTER]`, and `[OUTPUT]`.
Update the `[OUTPUT]` stanza with your LogScale ingest credentials.

###### Tolerations
Add a toleration to the DaemonSet, to allow Fluent-Bit to be scheduled on control-plane master nodes:
```
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
```

###### Auditing
Update the fluent-bit ClusterRole yaml to allow access to `events` resources:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
  . . .
  . . .
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  - pods
[[- events]] add this resource <-------
  verbs:
  - get
  - list
  - watch
```

*note: without `events` resource access, the following error will occur:*
```
{"log":"[2023/XX/XX XX:XX:XX] [error] [input:kubernetes_events:kubernetes_events.4] http_status=403:\n","stream":"stderr","time":"2023-XX-XXTXX:XX:XX.XXXXXXXXXZ"}
{"log":"{\"kind\":\"Status\",\"apiVersion\":\"v1\",\"metadata\":{},\"status\":\"Failure\",\"message\":\"events is forbidden: User \\\"system:serviceaccount:logging:fluent-bit\\\" cannot list resource \\\"events\\\" in API group \\\"\\\" at the cluster scope\",\"reason\":\"Forbidden\",\"details\":{\"kind\":\"events\"},\"code\":403}\n","stream":"stderr","time":"2023-XX-XXTXX:XX:XX.XXXXXXXXZ"}
```

##### Helm Install Fluent-Bit
`helm install fluent-bit .`

Once fluent-bit is enabled and the DaemonSet is running on all nodes, events should begin to appear in LogScale.

---

<p align="center"><img src="https://raw.githubusercontent.com/CrowdStrike/Kubernetes-FluentBit-Logging-Falcon-Logscale-Integration/main/docs/assets/cs-logo-footer.png"><BR/><img width="250px" src="https://raw.githubusercontent.com/CrowdStrike/Kubernetes-FluentBit-Logging-Falcon-Logscale-Integration/main/docs/assets/adversary-red-eyes.png"></P>
<h3><P align="center">WE STOP BREACHES</P></h3>
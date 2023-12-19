# cncf-green-reviews-testing

[![Falco Infra Repository](https://github.com/falcosecurity/evolution/blob/main/repos/badges/falco-infra-blue.svg)](https://github.com/falcosecurity/evolution/blob/main/REPOSITORIES.md#infra-scope) [![Sandbox](https://img.shields.io/badge/status-sandbox-red?style=for-the-badge)](https://github.com/falcosecurity/evolution/blob/main/REPOSITORIES.md#sandbox) [![License](https://img.shields.io/github/license/falcosecurity/testing?style=for-the-badge)](./LICENSE)

Welcome to The Falco Project's collaborative testing initiatives in partnership with the [CNCF Environmental Sustainability Technical Advisory Group (TAG ENV)](https://github.com/cncf/tag-env-sustainability) - Green Reviews Working Group.

This repository functions as the hosting platform for Falcos' daemonset configurations intended for testing with the CNCF Green Reviews Working Group. These configurations will be used within the following repository: https://github.com/cncf-tags/green-reviews-tooling/, leveraging the [Flux](https://fluxcd.io/flux/) framework.


The primary directory structure is outlined below:

```
├── kustomize
│   ├── falco-driver
│   │   ├── bpf
│   │   ├── kmod
│   │   └── modern-bpf
│   │       ├── configmap.yaml
│   │       └── daemonset.yaml
│   ├── falco-generic
│   │   ├── falcoctl-configmap.yaml
│   │   └── serviceaccount.yaml
│   ├── kustomization.yaml
│   └── synthetic-workloads
│       ├── falco-event-generator.yaml
│       ├── redis.yaml
│       ├── ...
│       └── stress-ng.yaml
├── LICENSE
├── OWNERS
└── README.md
```

## Falco Deployment

The Falco daemonset generic under `./kustomize` resemble existing generic available at https://github.com/falcosecurity/deploy-kubernetes/, but are customized to cater to specific purposes and requirements (e.g. namespace `falco` and nodeSelector `cncf-project-sub: "falco-driver-modern-bpf"`).

Furthermore, there's a customized setup within the Falco container entrypoint and `falco.yaml` settings, focusing on benchmarking Falco's performance. Notably, we direct Falco alerts and internal metrics solely to log-rotated files, unlike real-world scenarios where this data is usually sent off the knode to a data lake.

For our testing process, each Falco [driver](https://github.com/falcosecurity/libs/tree/master/driver) type undergoes testing on its own dedicated knode.

## Synthetic Workloads Deployment

The `./kustomize/synthetic-workloads` directory contains deployments for microservices or teststress frameworks to generate synthetic workloads on the CNC testbed servers. This aims to test Falco with a certain level of realistic application activity (w/ nodeSelector `cncf-project: "falco"`).

## Summary CNCF Green Reviews Cluster Requirements

| Knode   | Falco Driver | Namespace | Node Selector                           |
|---------|--------------|-----------|----------------------------------------|
| knode A | modern-bpf   | falco     | cncf-project: "falco"                  |
|         |              |           | cncf-project-sub: "falco-driver-modern-bpf" |
| knode B | bpf          | falco     | cncf-project: "falco"                  |
|         |              |           | cncf-project-sub: "falco-driver-bpf"   |
| knode C | kmod         | falco     | cncf-project: "falco"                  |
|         |              |           | cncf-project-sub: "falco-driver-kmod"  |

| Knode   | Kernel Version Requirement | Additional Requirements  | BPF Stats Enabled |
|---------|---------------------------|--------------------------|-------------------|
| knode A | >= 5.8                    | eBPF supported           | 1                 |
| knode B | >= 4.14                   | eBPF supported           | 1                 |
| knode C | >= 2.6.32                 | DKMS package installed   | N/A               |


Notes:
- The Falco Deployment enables `kernel.bpf_stats_enabled` by default.
- For both `bpf` and `kmod`, additional host mounts are required, such as `/usr/src/kernels/` and `/lib/modules`. Please refer to the respective daemonset configuration for more details.
- We anticipate `containerd` to be the container runtime socket located at `/run/containerd/containerd.sock`.


## HowTo: A Guide for `localhost` Testing

<details>
	<summary>Expand Testing Instructions</summary>

To test these configurations for the modern BPF driver on localhost using [minikube](https://minikube.sigs.k8s.io/docs/start/), make sure you have minikube installed and running. In order to test `kmod` and `bpf` drivers, additional host mounts are required. Minikube needs a specific setting to accommodate this, as shown below:

```
minikube start --mount --mount-string="/usr/src:/usr/src" --driver=docker
```

__NOTE__: You won't be able to properly test Falco's container engine using `minikube`. Please be aware of this limitation.

Proceed by executing the following setup commands:

```
kubectl create namespace falco;
# Always need the generic label
kubectl label nodes minikube cncf-project=falco;

# Test modern-bpf (easiest)
kubectl label nodes minikube cncf-project-sub=falco-driver-modern-bpf;

# Test bpf
# kubectl label nodes minikube cncf-project-sub=falco-driver-bpf --overwrite;

# Test kmod
# kubectl label nodes minikube cncf-project-sub=falco-driver-kmod --overwrite;

kubectl get nodes --show-labels;
```

Apply the configurations by executing the following command:

```
kubectl apply -k ./kustomize
```

Verify if the pods are up and running (Note that the output below is not regularly updated, and there might be more pods and containers running than displayed): 

```
kubectl get pods -n falco

NAME                                     READY   STATUS    RESTARTS   AGE
falco-driver-modern-bpf-5vwl6            1/1     Running   0          15m
falco-event-generator-65d99cdd6c-5hq8t   1/1     Running   0          15m
redis-54f4f4997d-9d652                   3/3     Running   0          15m
stress-ng-7b488f58c4-tjdlc               2/2     Running   0          15m
```

To drop interactively into the Falco container, execute the `exec` command as follows:

```
kubectl -n falco exec -it falco-driver-modern-bpf-5vwl6 -c falco -- bash
```

Execute dummy suspicious commands and examine Falco's alert outputs and native metrics logs:

```
cat /etc/shadow
# Falco alerts outputs
cat /tmp/falco/events.jsonl
# Falco native metrics logs; recommend adjusting `interval: 1m` for quicker testing
cat /tmp/stats/falco_stats.jsonl
```

The Falco container includes utilities installed for ad-hoc checks on the Falco process:

```
ps aux 
htop
```

Extra Tips

```
# Check if Falco's kmod was loaded
lsmod | grep falco
# Inspect possible issues with a pod
kubectl -n falco describe pod falco-driver-modern-bpf-5vwl6
```

</details>

</br>

## Versioning

The respective `CONFIG_VERSION` environment variable within the daemonset deployment contains the semver-compatible version of the testbed setup. We inject it (as a suffix) into the `FALCO_HOSTNAME` environment variable to maintain a version record extending beyond the Falco version in the native Falco metrics. Every merge into the main branch necessitates a (mostly minor) version increment.

## How to Contribute

Please refer to the [contributing guide](https://github.com/falcosecurity/.github/blob/main/CONTRIBUTING.md) and the [code of conduct](https://github.com/falcosecurity/evolution/CODE_OF_CONDUCT.md) for more information on how to contribute.

## License

This project is licensed to you under the [Apache 2.0](./COPYING) open source license.

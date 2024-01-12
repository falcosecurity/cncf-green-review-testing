# cncf-green-reviews-testing

[![Falco Infra Repository](https://github.com/falcosecurity/evolution/blob/main/repos/badges/falco-infra-blue.svg)](https://github.com/falcosecurity/evolution/blob/main/REPOSITORIES.md#infra-scope) [![Sandbox](https://img.shields.io/badge/status-sandbox-red?style=for-the-badge)](https://github.com/falcosecurity/evolution/blob/main/REPOSITORIES.md#sandbox) [![License](https://img.shields.io/github/license/falcosecurity/testing?style=for-the-badge)](./LICENSE)

Welcome to The Falco Project's collaborative testing initiatives in partnership with the [CNCF Environmental Sustainability Technical Advisory Group (TAG ENV)](https://github.com/cncf/tag-env-sustainability) - Green Reviews Working Group.

This repository functions as the hosting platform for Falcos' daemonset configurations intended for testing with the CNCF Green Reviews Working Group. These configurations will be used within the following repository: https://github.com/cncf-tags/green-reviews-tooling/, leveraging the [Flux](https://fluxcd.io/flux/) framework.


The primary directory structure is outlined below:

```
├── kustomize
│   ├── falco-driver
│   │   ├── ebpf
│   │   │   ├── configmap.yaml
│   │   │   ├── daemonset.yaml
│   │   │   ├── falco-event-generator.yaml
│   │   │   ├── redis.yaml
│   │   │   └── stress-ng.yaml
│   │   ├── kmod
│   │   │   ├── configmap.yaml
│   │   │   ├── daemonset.yaml
│   │   │   ├── falco-event-generator.yaml
│   │   │   ├── redis.yaml
│   │   │   └── stress-ng.yaml
│   │   └── modern_ebpf
│   │       ├── configmap.yaml
│   │       ├── daemonset.yaml
│   │       ├── falco-event-generator.yaml
│   │       ├── redis.yaml
│   │       └── stress-ng.yaml
│   ├── falco-generic
│   │   ├── falcoctl-configmap.yaml
│   │   └── serviceaccount.yaml
│   └── kustomization.yaml
├── LICENSE
├── OWNERS
└── README.md
```

## Falco Deployment

The Falco daemonset definitions under `./kustomize/driver/{ebpf,kmod,modern_ebpf}/daemonset.yaml` resemble existing templates available at https://github.com/falcosecurity/deploy-kubernetes/, but are customized to cater to specific purposes and requirements (e.g. namespace `falco` and nodeSelector `cncf-project-sub: "falco-driver-modern-ebpf"`).

Furthermore, there's a customized setup within the Falco container entrypoint and `falco.yaml` settings, focusing on benchmarking Falco's performance. Notably, we direct Falco alerts and internal metrics solely to log-rotated files, unlike real-world scenarios where this data is usually sent off the knode to a data lake.

For our testing process, each Falco [driver](https://github.com/falcosecurity/libs/tree/master/driver) type undergoes testing on its own dedicated knode.

## Synthetic Workloads Deployment

Each Falco driver-specific deployment under `./kustomize/driver/{ebpf,kmod,modern_ebpf}/` also contains deployments for microservices or teststress frameworks aimed at generating synthetic workloads on the CNC testbed servers.

## Summary CNCF Green Reviews Cluster Requirements

| Knode   | Falco Driver | Namespace | Node Selector                           |
|---------|--------------|-----------|----------------------------------------|
| knode A | modern-ebpf   | falco     | cncf-project: "falco"                  |
|         |              |           | cncf-project-sub: "falco-driver-modern-ebpf" |
| knode B | ebpf          | falco     | cncf-project: "falco"                  |
|         |              |           | cncf-project-sub: "falco-driver-ebpf"   |
| knode C | kmod         | falco     | cncf-project: "falco"                  |
|         |              |           | cncf-project-sub: "falco-driver-kmod"  |

| Knode   | Kernel Version Requirement | Additional Requirements  | BPF Stats Enabled |
|---------|---------------------------|--------------------------|-------------------|
| knode A | >= 5.8                    | eBPF supported           | 1                 |
| knode B | >= 4.14                   | eBPF supported, Kernel headers installed           | 1                 |
| knode C | >= 2.6.32                 | DKMS package, Kernel headers installed   | N/A               |

Notes:
- The Falco Deployment enables `kernel.bpf_stats_enabled` by default.
- For both `ebpf` and `kmod`, additional host mounts are required, such as `/usr/src/` and `/lib/modules`. Please refer to the respective daemonset configuration for more details.
- We anticipate `containerd` to be the container runtime socket located at `/run/containerd/containerd.sock`.

## HowTo: A Guide for `localhost` Testing

<details>
	<summary>Expand Testing Instructions</summary>

To test these configurations on localhost using [minikube](https://minikube.sigs.k8s.io/docs/start/), make sure you have minikube and [kubectl](https://pwittrock.github.io/docs/tasks/tools/install-kubectl/) installed and running. In order to test `kmod` and `ebpf` drivers, additional host mounts are required. Minikube needs a specific setting to accommodate this, as shown below:

```
minikube start --mount --mount-string="/usr/src:/usr/src" --mount --mount-string="/dev:/dev" --driver=docker --nodes 4
```

__NOTE__: You won't be able to properly test Falco's container engine using `minikube`. Please be aware of this limitation.

__NOTE__: For `localhost` testing reduce the number of replicas for the synthetic workload deployments.

Proceed by executing the following setup commands:

```bash
kubectl create namespace falco;
kubectl get nodes;

# Test cncf-project-sub=falco-driver-modern-ebpf (easiest)
kubectl label nodes minikube-m02 cncf-project=falco cncf-project-sub=falco-driver-modern-ebpf --overwrite;

# Test cncf-project-sub=falco-driver-ebpf
kubectl label nodes minikube-m03 cncf-project=falco cncf-project-sub=falco-driver-ebpf --overwrite;

# Test cncf-project-sub=falco-driver-kmod
# WARNING: Testing kernel modules on a local dev box is more risky, 
# remember to unload the module `sudo rmmod falco`
# Testing kmod within a smaller VM with minikube likely crashes, only test w/ minikube on a larger native box

# kubectl label nodes minikube-m04 cncf-project=falco cncf-project-sub=falco-driver-kmod --overwrite;

kubectl get nodes --show-labels;
```

Apply the configurations by executing the following command:

```bash
kubectl apply -k ./kustomize
# Tear-down
kubectl delete -k ./kustomize
```

Verify if the pods are up and running (Note that the output below is not regularly updated, and there might be more pods and containers running than displayed): 

```bash
kubectl get pods -n falco

NAME                                                        READY   STATUS    RESTARTS   AGE
falco-driver-ebpf-bjvgc                                     1/1     Running   0          5m26s
falco-driver-modern-ebpf-fpph9                              1/1     Running   0          5m26s
falco-event-generator-driver-ebpf-785c6cc7dc-58wjr          1/1     Running   0          5m27s
falco-event-generator-driver-modern-ebpf-64674f78bf-fjvn7   1/1     Running   0          5m27s
redis-driver-ebpf-cbdd47b74-4drg4                           3/3     Running   0          5m27s
redis-driver-ebpf-cbdd47b74-lb6wt                           3/3     Running   0          5m27s
redis-driver-ebpf-cbdd47b74-lt6q7                           3/3     Running   0          5m27s
redis-driver-ebpf-cbdd47b74-pcm8g                           3/3     Running   0          5m27s
redis-driver-ebpf-cbdd47b74-rv2ww                           3/3     Running   0          5m27s
redis-driver-modern-ebpf-7c4bdd9d58-2fqp9                   3/3     Running   0          5m27s
redis-driver-modern-ebpf-7c4bdd9d58-2ms8j                   3/3     Running   0          5m27s
redis-driver-modern-ebpf-7c4bdd9d58-k5vtw                   3/3     Running   0          5m27s
redis-driver-modern-ebpf-7c4bdd9d58-kztgj                   3/3     Running   0          5m27s
redis-driver-modern-ebpf-7c4bdd9d58-rf9m2                   3/3     Running   0          5m27s
stress-ng-driver-ebpf-78766f6fbd-cxljg                      2/2     Running   0          5m27s
stress-ng-driver-ebpf-78766f6fbd-rb9wn                      2/2     Running   0          5m27s
stress-ng-driver-modern-ebpf-7885fdc996-mkb78               2/2     Running   0          5m27s
stress-ng-driver-modern-ebpf-7885fdc996-rzl4h               2/2     Running   0          5m26s
...

```

To drop interactively into the Falco container, execute the `exec` command as follows:

```bash
kubectl -n falco exec -it falco-driver-modern-ebpf-5vwl6 -c falco -- bash
```

Execute dummy suspicious commands and examine Falco's alert outputs and native metrics logs:

```bash
cat /etc/shadow
# Falco alerts outputs
cat /tmp/falco/events.jsonl
# Falco native metrics logs; recommend adjusting `interval: 1m` for quicker testing
cat /tmp/stats/falco_stats.jsonl
```

The Falco container includes utilities installed for ad-hoc checks on the Falco process:

```bash
ps aux 
htop
```

Extra Tips

```bash
# Check if Falco's kmod was loaded
lsmod | grep falco
# Inspect possible issues with a pod
kubectl -n falco describe pod falco-driver-modern-ebpf-5vwl6
```

</details>

</br>

## Versioning

The respective `CONFIG_VERSION` environment variable within the daemonset deployment contains the semver-compatible version of the testbed setup. We inject it (as a suffix) into the `FALCO_HOSTNAME` environment variable to maintain a version record extending beyond the Falco version in the native Falco metrics. Every merge into the main branch necessitates a (mostly minor) version increment.

## How to Contribute

Please refer to the [contributing guide](https://github.com/falcosecurity/.github/blob/main/CONTRIBUTING.md) and the [code of conduct](https://github.com/falcosecurity/evolution/CODE_OF_CONDUCT.md) for more information on how to contribute.

## License

This project is licensed to you under the [Apache 2.0](./COPYING) open source license.

# cncf-green-review-testing

[![Falco Infra Repository](https://github.com/falcosecurity/evolution/blob/main/repos/badges/falco-infra-blue.svg)](https://github.com/falcosecurity/evolution/blob/main/REPOSITORIES.md#infra-scope) [![Sandbox](https://img.shields.io/badge/status-sandbox-red?style=for-the-badge)](https://github.com/falcosecurity/evolution/blob/main/REPOSITORIES.md#sandbox) [![License](https://img.shields.io/github/license/falcosecurity/testing?style=for-the-badge)](./LICENSE)

Welcome to The Falco Project's collaborative testing initiatives in partnership with the [CNCF Environmental Sustainability Technical Advisory Group (TAG ENV)](https://github.com/cncf/tag-env-sustainability) - Green Reviews Working Group.

This repository functions as the hosting platform for Falcos' daemonset configurations intended for testing with the CNCF Green Reviews Working Group. These configurations will be used within the following repository: https://github.com/cncf-tags/green-reviews-tooling/, leveraging the [Flux](https://fluxcd.io/flux/) framework (see the [Falco Flux Config](https://github.com/cncf-tags/green-reviews-tooling/tree/main/projects/falco)).


The primary directory structure is outlined below:

```
├── benchmark-tests
│   └── falco-benchmark-tests.yaml
├── kustomize
│   ├── falco-driver
│   │   ├── ebpf
│   │   │   ├── configmap.yaml
│   │   │   ├── daemonset.yaml
│   │   │   └── kustomization.yaml
│   │   ├── kmod
│   │   │   ├── configmap.yaml
│   │   │   ├── daemonset.yaml
│   │   │   └── kustomization.yaml
│   │   └── modern_ebpf
│   │       ├── configmap.yaml
│   │       ├── daemonset.yaml
│   │       └── kustomization.yaml
│   └── falco-generic
│       ├── falcoctl-configmap.yaml
│       ├── kustomization.yaml
│       └── serviceaccount.yaml
├── LICENSE
├── OWNERS
└── README.md
```

## Falco Deployment

The Falco daemonset definitions under `./kustomize/falco-driver/{ebpf,kmod,modern_ebpf}/daemonset.yaml` resemble existing templates available at https://github.com/falcosecurity/deploy-kubernetes/, but are customized to cater to specific purposes and requirements.

For our testing process, each Falco [driver](https://github.com/falcosecurity/libs/tree/master/driver) type undergoes testing on its own dedicated knode.

Falco's performance metrics (see the [Official Falco Metrics Framework](https://falco.org/docs/concepts/metrics/)) are exposed through its internal web server, utilizing the Prometheus metrics sink (verify via `curl http://127.0.0.1:8765/metrics`). Falco's alerts are not sent anywhere at this time.


## Synthetic Workloads Deployment

Initial synthetic workload deployments are hosted under the `./benchmark-tests` directory.

## Summary CNCF Green Reviews Cluster Requirements

| Knode   | Falco Driver | Namespace | Node Selector                          |
|---------|--------------|-----------|----------------------------------------|
| knode A | `modern_ebpf`   | falco     | node-role.kubernetes.io/benchmark: "true" |
| knode B | `ebpf`          | falco     | node-role.kubernetes.io/benchmark: "true" |
| knode C | `kmod`         | falco     | node-role.kubernetes.io/benchmark: "true" |

| Knode   | Kernel Version Requirement | Additional Requirements  | BPF Stats Enabled |
|---------|---------------------------|--------------------------|-------------------|
| knode A | >= 5.8                    | eBPF supported, CO-RE Support | 1 |
| knode B | >= 4.14                   | eBPF supported, Kernel headers installed | 1 |
| knode C | >= 2.6.32                 | DKMS package, Kernel headers installed   | N/A |

Notes:
- The Falco Deployment enables `kernel.bpf_stats_enabled` by default.
- For both `ebpf` and `kmod`, additional host mounts are required, such as `/usr/src/` and `/lib/modules`. Please refer to the respective daemonset configuration for more details.
- We anticipate `containerd` to be the container runtime socket located at `/run/k3s/containerd/containerd.sock`.
- The CNCF test Kubernetes cluster offers machines with 16 CPUs.

## HowTo: A Guide for `localhost` Testing

<details>
	<summary>Expand Testing Instructions</summary>

To test these configurations on localhost using [minikube](https://minikube.sigs.k8s.io/docs/start/), make sure you have minikube and [kubectl](https://pwittrock.github.io/docs/tasks/tools/install-kubectl/) installed and running. In order to test `kmod` and `ebpf` drivers, additional host mounts are required. Minikube needs a specific setting to accommodate this, as shown below:

```bash
minikube start --mount --mount-string="/usr/src:/usr/src" --mount --mount-string="/dev:/dev" --driver=docker --nodes 1
```

__NOTE__: You won't be able to properly test Falco's container engine using `minikube`. Please be aware of this limitation, and there can still be issues with host mounts.

__NOTE__: We recommend testing on Ubuntu to reflect the CNCF testbed setup. You can use the Vagrant VM config shared [here](https://github.com/falcosecurity/cncf-green-review-testing/issues/7).

Proceed by executing the following setup commands:

```bash
kubectl create namespace falco;
kubectl create namespace benchmark;
kubectl get nodes;

kubectl label nodes --all node-role.kubernetes.io/benchmark=true --overwrite;

kubectl get nodes --show-labels;
```

Apply the configurations by executing the following command:

```bash
kubectl apply -k ./kustomize/falco-driver/modern_ebpf
# Tear-down
kubectl delete -k ./kustomize/falco-driver/modern_ebpf

# kubectl apply -k ./kustomize/falco-driver/ebpf
# kubectl delete -k ./kustomize/falco-driver/ebpf

# WARNING: Testing kernel modules on a local dev box is more risky, 
# remember to unload the module `sudo rmmod falco`
# Testing kmod within a smaller VM with minikube likely crashes, only test w/ minikube on a larger native box

# kubectl apply -k ./kustomize/falco-driver/kmod
# kubectl delete -k ./kustomize/falco-driver/kmod
```


```bash
kubectl apply -f ./benchmark-tests/falco-benchmark-tests.yaml
# Tear-down
kubectl delete -f ./benchmark-tests/falco-benchmark-tests.yaml
```

Verify if the pods are up and running (Note that the output below is not regularly updated, and there might be more pods and containers running than displayed): 

```bash
kubectl get pods -n falco

NAME                             READY   STATUS    RESTARTS   AGE
falco-driver-modern-ebpf-k279z   1/1     Running   0          5m56s
```

```bash
kubectl get pods -n benchmark

NAME                                     READY   STATUS    RESTARTS   AGE
falco-event-generator-5d849f46fd-7kfmc   1/1     Running   0          43s
redis-866d5cc6f6-qrr7q                   2/2     Running   0          43s
stress-ng-767cbd6477-72v8b               1/1     Running   0          43s
```

To drop interactively into the Falco container, execute the `exec` command as follows:

```bash
kubectl -n falco exec -it falco-driver-modern-ebpf-k279z -c falco -- bash
```

Check Prometheus metrics:

```bash
curl http://127.0.0.1:8765/metrics
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
kubectl -n falco describe pod POD_NAME
```

</details>

</br>

## Versioning

The respective `CONFIG_VERSION` environment variable within the daemonset deployment contains the semver-compatible version of the testbed setup. We inject it (as a suffix) into the `FALCO_HOSTNAME` environment variable to maintain a version record extending beyond the Falco version in the native Falco metrics. Every merge into the main branch necessitates a (mostly minor) version increment.

## How to Contribute

Please refer to the [contributing guide](https://github.com/falcosecurity/.github/blob/main/CONTRIBUTING.md) and the [code of conduct](https://github.com/falcosecurity/evolution/CODE_OF_CONDUCT.md) for more information on how to contribute.

## License

This project is licensed to you under the [Apache 2.0](./COPYING) open source license.

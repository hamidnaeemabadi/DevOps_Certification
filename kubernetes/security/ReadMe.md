## Table of Contents:
- [Table of Contents:](#table-of-contents)
  - [Kubernetes Audit Logs](#kubernetes-audit-logs)
    - [Configuring Kubernetes Auditing](#configuring-kubernetes-auditing)
  - [Kube-bench](#kube-bench)
    - [Install kube-bench:](#install-kube-bench)
    - [Running kube-bench:](#running-kube-bench)
    - [Running Kube-bench In a Pod:](#running-kube-bench-in-a-pod)
  - [Trivy](#trivy)
    - [Installing Trivy and Enable shell completion](#installing-trivy-and-enable-shell-completion)
    - [Kubernetes Scanning Tutorial](#kubernetes-scanning-tutorial)
    - [Trivy Operator](#trivy-operator)
  - [Kubernetes RBAC](#kubernetes-rbac)
    - [Create read only user on namespace](#create-read-only-user-on-namespace)
    - [Create read only user on cluster](#create-read-only-user-on-cluster)
  - [Network policy](#network-policy)
    - [Network policy sample scenario](#network-policy-sample-scenario)

### Kubernetes Audit Logs

Kubernetes auditing provides a security-relevant, chronological set of records documenting the sequence of actions in a cluster. The cluster audits the activities generated by users, by applications that use the Kubernetes API, and by the control plane itself.

Auditing allows cluster administrators to answer the following questions:

- what happened?
- when did it happen?
- who initiated it?
- on what did it happen?
- where was it observed?
- from where was it initiated?
- to where was it going?

Audit records begin their lifecycle inside the kube-apiserver component. Each request on each stage of its execution generates an audit event, which is then pre-processed according to a certain policy and written to a backend. The policy determines what's recorded and the backends persist the records. The current backend implementations include logs files and webhooks.

Each request can be recorded with an associated stage. The defined stages are:

- RequestReceived - The stage for events generated as soon as the audit handler receives the request, and before it is delegated down the handler chain.
- ResponseStarted - Once the response headers are sent, but before the response body is sent. This stage is only generated for long-running requests (e.g. watch).
- ResponseComplete - The response body has been completed and no more bytes will be sent.
Panic - Events generated when a panic occurred.

#### Configuring Kubernetes Auditing

**Step 1:** Connect to control-plane
Connect to the control-plane node and create a directory to host the audit policy as well as audit logs.
```bash
mkdir /etc/kubernetes/audit
```

**Step 2:** Create an audit policy
Create an audit policy file named /etc/kubernetes/audit/policy.yaml with the following data:
```bash
cat <<EOF > /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:

- level: None
  verbs: ["get", "watch", "list"]

- level: None
  resources:
  - group: "" # core
    resources: ["events"]

- level: None
  users:
  - "system:kube-scheduler"
  - "system:kube-proxy"
  - "system:apiserver"
  - "system:kube-controller-manager"
  - "system:serviceaccount:gatekeeper-system:gatekeeper-admin"

- level: None
  userGroups: ["system:nodes"]

- level: RequestResponse
EOF
```

**Step 3:** Add required entries
Add following entries in /etc/kubernetes/manifests/kube-apiserver.yaml:

Add these line under command section:
```bash
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/etc/kubernetes/audit/audit.log
    - --audit-log-maxsize=500
    - --audit-log-maxbackup=10
    - --audit-log-maxage=30
```

Now mount this volume by adding the following data under the volumeMounts section:
```bash
- mountPath: /etc/kubernetes/audit
  name: audit
```

Scroll down and add a volume under the volumes section:
```bash
- hostPath:
    path: /etc/kubernetes/audit
    type: DirectoryOrCreate
  name: audit
```

**Step 4:** Restart api-server pod
```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /opt/
sleep 5
mv /opt/kube-apiserver.yaml /etc/kubernetes/manifests
```

**Step 5:** check audit log
```bash
tail -f /etc/kubernetes/audit/audit.log | jq
```
[Reference](https://signoz.io/blog/kubernetes-audit-logs/)

### Kube-bench

kube-bench is a tool that checks whether Kubernetes is deployed securely by running the checks documented in the CIS Kubernetes Benchmark.

#### Install kube-bench:

**Download and Install binaries `kube-bench`**
It is possible to manually install and run kube-bench release binaries. In order to do that, you must have access to your Kubernetes cluster nodes. Note that if you're using one of the managed Kubernetes services (e.g. EKS, AKS, GKE, ACK, OCP), you will not have access to the master nodes of your cluster and you can’t perform any tests on the master nodes.

**First, log into one of the nodes using SSH.**
Install kube-bench binary for your platform using the commands below. Note that there may be newer releases available. See [releases page](https://github.com/aquasecurity/kube-bench/releases).

For Ubuntu/Debian:

```bash
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.7.2/kube-bench_0.7.2_linux_amd64.deb -o kube-bench_0.7.2_linux_amd64.deb

sudo apt install ./kube-bench_0.7.2_linux_amd64.deb -f
```

#### Running kube-bench:

If you run kube-bench directly from the command line you may need to be root / sudo to have access to all the config files.

By default kube-bench attempts to auto-detect the running version of Kubernetes, and map this to the corresponding CIS Benchmark version. For example, Kubernetes version 1.15 is mapped to CIS Benchmark version cis-1.15 which is the benchmark version valid for Kubernetes 1.15.

kube-bench also attempts to identify the components running on the node, and uses this to determine which tests to run (for example, only running the master node tests if the node is running an API server).

**Kubernetes version 1.28 CIS Kubernetes Benchmark:**
The Center for Internet Security (CIS) publishes the CIS Kubernetes Benchmark as a framework of specific steps to configure Kubernetes more securely and with standards that are commensurate to various industry regulations. This document contains the results of the version 1.5 CIS Kubernetes benchmark for clusters that run Kubernetes version 1.28. For more information or help understanding the benchmark, see Using the benchmark. [link](https://cloud.ibm.com/docs/containers?topic=containers-cis-benchmark)

```bash
kube-bench run
# OR
kube-bench run --config-dir /etc/kube-bench/cfg --benchmark cis-1.5
```

then check report.

#### Running Kube-bench In a Pod:

Another method to run kube-bench is by deploying it as a Kubernetes job pod. This method is particularly useful for running CIS benchmarks on managed Kubernetes clusters where root access to the control plane or worker nodes is not available.

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

Then kube-bench report will be available in the pod logs. First List the pod

```bash
kubectl get pods
```

Now use the pod name to get the logs. Replace kube-bench-4j2bs with your pod name.

```bash
kubectl logs kube-bench-4j2bs
```

You can also export the kube-bench log to a file

```bash
kubectl logs kube-bench-4j2bs > kube-bench.report
```

### Trivy

Trivy is a comprehensive and versatile security scanner. Trivy has scanners that look for security issues, and targets where it can find those issues.

Targets (what Trivy can scan):

- Container Image
- Filesystem
- Git Repository (remote)
- Virtual Machine Image
- Kubernetes
- AWS

Scanners (what Trivy can find there):

- OS packages and software dependencies in use (SBOM)
- Known vulnerabilities (CVEs)
- IaC issues and misconfigurations
- Sensitive information and secrets
- Software licenses

Trivy supports most popular programming languages, operating systems, and platforms. For a complete list, see the Scanning [Coverage](https://aquasecurity.github.io/trivy/v0.50/docs/coverage/) page.

#### Installing Trivy and Enable shell completion

In this section you will find an aggregation of the different ways to install Trivy. installations are listed as either "official" or "community". Official integrations are developed by the core Trivy team and supported by it. Community integrations are integrations developed by the community, and collected here for your convenience. For support or questions about community integrations, please contact the original developers.

**Debian/Ubuntu (Official)**

Add repository setting to /etc/apt/sources.list.d.

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

Enable shell completion:
```
echo "autoload -U compinit; compinit" >> ~/.zshrc
source <(trivy completion zsh); compdef _trivy trivy
trivy completion zsh > "${fpath[1]}/_trivy"
```

#### Kubernetes Scanning Tutorial

**Prerequisites**
To test the following commands yourself, make sure that you’re connected to a Kubernetes cluster. A simple kind, a Docker-Desktop or microk8s cluster will do. In our case, we’ll use a one-node kind cluster.

Pro tip: The output of the commands will be even more interesting if you have some workloads running in your cluster.

**Cluster Scanning**
Trivy K8s is great to get an overview of all the vulnerabilities and misconfiguration issues or to scan specific workloads that are running in your cluster. You would want to use the Trivy K8s command either on your own local cluster or in your CI/CD pipeline post deployments.

The `trivy k8s` command is part of the Trivy CLI.

With the following command, we can scan our entire Kubernetes cluster for vulnerabilities and get a summary of the scan:

    trivy k8s --report=summary cluster

To get detailed information for all your resources, just replace ‘summary’ with ‘all’:

    trivy k8s --report=all cluster

However, we recommend displaying all information only in case you scan a specific namespace or resource since you can get overwhelmed with additional details.

Furthermore, we can specify the namespace that Trivy is supposed to scan to focus on specific resources in the scan result:

    trivy k8s -n kube-system --report=summary cluster

Again, if you’d like to receive additional details, use the ‘--report=all’ flag:

    trivy k8s -n kube-system --report=all cluster

Like with scanning for vulnerabilities, we can also filter in-cluster security issues by severity of the vulnerabilities:

    trivy k8s --severity=CRITICAL --report=summary cluster

Note that you can use any of the Trivy flags on the Trivy K8s command.

With the Trivy K8s command, you can also scan specific workloads that are running within your cluster, such as our deployment:

    trivy k8s --namespace  app --report=summary deployments/react-application

#### Trivy Operator
The Trivy K8s command is an imperative model to scan resources. We wouldn’t want to manually scan each resource across different environments. The larger the cluster and the more workloads are running in it, the more error-prone this process would become. With the Trivy Operator, we can automate the scanning process after the deployment.

The Trivy Operator follows the Kubernetes Operator Model. Operators automate human actions, and the result of the task is saved as custom resource definitions (CRDs) within your cluster.

This has several benefits:

- Trivy Operator is installed CRDs in our cluster. As a result, all our resources, including our security scanner and its scan results, are Kubernetes resources. This makes it much easier to integrate the Trivy Operator directly into our existing processes, such as connecting Trivy with Prometheus, a monitoring system.

- The Trivy Operator will automatically scan your resources every six hours. You can set up automatic alerting in case new critical security issues are discovered.

- The CRDs can be both machine and human-readable depending on which applications consume the CRDs. This allows for more versatile applications of the Trivy operator.

There are several ways that you can install the Trivy Operator in your cluster. In this guide, we’re going to use the Helm installation based on the [following documentation](https://aquasecurity.github.io/trivy/v0.50/docs/target/kubernetes/#trivy-operator).

Please follow the Trivy Operator documentation for further information on:

- [Installation of the Trivy Operator](https://aquasecurity.github.io/trivy-operator/latest/getting-started/installation/)
- [Getting started guide](https://aquasecurity.github.io/trivy-operator/latest/getting-started/quick-start/)

### Kubernetes RBAC
Kubernetes Role-based access control (RBAC) is a key security control to ensure that cluster users and workloads have only the access to resources required to execute their roles. It is important to ensure that, when designing permissions for cluster users, the cluster administrator understands the areas where privilege escalation could occur, to reduce the risk of excessive access leading to security incidents.

**General good practice:**
- Least privilege
- Minimize distribution of privileged tokens
- Hardening

#### Create read only user on namespace

**Step 1:** create namespace
```bash
kubectl create namespace mon
```

**Step 2:** create service account
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ahmad
  namespace: mon
EOF
```

**Step 2:** create role
api groups: all
resource: all
verb: get, watch, list
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mon
  name: reader
rules:
- apiGroups: ["*"] # "" indicates the core API group
  resources: ["*"]
  verbs: ["get", "watch", "list"]
EOF
```

**Step 3:** create cluster role binding
cluster role: reader
service account: ahmad
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: reader
  namespace: mon
roleRef:
  kind: Role
  name: reader
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: ahmad
  namespace: mon
EOF
```

**Step 4:** get token for ahmad service account
```bash
kubectl -n mon create token ahmad
```

#### Create read only user on cluster

**Step 1:** create service account
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ahmad
  namespace: default
EOF
```

**Step 2:** create cluster role
api groups: all
resource: all
verb: get, watch, list
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: reader
rules:
- apiGroups: ["*"] # "" indicates the core API group
  resources: ["*"]
  verbs: ["get", "watch", "list"]
EOF
```

**Step 3:** create cluster role binding
cluster role: reader
service account: ahmad
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: reader
subjects:
- kind: ServiceAccount
  name: ahmad
  namespace: default
EOF
```

**Step 4:** get token for ahmad service account
```bash
kubectl -n default create token ahmad
```

### Network policy

If you want to control traffic flow at the IP address or port level for TCP, UDP, and SCTP protocols, then you might consider using Kubernetes NetworkPolicies for particular applications in your cluster. NetworkPolicies are an application-centric construct which allow you to specify how a pod is allowed to communicate with various network "entities" (we use the word "entity" here to avoid overloading the more common terms such as "endpoints" and "services", which have specific Kubernetes connotations) over the network. NetworkPolicies apply to a connection with a pod on one or both ends, and are not relevant to other connections.

The entities that a Pod can communicate with are identified through a combination of the following three identifiers:

- Other pods that are allowed (exception: a pod cannot block access to itself)
- Namespaces that are allowed
- IP blocks (exception: traffic to and from the node where a Pod is running is always allowed, regardless of the IP address of the Pod or the node)

When defining a pod- or namespace-based NetworkPolicy, you use a selector to specify what traffic is allowed to and from the Pod(s) that match the selector.

Meanwhile, when IP-based NetworkPolicies are created, we define policies based on IP blocks (CIDR ranges).

#### Network policy sample scenario

Create an nginx deployment and expose it via a service
To see how Kubernetes network policy works, start off by creating an nginx Deployment.

```bash
kubectl create deployment nginx --image=nginx
```

Expose the Deployment through a Service called nginx.

```bash
kubectl expose deployment nginx --port=80
```

The above commands create a Deployment with an nginx Pod and expose the Deployment through a Service named nginx. The nginx Pod and Deployment are found in the default namespace.

```bash
kubectl get svc,pod

# sample output
NAME                        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/kubernetes          10.100.0.1    <none>        443/TCP    46m
service/nginx               10.100.0.16   <none>        80/TCP     33s

NAME                        READY         STATUS        RESTARTS   AGE
pod/nginx-701339712-e0qfq   1/1           Running       0          35s
```

Test the service by accessing it from another Pod
You should be able to access the new nginx service from other Pods. To access the nginx Service from another Pod in the default namespace, start a busybox container:

```bash
kubectl run busybox --rm -ti --image=busybox:1.28 -- /bin/sh
```

In your shell, run the following command:

```bash
wget --spider --timeout=1 nginx

# Sample output
Connecting to nginx (10.100.0.16:80)
remote file exists
```

Limit access to the nginx service
To limit the access to the nginx service so that only Pods with the label access: true can query it, create a NetworkPolicy object as follows:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
EOF
```

Test access to the service when access label is not defined
When you attempt to access the nginx Service from a Pod without the correct labels, the request times out:

```bash
kubectl run busybox --rm -ti --image=busybox:1.28 -- /bin/sh
```

In your shell, run the command:

```bash
wget --spider --timeout=1 nginx

# Sample output
Connecting to nginx (10.100.0.16:80)
wget: download timed out
```

Define access label and test again
You can create a Pod with the correct labels to see that the request is allowed:

```bash
kubectl run busybox --rm -ti --labels="access=true" --image=busybox:1.28 -- /bin/sh
```

In your shell, run the command:

```bash
wget --spider --timeout=1 nginx

# Sample output
Connecting to nginx (10.100.0.16:80)
remote file exists
```

[References](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
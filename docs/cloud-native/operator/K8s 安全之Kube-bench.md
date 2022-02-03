# K8s 安全至Kube-bench

# 一 简介

Kube-Bench是一款针对Kubernete的安全检测工具，从本质上来说，Kube-Bench是一个基于Go开发的应用程序，它可以帮助研究人员对部署的Kubernete进行安全检测，安全检测原则遵循[CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/)。

测试规则需要通过YAML文件进行配置，因此我们可以轻松更新该工具的测试规则。

https://github.com/aquasecurity/kube-bench#download-and-install-binaries

# 二 使用

## 2.1 安装部署

* 直接二进制安装

```shell
[root@master software]# wget https://github.com/aquasecurity/kube-bench/releases/download/v0.4.0/kube-bench_0.4.0_linux_amd64.tar.gz
[root@master software]# tar -zxvf kube-bench_0.4.0_linux_amd64.tar.gz
[root@master software]# mv kube-bench /usr/bin/
```

## 2.2 安全检测

### 2.1.1 检测master组件

```shell


[root@master software]# ./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml master
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 Master Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
[PASS] 1.1.3 Ensure that the controller manager pod specification file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.4 Ensure that the controller manager pod specification file ownership is set to root:root (Automated)
[PASS] 1.1.5 Ensure that the scheduler pod specification file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.6 Ensure that the scheduler pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.7 Ensure that the etcd pod specification file permissions are set to 644 or more restrictive (Automated)
[FAIL] 1.1.8 Ensure that the etcd pod specification file ownership is set to root:root (Automated)
[WARN] 1.1.9 Ensure that the Container Network Interface file permissions are set to 644 or more restrictive (Manual)
[WARN] 1.1.10 Ensure that the Container Network Interface file ownership is set to root:root (Manual)
[FAIL] 1.1.11 Ensure that the etcd data directory permissions are set to 700 or more restrictive (Automated)
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
[PASS] 1.1.13 Ensure that the admin.conf file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.14 Ensure that the admin.conf file ownership is set to root:root (Automated)
[PASS] 1.1.15 Ensure that the scheduler.conf file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.16 Ensure that the scheduler.conf file ownership is set to root:root (Automated)
[PASS] 1.1.17 Ensure that the controller-manager.conf file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.18 Ensure that the controller-manager.conf file ownership is set to root:root (Automated)
[FAIL] 1.1.19 Ensure that the Kubernetes PKI directory and file ownership is set to root:root (Automated)
[WARN] 1.1.20 Ensure that the Kubernetes PKI certificate file permissions are set to 644 or more restrictive (Manual)
[WARN] 1.1.21 Ensure that the Kubernetes PKI key file permissions are set to 600 (Manual)
[INFO] 1.2 API Server
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
[PASS] 1.2.2 Ensure that the --basic-auth-file argument is not set (Automated)
[PASS] 1.2.3 Ensure that the --token-auth-file parameter is not set (Automated)
[PASS] 1.2.4 Ensure that the --kubelet-https argument is set to true (Automated)
[PASS] 1.2.5 Ensure that the --kubelet-client-certificate and --kubelet-client-key arguments are set as appropriate (Automated)
[FAIL] 1.2.6 Ensure that the --kubelet-certificate-authority argument is set as appropriate (Automated)
[PASS] 1.2.7 Ensure that the --authorization-mode argument is not set to AlwaysAllow (Automated)
[PASS] 1.2.8 Ensure that the --authorization-mode argument includes Node (Automated)
[PASS] 1.2.9 Ensure that the --authorization-mode argument includes RBAC (Automated)
[WARN] 1.2.10 Ensure that the admission control plugin EventRateLimit is set (Manual)
[PASS] 1.2.11 Ensure that the admission control plugin AlwaysAdmit is not set (Automated)
[WARN] 1.2.12 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
[WARN] 1.2.13 Ensure that the admission control plugin SecurityContextDeny is set if PodSecurityPolicy is not used (Manual)
[PASS] 1.2.14 Ensure that the admission control plugin ServiceAccount is set (Automated)
[PASS] 1.2.15 Ensure that the admission control plugin NamespaceLifecycle is set (Automated)
[FAIL] 1.2.16 Ensure that the admission control plugin PodSecurityPolicy is set (Automated)
[PASS] 1.2.17 Ensure that the admission control plugin NodeRestriction is set (Automated)
[PASS] 1.2.18 Ensure that the --insecure-bind-address argument is not set (Automated)
[PASS] 1.2.19 Ensure that the --insecure-port argument is set to 0 (Automated)
[PASS] 1.2.20 Ensure that the --secure-port argument is not set to 0 (Automated)
[PASS] 1.2.21 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 1.2.22 Ensure that the --audit-log-path argument is set (Automated)
[FAIL] 1.2.23 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)
[FAIL] 1.2.24 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)
[FAIL] 1.2.25 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)
[PASS] 1.2.26 Ensure that the --request-timeout argument is set as appropriate (Automated)
[PASS] 1.2.27 Ensure that the --service-account-lookup argument is set to true (Automated)
[PASS] 1.2.28 Ensure that the --service-account-key-file argument is set as appropriate (Automated)
[PASS] 1.2.29 Ensure that the --etcd-certfile and --etcd-keyfile arguments are set as appropriate (Automated)
[PASS] 1.2.30 Ensure that the --tls-cert-file and --tls-private-key-file arguments are set as appropriate (Automated)
[PASS] 1.2.31 Ensure that the --client-ca-file argument is set as appropriate (Automated)
[PASS] 1.2.32 Ensure that the --etcd-cafile argument is set as appropriate (Automated)
[WARN] 1.2.33 Ensure that the --encryption-provider-config argument is set as appropriate (Manual)
[WARN] 1.2.34 Ensure that encryption providers are appropriately configured (Manual)
[WARN] 1.2.35 Ensure that the API Server only makes use of Strong Cryptographic Ciphers (Manual)
[INFO] 1.3 Controller Manager
[PASS] 1.3.1 Ensure that the --terminated-pod-gc-threshold argument is set as appropriate (Manual)
[PASS] 1.3.2 Ensure that the --profiling argument is set to false (Automated)
[PASS] 1.3.3 Ensure that the --use-service-account-credentials argument is set to true (Automated)
[PASS] 1.3.4 Ensure that the --service-account-private-key-file argument is set as appropriate (Automated)
[PASS] 1.3.5 Ensure that the --root-ca-file argument is set as appropriate (Automated)
[FAIL] 1.3.6 Ensure that the RotateKubeletServerCertificate argument is set to true (Automated)
[FAIL] 1.3.7 Ensure that the --bind-address argument is set to 127.0.0.1 (Automated)
[INFO] 1.4 Scheduler
[FAIL] 1.4.1 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 1.4.2 Ensure that the --bind-address argument is set to 127.0.0.1 (Automated)

== Remediations ==
1.1.7 Run the below command (based on the file location on your system) on the master node.
For example,
chmod 644 /etc/kubernetes/manifests/etcd.yaml

1.1.8 Run the below command (based on the file location on your system) on the master node.
For example,
chown root:root /etc/kubernetes/manifests/etcd.yaml

1.1.9 Run the below command (based on the file location on your system) on the master node.
For example,
chmod 644 <path/to/cni/files>

1.1.10 Run the below command (based on the file location on your system) on the master node.
For example,
chown root:root <path/to/cni/files>

1.1.11 On the etcd server node, get the etcd data directory, passed as an argument --data-dir,
from the below command:
ps -ef | grep etcd
Run the below command (based on the etcd data directory found above). For example,
chmod 700 /var/lib/etcd

1.1.12 On the etcd server node, get the etcd data directory, passed as an argument --data-dir,
from the below command:
ps -ef | grep etcd
Run the below command (based on the etcd data directory found above).
For example, chown etcd:etcd /var/lib/etcd

1.1.19 Run the below command (based on the file location on your system) on the master node.
For example,
chown -R root:root /etc/kubernetes/pki/

1.1.20 audit test did not run: failed to run: "find /etc/kubernetes/pki -name '*.crt' | xargs stat -c permissions=%a", output: "stat: missing operand\nTry 'stat --help' for more information.\n", error: exit status 123
1.1.21 audit test did not run: failed to run: "find /etc/kubernetes/pki -name '*.key' | xargs stat -c permissions=%a", output: "stat: missing operand\nTry 'stat --help' for more information.\n", error: exit status 123
1.2.1 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the below parameter.
--anonymous-auth=false

1.2.6 Follow the Kubernetes documentation and setup the TLS connection between
the apiserver and kubelets. Then, edit the API server pod specification file
/etc/kubernetes/manifests/kube-apiserver.yaml on the master node and set the
--kubelet-certificate-authority parameter to the path to the cert file for the certificate authority.
--kubelet-certificate-authority=<ca-string>

1.2.10 Follow the Kubernetes documentation and set the desired limits in a configuration file.
Then, edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
and set the below parameters.
--enable-admission-plugins=...,EventRateLimit,...
--admission-control-config-file=<path/to/configuration/file>

1.2.12 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --enable-admission-plugins parameter to include
AlwaysPullImages.
--enable-admission-plugins=...,AlwaysPullImages,...

1.2.13 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --enable-admission-plugins parameter to include
SecurityContextDeny, unless PodSecurityPolicy is already in place.
--enable-admission-plugins=...,SecurityContextDeny,...

1.2.16 Follow the documentation and create Pod Security Policy objects as per your environment.
Then, edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --enable-admission-plugins parameter to a
value that includes PodSecurityPolicy:
--enable-admission-plugins=...,PodSecurityPolicy,...
Then restart the API Server.

1.2.22 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --audit-log-path parameter to a suitable path and
file where you would like audit logs to be written, for example:
--audit-log-path=/var/log/apiserver/audit.log

1.2.23 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --audit-log-maxage parameter to 30 or as an appropriate number of days:
--audit-log-maxage=30

1.2.24 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --audit-log-maxbackup parameter to 10 or to an appropriate
value.
--audit-log-maxbackup=10

1.2.25 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --audit-log-maxsize parameter to an appropriate size in MB.
For example, to set it as 100 MB:
--audit-log-maxsize=100

1.2.33 Follow the Kubernetes documentation and configure a EncryptionConfig file.
Then, edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --encryption-provider-config parameter to the path of that file: --encryption-provider-config=</path/to/EncryptionConfig/File>

1.2.34 Follow the Kubernetes documentation and configure a EncryptionConfig file.
In this file, choose aescbc, kms or secretbox as the encryption provider.

1.2.35 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the below parameter.
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM
_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM
_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM
_SHA384

1.3.6 Edit the Controller Manager pod specification file /etc/kubernetes/manifests/kube-controller-manager.yaml
on the master node and set the --feature-gates parameter to include RotateKubeletServerCertificate=true.
--feature-gates=RotateKubeletServerCertificate=true

1.3.7 Edit the Controller Manager pod specification file /etc/kubernetes/manifests/kube-controller-manager.yaml
on the master node and ensure the correct value for the --bind-address parameter

1.4.1 Edit the Scheduler pod specification file /etc/kubernetes/manifests/kube-scheduler.yaml file
on the master node and set the below parameter.
--profiling=false

1.4.2 Edit the Scheduler pod specification file /etc/kubernetes/manifests/kube-scheduler.yaml
on the master node and ensure the correct value for the --bind-address parameter


== Summary ==
39 checks PASS
15 checks FAIL
11 checks WARN
0 checks INFO
```

### 2.1.2 检测node组件

```shell
[root@master software]# kube-bench --config-dir /data/software/kube-bench/cfg --config /data/software/kube-bench/cfg/config.yaml node
[INFO] 4 Worker Node Security Configuration
[INFO] 4.1 Worker Node Configuration Files
[PASS] 4.1.1 Ensure that the kubelet service file permissions are set to 644 or more restrictive (Automated)
[PASS] 4.1.2 Ensure that the kubelet service file ownership is set to root:root (Automated)
[WARN] 4.1.3 If proxy kubeconfig file exists ensure permissions are set to 644 or more restrictive (Manual)
[WARN] 4.1.4 Ensure that the proxy kubeconfig file ownership is set to root:root (Manual)
[PASS] 4.1.5 Ensure that the --kubeconfig kubelet.conf file permissions are set to 644 or more restrictive (Automated)
[PASS] 4.1.6 Ensure that the --kubeconfig kubelet.conf file ownership is set to root:root (Manual)
[PASS] 4.1.7 Ensure that the certificate authorities file permissions are set to 644 or more restrictive (Manual)
[PASS] 4.1.8 Ensure that the client certificate authorities file ownership is set to root:root (Manual)
[PASS] 4.1.9 Ensure that the kubelet --config configuration file has permissions set to 644 or more restrictive (Automated)
[PASS] 4.1.10 Ensure that the kubelet --config configuration file ownership is set to root:root (Automated)
[INFO] 4.2 Kubelet
[PASS] 4.2.1 Ensure that the anonymous-auth argument is set to false (Automated)
[PASS] 4.2.2 Ensure that the --authorization-mode argument is not set to AlwaysAllow (Automated)
[PASS] 4.2.3 Ensure that the --client-ca-file argument is set as appropriate (Automated)
[PASS] 4.2.4 Ensure that the --read-only-port argument is set to 0 (Manual)
[PASS] 4.2.5 Ensure that the --streaming-connection-idle-timeout argument is not set to 0 (Manual)
[FAIL] 4.2.6 Ensure that the --protect-kernel-defaults argument is set to true (Automated)
[PASS] 4.2.7 Ensure that the --make-iptables-util-chains argument is set to true (Automated)
[WARN] 4.2.8 Ensure that the --hostname-override argument is not set (Manual)
[WARN] 4.2.9 Ensure that the --event-qps argument is set to 0 or a level which ensures appropriate event capture (Manual)
[WARN] 4.2.10 Ensure that the --tls-cert-file and --tls-private-key-file arguments are set as appropriate (Manual)
[PASS] 4.2.11 Ensure that the --rotate-certificates argument is not set to false (Manual)
[WARN] 4.2.12 Verify that the RotateKubeletServerCertificate argument is set to true (Manual)
[WARN] 4.2.13 Ensure that the Kubelet only makes use of Strong Cryptographic Ciphers (Manual)

== Remediations ==
4.1.3 Run the below command (based on the file location on your system) on the each worker node.
For example,
chmod 644 /etc/kubernetes/proxy.conf

4.1.4 Run the below command (based on the file location on your system) on the each worker node.
For example, chown root:root /etc/kubernetes/proxy.conf

4.2.6 If using a Kubelet config file, edit the file to set protectKernelDefaults: true.
If using command line arguments, edit the kubelet service file
/etc/systemd/system/kubelet.service on each worker node and
set the below parameter in KUBELET_SYSTEM_PODS_ARGS variable.
--protect-kernel-defaults=true
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service

4.2.8 Edit the kubelet service file /etc/systemd/system/kubelet.service
on each worker node and remove the --hostname-override argument from the
KUBELET_SYSTEM_PODS_ARGS variable.
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service

4.2.9 If using a Kubelet config file, edit the file to set eventRecordQPS: to an appropriate level.
If using command line arguments, edit the kubelet service file
/etc/systemd/system/kubelet.service on each worker node and
set the below parameter in KUBELET_SYSTEM_PODS_ARGS variable.
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service

4.2.10 If using a Kubelet config file, edit the file to set tlsCertFile to the location
of the certificate file to use to identify this Kubelet, and tlsPrivateKeyFile
to the location of the corresponding private key file.
If using command line arguments, edit the kubelet service file
/etc/systemd/system/kubelet.service on each worker node and
set the below parameters in KUBELET_CERTIFICATE_ARGS variable.
--tls-cert-file=<path/to/tls-certificate-file>
--tls-private-key-file=<path/to/tls-key-file>
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service

4.2.12 Edit the kubelet service file /etc/systemd/system/kubelet.service
on each worker node and set the below parameter in KUBELET_CERTIFICATE_ARGS variable.
--feature-gates=RotateKubeletServerCertificate=true
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service

4.2.13 If using a Kubelet config file, edit the file to set TLSCipherSuites: to
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_128_GCM_SHA256
or to a subset of these values.
If using executable arguments, edit the kubelet service file
/etc/systemd/system/kubelet.service on each worker node and
set the --tls-cipher-suites parameter as follows, or to a subset of these values.
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_128_GCM_SHA256
Based on your system, restart the kubelet service. For example:
systemctl daemon-reload
systemctl restart kubelet.service


== Summary ==
15 checks PASS
1 checks FAIL
7 checks WARN
0 checks INFO
```

# 三 支持版本

Kube-Bench支持的Kubernete测试规则定义在CIS Kubernetes Benchmark之中：

| **CIS Kubernetes Benchmark**                                 | **kube-bench****配置** | **Kubernetes****版本** |
| ------------------------------------------------------------ | ---------------------- | ---------------------- |
| [1.3.0](https://workbench.cisecurity.org/benchmarks/602)     | cis-1.3                | 1.11-1.12              |
| [1.4.1](https://workbench.cisecurity.org/benchmarks/2351)    | cis-1.4                | 1.13-1.14              |
| [1.5.0](https://workbench.cisecurity.org/benchmarks/1370)    | cis-1.5                | 1.15-                  |
| [GKE 1.0.0](https://workbench.cisecurity.org/benchmarks/4536) | gke-1.0                | GKE                    |
| [EKS 1.0.0](https://workbench.cisecurity.org/benchmarks/5190) | eks-1.0                | EKS                    |
| Red Hat OpenShift hardening guide                            | rh-0.7                 | OCP 3.10-3.11          |

# 四 注意事项

- Kube-Bench尽可能地实现了[CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/)，如果kube bench没有正确执行安全基准测试。
- Kubernete版本和CIS基准测试版本之间没有一对一的映射。，以查看基准测试的不同版本包含哪些Kubernetes版本。
- Kube-Bench无法检查受管集群的主节点，例如GKE、EKS和AKS，因为Kube-Bench不能访问这些节点。不过，Kube-Bench在这些环境中仍然可以检查worker节点配置。

# 五 反思

使用kube-bench可以快速的检测k8s中存在的问题，可以针对修复意见进行修复，配合自定义定时任务，可以自己根据数据生成报表，邮件或其他通信软件通知，及时获取集群安全状态。

# 参考链接

* https://www.cisecurity.org/benchmark/kubernetes/

* https://www.cisecurity.org/cybersecurity-tools/
* https://github.com/aquasecurity/kube-bench
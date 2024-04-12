kubeadm
==============

参考文档 https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/


环境准备
~~~~~~~~~

准备三台Linux机器（本文以Ubuntu22.04LTS系统为例），三台机器之间能相互通信。

以下是本文使用的三台Ubuntu 22.04LTS：


.. list-table:: Kubeadm环境主机
   :header-rows: 1

   * - hostname
     - IP
     - system
     - memory
   * - k8s-master
     - 192.168.56.10
     - Ubuntu 22.04 LTS
     - 4GB
   * - k8s-worker1
     - 192.168.56.11
     - Ubuntu 22.04 LTS
     - 2GB
   * - k8s-worker2
     - 192.168.56.12
     - Ubuntu 22.04 LTS
     - 2GB

.. code-block:: markdown

  https://asciiflow.com/#/
  +--------------+      +----------------+      +---------------+
  |              |      |                |      |               |
  |              |      |                |      |               |
  |  k8s-master  |      |  k8s-worker1   |      |  k8s-worker2  |
  |              |      |                |      |               |
  |              |      |                |      |               |
  |              |      |                |      |               |
  | 192.168.56.10|      |192.168.56.11   |      | 192.168.56.12 |
  +------+-------+      +-------+--------+      +---------+-----+
        |                      |                         |
        |                      |                         |
        |                      |                         |
        |                      |                         |
        +----------------------+-------------------------+

.. warning::

   请注意上面准备的机器必须能够访问互联网，中国大陆的朋友要确保机器能访问Google

.. warning::

   如果你使用的是云服务提供的虚拟机，请确保把安全策略组配置好，确保三台机器之间可以访问任意端口，https://kubernetes.io/docs/reference/ports-and-protocols/



安装containerd, kubeadm, kubelet, kubectl
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在所有节点上运行下面的命令安装containerd和一些必要的工具。

.. code-block:: bash

  curl -fsSL https://raw.githubusercontent.com/xiaopeng163/learn-k8s-from-scratch/master/source/_code/k8s-install/install.sh -o install.sh
  sudo sh install.sh


.. literalinclude:: ../_code/k8s-install/install.sh
   :language: shell
   :linenos:

脚本结束以后， 在master节点上运行  ``apt list -a kubeadm`` 查看可用版本， 当前我们使用的版本是 ``1.29.2-1.1``

.. code-block:: bash

  $ apt list -a kubeadm
  Listing... Done
  kubeadm/unknown 1.29.2-1.1 amd64
  kubeadm/unknown 1.29.1-1.1 amd64
  kubeadm/unknown 1.29.0-1.1 amd64

  kubeadm/unknown,now 1.29.2-1.1 arm64 [installed]
  kubeadm/unknown 1.29.1-1.1 arm64
  kubeadm/unknown 1.29.0-1.1 arm64

  kubeadm/unknown 1.29.2-1.1 ppc64el
  kubeadm/unknown 1.29.1-1.1 ppc64el
  kubeadm/unknown 1.29.0-1.1 ppc64el

  kubeadm/unknown 1.29.2-1.1 s390x
  kubeadm/unknown 1.29.1-1.1 s390x
  kubeadm/unknown 1.29.0-1.1 s390x

在所有节点上运行下面的命令安装kubeadm/kubelet/kubectl，确保版本一致。

.. code-block:: bash

  sudo apt install  -y kubeadm=1.29.2-1.1 kubelet=1.29.2-1.1 kubectl=1.29.2-1.1


可以检查下kubeadm，kubelet，kubectl的安装情况,如果都能获取到版本号，说明安装成功。


.. code-block:: bash

    kubeadm version
    kubelet --version
    kubectl version

初始化master节点
~~~~~~~~~~~~~~~~~~~~~~

.. warning::

    以下操作都在master节点上进行。

可以先拉取集群所需要的images（可做可不做）

.. code-block:: bash

    sudo kubeadm config images pull

如果拉取成功，会看到类似下面的输出：

.. code-block:: bash

  [config/images] Pulled registry.k8s.io/kube-apiserver:v1.29.2
  [config/images] Pulled registry.k8s.io/kube-controller-manager:v1.29.2
  [config/images] Pulled registry.k8s.io/kube-scheduler:v1.29.2
  [config/images] Pulled registry.k8s.io/kube-proxy:v1.29.2
  [config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.1
  [config/images] Pulled registry.k8s.io/pause:3.9
  [config/images] Pulled registry.k8s.io/etcd:3.5.10-0

初始化Kubeadm

- ``--apiserver-advertise-address``  这个地址是本地用于和其他节点通信的IP地址
- ``--pod-network-cidr``  pod network 地址空间

.. code-block:: bash

    vagrant@k8s-master:~$ sudo kubeadm init --apiserver-advertise-address=192.168.56.10  --pod-network-cidr=10.244.0.0/16

最后一段的输出要保存好, 这一段指出后续需要做什么配置。

- 1. 准备 .kube
- 2. 部署pod network方案
- 3. 添加worker节点

.. code-block:: bash

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 192.168.56.10:6443 --token 0pdoeh.wrqchegv3xm3k1ow \
    --discovery-token-ca-cert-hash sha256:f4e693bde148f5c0ff03b66fb24c51f948e295775763e8c5c4e60d24ff57fe82

1. 配置 .kube

.. code-block:: bash

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

检查状态：

.. code-block:: bash

    kubectl get nodes
    kubectl get pods -A

得到的输出类似于：

.. code-block:: bash

  vagrant@k8s-master:~$ kubectl get nodes -o wide
  NAME         STATUS     ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
  k8s-master   NotReady   control-plane   3m10s   v1.29.2   10.211.55.4   <none>        Ubuntu 22.04.2 LTS   5.15.0-76-generic   containerd://1.6.28
  vagrant@k8s-master:~$ kubectl get pod -A
  NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
  kube-system   coredns-76f75df574-4l8fv             0/1     Pending   0          3m18s
  kube-system   coredns-76f75df574-ztqqx             0/1     Pending   0          3m18s
  kube-system   etcd-k8s-master                      1/1     Running   0          3m32s
  kube-system   kube-apiserver-k8s-master            1/1     Running   0          3m32s
  kube-system   kube-controller-manager-k8s-master   1/1     Running   0          3m33s
  kube-system   kube-proxy-4mzl6                     1/1     Running   0          3m18s
  kube-system   kube-scheduler-k8s-master            1/1     Running   0          3m32s

shell 自动补全(Bash)

more information can be found https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete

.. code-block:: bash

    source <(kubectl completion bash)
    echo "source <(kubectl completion bash)" >> ~/.bashrc


2. 部署pod network方案

去https://kubernetes.io/docs/concepts/cluster-administration/addons/ 选择一个network方案， 根据提供的具体链接去部署。


这里我们选择overlay的方案，名字叫 ``flannel`` 部署方法如下：

下载文件 https://raw.githubusercontent.com/flannel-io/flannel/v0.24.2/Documentation/kube-flannel.yml ，并进行如下修改：

.. code-block:: bash

    curl -LO https://raw.githubusercontent.com/flannel-io/flannel/v0.24.2/Documentation/kube-flannel.yml


确保network是我们配置的 --pod-network-cidr  10.244.0.0/16

.. code-block:: yaml

    net-conf.json: |
      {
        "Network": "10.244.0.0/16",
        "Backend": {
          "Type": "vxlan"
        }
      }

在 kube-flannel的容器args里，确保有iface=enp0s8, 其中enp0s8是我们的--apiserver-advertise-address=192.168.56.10 接口名

.. code-block:: yaml

    - name: kube-flannel
      image: docker.io/flannel/flannel:v0.24.2
      command:
      - /opt/bin/flanneld
      args:
      - --ip-masq
      - --kube-subnet-mgr
      - --iface=enp0s8


比如我们的机器，这个IP的接口名是 ``enp0s8``

.. code-block:: bash

  vagrant@k8s-master:~$ ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
  2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 02:9a:67:51:1e:b6 brd ff:ff:ff:ff:ff:ff
      inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
        valid_lft 85351sec preferred_lft 85351sec
      inet6 fe80::9a:67ff:fe51:1eb6/64 scope link
        valid_lft forever preferred_lft forever
  3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:59:c5:26 brd ff:ff:ff:ff:ff:ff
      inet 192.168.56.10/24 brd 192.168.56.255 scope global enp0s8
        valid_lft forever preferred_lft forever
      inet6 fe80::a00:27ff:fe59:c526/64 scope link
        valid_lft forever preferred_lft forever

把修改好的文件保存一个新文件，文件名flannel.yaml，上传到master节点，然后运行

.. code-block:: bash

  kubectl apply -f flannel.yaml

输出结果：

.. code-block:: bash

  vagrant@k8s-master:~$ kubectl apply -f flannel.yml
  namespace/kube-flannel created
  clusterrole.rbac.authorization.k8s.io/flannel created
  clusterrolebinding.rbac.authorization.k8s.io/flannel created
  serviceaccount/flannel created
  configmap/kube-flannel-cfg created
  daemonset.apps/kube-flannel-ds created

检查结果， 如果显示下面的结果，pod都是running的状态，说明我们的network方案部署成功。

.. code-block:: bash

  kubectl get pods -A

.. code-block:: bash

  NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
  kube-system   coredns-6d4b75cb6d-m5vms             1/1     Running   0          3h19m
  kube-system   coredns-6d4b75cb6d-mmdrx             1/1     Running   0          3h19m
  kube-system   etcd-k8s-master                      1/1     Running   0          3h19m
  kube-system   kube-apiserver-k8s-master            1/1     Running   0          3h19m
  kube-system   kube-controller-manager-k8s-master   1/1     Running   0          3h19m
  kube-system   kube-flannel-ds-blhqr                1/1     Running   0          3h18m
  kube-system   kube-proxy-jh4w5                     1/1     Running   0          3h17m
  kube-system   kube-scheduler-k8s-master            1/1     Running   0          3h19m

并且node是Ready

.. code-block:: bash

  kubectl get node -o wide
  NAME         STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
  k8s-master   Ready    control-plane   15m   v1.29.2   10.211.55.4   <none>        Ubuntu 22.04.2 LTS   5.15.0-76-generic   containerd://1.6.28

添加worker节点
~~~~~~~~~~~~~~~~~


添加worker节点非常简单，直接在worker节点上运行join即可，注意--token


.. code-block:: bash

  $ sudo kubeadm join 192.168.56.10:6443 --token 0pdoeh.wrqchegv3xm3k1ow \
    --discovery-token-ca-cert-hash sha256:f4e693bde148f5c0ff03b66fb24c51f948e295775763e8c5c4e60d24ff57fe82

.. warning::

  不小心忘记join的``token``和``discovery-token-ca-cert-hash`` 怎么办？

token 可以通过 ``kubeadm token list``获取到，比如 ``0pdoeh.wrqchegv3xm3k1ow``

.. code-block:: bash

  $ kubeadm token list
  TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
  0pdoeh.wrqchegv3xm3k1ow   23h         2022-07-19T20:13:00Z   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token

而 ``discovery-token-ca-cert-hash`` 可以通过

.. code-block:: bash

  openssl x509 -in /etc/kubernetes/pki/ca.crt -pubkey -noout |
  openssl pkey -pubin -outform DER |
  openssl dgst -sha256

结果类似于 (stdin)= d301f5ac98d4114cdbe930717705f3bc284243f443c4ff33d32c2cee01bf7945

最后在master节点查看node和pod结果。(比如我们有两个worker节点)

.. code-block:: bash

  vagrant@k8s-master:~$ kubectl get nodes -o wide
  NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
  k8s-master    Ready    control-plane   17m   v1.29.2   10.211.55.4   <none>        Ubuntu 22.04.2 LTS   5.15.0-76-generic   containerd://1.6.28
  k8s-worker1   Ready    <none>          50s   v1.29.2   10.211.55.5   <none>        Ubuntu 22.04.2 LTS   5.15.0-97-generic   containerd://1.6.28
  k8s-worker2   Ready    <none>          20s   v1.29.2   10.211.55.6   <none>        Ubuntu 22.04.2 LTS   5.15.0-76-generic   containerd://1.6.28
  vagrant@k8s-master:~$


pod的话，应该可以看到三个flannel，三个proxy的pod


.. code-block:: bash

  vagrant@k8s-master:~$ kubectl get pods -A
  NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
  kube-system   coredns-6d4b75cb6d-m5vms             1/1     Running   0          3h19m
  kube-system   coredns-6d4b75cb6d-mmdrx             1/1     Running   0          3h19m
  kube-system   etcd-k8s-master                      1/1     Running   0          3h19m
  kube-system   kube-apiserver-k8s-master            1/1     Running   0          3h19m
  kube-system   kube-controller-manager-k8s-master   1/1     Running   0          3h19m
  kube-system   kube-flannel-ds-blhqr                1/1     Running   0          3h18m
  kube-system   kube-flannel-ds-lsbg5                1/1     Running   0          3h16m
  kube-system   kube-flannel-ds-s7jtf                1/1     Running   0          3h17m
  kube-system   kube-proxy-jh4w5                     1/1     Running   0          3h17m
  kube-system   kube-proxy-mttvg                     1/1     Running   0          3h19m
  kube-system   kube-proxy-v4qxp                     1/1     Running   0          3h16m
  kube-system   kube-scheduler-k8s-master            1/1     Running   0          3h19m


至此我们的三节点集群搭建完成。


Fix node internal IP issue
-----------------------------


.. note::

   貌似Kubernetes ``v1.29.x`` 已经不存在这个问题了，如果你使用的是 ``v1.29.x`` 请忽略这一节。



如果node的internal IP不对， 例如我们希望的node internal IP地址是en0s8的地址。


.. code-block:: bash

  vagrant@k8s-master:~$ kubectl get nodes -o wide
  NAME          STATUS   ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
  k8s-master    Ready    control-plane   42m    v1.28.0   10.0.2.15     <none>        Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.6.24
  k8s-worker1   Ready    <none>          118s   v1.28.0   10.0.2.15     <none>        Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.6.24
  k8s-worker2   Ready    <none>          85s    v1.28.0   10.0.2.15     <none>        Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.6.24
  vagrant@k8s-master:~$

  vagrant@k8s-master:~$ ip -c a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
  2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 02:9a:67:51:1e:b6 brd ff:ff:ff:ff:ff:ff
      inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
        valid_lft 72219sec preferred_lft 72219sec
      inet6 fe80::9a:67ff:fe51:1eb6/64 scope link
        valid_lft forever preferred_lft forever
  3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:e1:e5:69 brd ff:ff:ff:ff:ff:ff
      inet 192.168.56.10/24 brd 192.168.56.255 scope global enp0s8
        valid_lft forever preferred_lft forever
      inet6 fe80::a00:27ff:fee1:e569/64 scope link
        valid_lft forever preferred_lft forever

join完成后在master节点上执行如下命令将worker1和worker2节点标记为worker role

.. code-block:: bash
kubectl label node worker1 node-role.kubernetes.io/worker=worker
kubectl label node worker2 node-role.kubernetes.io/worker=worker


修改文件 `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` ， 在最后一行末尾增加一个新的变量KUBELET_EXTRA_ARGS， 指定node ip是本机的enp0s8的地址，保存退出。

.. code-block:: bash

  # Note: This dropin only works with kubeadm and kubelet v1.11+
  [Service]
  Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
  Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
  # This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
  EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
  # This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
  # the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
  EnvironmentFile=-/etc/default/kubelet
  ExecStart=
  ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --node-ip=192.168.56.10


重启kubelet，就会发现本机master节点的internal IP显示正确了。

.. code-block:: bash

  vagrant@k8s-master:~$ sudo systemctl daemon-reload
  vagrant@k8s-master:~$ sudo systemctl restart kubelet
  vagrant@k8s-master:~$ kubectl get node -o wide
  NAME          STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
  k8s-master    Ready    control-plane   3h55m   v1.26.0   192.168.56.10   <none>        Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
  k8s-worker1   Ready    worker          3h35m   v1.26.0   10.0.2.15       <none>        Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
  k8s-worker2   Ready    worker          3h35m   v1.26.0   10.0.2.15       <none>        Ubuntu 20.04.4 LTS   5.4.0-113-generic   containerd://1.5.9
  vagrant@k8s-master:~$

通过同样的方法可以修改worker1和worker2节点的internal IP地址。

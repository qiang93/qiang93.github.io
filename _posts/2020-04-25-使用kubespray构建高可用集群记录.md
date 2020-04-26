---
layout:     post
title:      使用kubespray构建高可用集群
subtitle:   kubespray是一个快速构建k8s集群的项目工具
date:       2020-04-25
author:     huqiang
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Kubernetes
---
### 主要特点
虽然是包装了kubeadm,但结合了ansible让部署一个高可用快速增减节点的k8s集群灵活度門檻降低了

### 执行机需满足如下条件
    ansible==2.7.16
    jinja2==2.10.1
    netaddr==0.7.19
    pbr==5.2.0
    hvac==0.8.2
    jmespath==0.9.4
    ruamel.yaml==0.15.96
    
    以上依赖都在`requirements.txt`文件中
    对所有客户机免密钥登录
    
    > 建立免密钥通信后所有操作可通过ansible来管理，
---
### 安装pip3
```bash
sudo apt install python3-pip -y
sudo pip3 install --upgrade pip
```
---
### 安装release版本
```bash
wget https://codeload.github.com/kubernetes-sigs/kubespray/zip/v2.12.5
unzip v2.12.5
cd kubespray-2.12.5
```
---
### 使用方法

* Install dependencies from ``requirements.txt`` 

  ```bash
  sudo pip3 install -r requirements.txt
  ```

  

* Install dependencies from ``requirements.txt`` 

  ```
  sudo pip3 install -r requirements.txt
  ```

    > ERROR: paramiko 2.7.1 has requirement cryptography>=2.5, but you'll have cryptography 2.1.4 which is incompatible.
    >
    > ```
    > paramiko信赖cryptography>=2.5
    > 解决方法：
    > pip install cryptography==2.5
    > ```


* Copy ``inventory/sample`` as ``inventory/mycluster``   
    ```
    cp -rfp inventory/sample inventory/mycluster
    ```

    

* Update Ansible inventory file with inventory builder 
    ```
    declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
    CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
    ```
* Review and change parameters under ``inventory/mycluster/group_vars`` 
    ```
    cat inventory/mycluster/group_vars/all/all.yml
    cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
    ```

* 针对生产使用修改相应参数
    > 这里演示使用的配置为3master+2work节点组合
    默认calico且模式为ipip
    默认容器引擎为docker
    默认kubelet自动旋转到期证书从apiserver请求新证书（kubelet_rotate_certificates）
    ```
    # 主要是 group_vars/all/all.yml 和 group_vars/k8s-cluster/k8s-cluster.yml
    # 优先级 k8s-cluster.yml > all.yml > roles/xxx/defalut/main.yml
    # 所以想要覆盖 role 里面的默认配置, 优先看 k8s-cluster.yml 里面是否有同名配置, 如果有就同时修改 k8s-cluster.yml 和 all.yml, 没有就在 all.yml 里面添加
    # 或者直接使用 ansible-playbook -e @foo.yml 的方式, 因为 -e 指定的变量具有最高优先级
    ```

    * `group_vars/all/docker.yml`
        ```
        1、修改docker log size
        2、针对国内使用可以添加国内镜像地址
        ```
    * `group_vars/etcd.yml`
        ```
        ## 多长时间压缩一次历史数据，减少内存和磁盘空间的占用
        etcd_compaction_retention: 2
        ## 修改etcd内存大小，不能写成1024M，否则etcd是启动不了的
        etcd_memory_limit: "1G"
        ## 修改etcd空间配额，不能用8G，示例中是错误的
        etcd_quota_backend_bytes: "8388608"
        ```
    * `group_vars/all/all.yml`
        ```
        ## add auto update cert
        kubelet_rotate_certificates: true
        
        ## add nodes labels （便于后期在prometheus告警时区分集群）
        node_labels:
          cluster: stg-k8s-cluster
          env: staging
        ## 开启内部apiserver lb,使用nginx or haproxy
        loadbalancer_apiserver_localhost: true
        loadbalancer_apiserver_type: haproxy
        
        ## 加载内核模块，否则 ceph, gfs 等无法挂载客户端
        kubelet_load_modules: true
        ```
* `group_vars/k8s-cluster/k8s-net-calico.yml`
        
        ```
        # 根据主机网卡mtu再结合calico文档中定义的值修改，ipip模式为1480,flannet之类是自适应，除了calico需要手动调整以获得最佳网络性能
        calico_mtu: 1480
    ```
    
    * exec ansible-playbooks
        > Deploy Kubespray with Ansible Playbook - run the playbook as root
        > The option `--become` is required, as for example writing SSL keys in /etc/,
    > installing packages and interacting with various systemd daemons.
        > Without `--become` the playbook will fail to run!
    
        默认会在每个节点上进行镜像下载，如果内网无法上网或无法翻墙下载gcr.io中的镜像，可以指定在自己本机上下载再推送到所有node上
        download_run_once：仅下载一次镜像和二进制文件，并将其推送到集群节点
        download_localhost：下载在本地执行机上，需要安装docker,并且可以免密码sudo
    
        ```
        ansible-playbook -i inventory/stg-k8s-cluster/hosts.yaml cluster.yml \
            -e "{'download_run_once': true }" -e "{'download_localhost': true }"
        ```
        在集群node节点可以通过ps docker查看当前推送镜像的过程
        ```
        root     28327 28301  0 12:22 ?        00:00:00 sudo rsync --server -logDtprze.iLsfxC --log-format=%i --delay-updates . /tmp/releases/images/docker.io_calico_node_v3.7.3.tar
    root     28328 28327  0 12:22 ?        00:00:00 rsync --server -logDtprze.iLsfxC --log-format=%i --delay-updates . /tmp/releases/images/docker.io_calico_node_v3.7.3.tar
        ```
        也可以指定--tags让ansible-playbooks只执行指定步骤
        ```
        --tags download --skip-tags upload,upgrade
        ```
    
    * 注意事项
        > ansible_host：是ansible远程执行的地址，ip、access_ip是服务地址，在公有云服务器上要注意
        > 需要关闭防火墙，服务之间需要能ping通信



### 添加节点
* 建立免密钥认证，hosts.yaml添加相应的host
* --limit=node1限制Kubespray来避免干扰群集中的其他节点
    ```bash
    ansible-playbook -i inventory/mycluster/hosts.yaml scale.yml --limit=node5
    ```
### 删除节点
* 驱除节点pod
    ```bash
    kubectl drain NODE_NAME
    ```
* 传递-e node=NODE_NAME参数，以将执行限制为要删除的节点。
    ```bash
    ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -e node=node4
    ```
### 升级集群

### 使用小技巧
* 当安装某个阶段失败了，可针对性测试 
   如只针对tags为master的进行运行

  > ansible-playbook -i hosts cluster.yml -uroot -k --tags master

* scripts/get-tags.sh可以查看所有步骤 

  bash scripts/gen_tags.sh

  ```txt
  |                 Tag name | Used for
  |--------------------------|---------
  |                 annotate |
  |                     apps | K8s apps definitions
  |             bootstrap-os | Anything related to host OS configuration
  |        cinder-csi-driver |
  |                   client |
  |            cluster-roles |
  |                      cni |
  |                 dns_late |
  |                 download | Fetching container images to a delegate host
  |                     etcd | Configuring etcd cluster
  |       etcd_cluster_setup |
  |     external-provisioner |
  |                    facts | Gathering facts and misc check results
  |       ingress-controller |
  |                     init |
  |                  kubeadm |
  |                   master | Configuring K8s master node role
  |                  network | Configuring networking plugins for K8s
  |                     node | Configuring K8s minion (compute) node role
  |             node-webhook |
  |                      oci |
  |        policy-controller |
  |              post-remove |
  |             post-upgrade |
  |               preinstall | Preliminary configuration steps
  |               pre-remove |
  |              pre-upgrade |
  |                    reset |
  |               resolvconf | Configuring /etc/resolv.conf for hosts/apps
  |            rotate_tokens |
  |    upgrade_cluster_setup |
  |                     when |
  ```

* etcd检查

  > etcdctl --endpoints=https://127.0.0.1:2379 --cert-file=/etc/ssl/etcd/ssl/member-node1.pem --key-file=/etc/ssl/etcd/ssl/member-node1-key.pem cluster-health


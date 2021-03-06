# 安装pip

```shell
yum -y install epel-release
yum -y install python3
#yum -y install python3-pip
```

# clone项目

> https://github.com/kubernetes-sigs/kubespray

```shell
git clone https://github.com/kubernetes-sigs/kubespray.git
# 线上不使用master分支，建议切换至最新tag
git checkout v2.13.2

```

# 准备

```shell
cd kubespray/
pip3 install -r requirements.txt
# 如果源有问题，切换
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt

cp -rfp inventory/sample inventory/mycluster

declare -a IPS=(10.200.120.240 10.200.120.241 10.200.120.245 10.200.120.246 10.200.120.247)
# git上文档可能没更新CONFIG_FILE=inventory/mycluster/hosts.yaml 不对，需要改名
mv inventory/mycluster/inventory.ini inventory/mycluster/hosts.yml
CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# 如果报sudo: sorry, you must have a tty to run sudo
# 修改ansible.cfg 中 pipelining = False  （会降速）
# 或者受控端 /etc/sudoers文件中关闭requiretty
ansible-playbook -i inventory/mycluster/hosts.yml  --become --become-user=root cluster.yml
```



# 国内访问可能涉及的镜像修改

> https://chenyongjun.vip/articles/128

```
https://storage.googleapis.com/kubernetes-release/release/v1.18.4/bin/linux/amd64/kubeadm
https://storage.googleapis.com/kubernetes-release/release/v1.18.4/bin/linux/amd64/kubectl
https://storage.googleapis.com/kubernetes-release/release/v1.18.4/bin/linux/amd64/kubelet
```

# 我的私有云

```shell
group_vars/k8s-cluster/k8s-cluster.yml:kube_image_repo: "registry.cn-shanghai.aliyuncs.com/gcr_org"
```



# 验证安装是否成功

登录Kubernete集群的Mater集群，执行如下命令：

```
kubectl get no
```
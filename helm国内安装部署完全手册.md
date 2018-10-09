## 国内网络环境在安装helm


首先我们需要部分的翻墙功能安装我们需要的文件或者联网操作。

### 翻墙操作步骤

#### 1 helm客户端

    curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash

####  安装helm命令补全脚本


    cd ~ && helm completion bash > .helmrc && echo "source .helmrc" >> .bashrc

#### 然后安装helm服务端tiller

    helm init -i xxxx/k8s.gcr.io/tiller:v2.9.0


### 无需翻墙的操作步骤

### 2tiller服务器

#### 创建tiller的serviceaccount和clusterrolebinding

    #kubectl create serviceaccount --namespace kube-system tiller
    #kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller


#### 使用-i指定自己的镜像，因为官方的镜像因为某些原因无法拉取。

#### 为应用程序设置serviceAccount：

    kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'


### 检查是否安装成功：

    $ kubectl -n kube-system get pods|grep tiller
    tiller-deploy-f5597467b-wdwfl  1/1   Running   0  2h
    $ helm version
    Client: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}


安装出问题时使用 helm reset && helm init 重置安装。

#### 1 使用第三方的 Chart 存储库

    helm repo add 存储库名 存储库URL
    helm repo update

关于 Helm 相关命令的说明，您可以参阅 Helm 文档

鉴于国内使用镜像非常不方便Σ(っ °Д °;)っ，请添加国内的镜像进行使用kube-charts-mirror

    helm repo add stable https://burdenbear.github.io/kube-charts-mirror/


使用helm安装应用例子

安装好pv或者storageclass

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: helm-mysql
    spec:
      capacity:
    storage: 8Gi
      accessModes:
    - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      mountOptions:
    - hard
    - nfsvers=4
      nfs:
    path: /helm-mysql
    server: 172.10.1.100
    
    kubectl apply -f helm-mysql-pv.yml

#### helm的使用

参考官方文档。https://docs.helm.sh/

自定义chart

创建一个Chart的骨架：

    helm create testapi-chart


目录结构如下所示，我们主要关注目录中的这三个文件即可：Chart.yaml、values.yaml和NOTES.txt。

    testapi-chart
    ├── charts
    ├── Chart.yaml
    ├── templates
    │   ├── deployment.yaml
    │   ├── _helpers.tpl
    │   ├── NOTES.txt
    │   └── service.yaml
    └── values.yaml


打开Chart.yaml，填写应用的详细信息

打开并根据需要编辑values.yaml

对Chart进行校验

    helm lint testapi-chart

对Chart进行打包：



    helm package testapi-chart --debug
    
    Successfully packaged chart and saved it to: /var/local/k8s/helm/alpine-0.1.0.tgz
    [debug] Successfully saved /var/local/k8s/helm/alpine-0.1.0.tgz to /root/.helm/repository/local

安装本地repository中的chart


    helm search local
    helm install --name example local/mychart --set service.type=NodePort

使用Helm serve命令启动一个repo server，该server缺省使用’$HELM_HOME/repository/local’目录作为Chart存储，并在8879端口上提供服务。

    helm serve --address 0.0.0.0:8879 &


启动本地repo server后，将其加入Helm的repo列表。

    helm repo add local http://127.0.0.1:8879
    "local" has been added to your repositories


现在再查找testapi chart包，就可以找到了。

chart 搜索

    helm search


添加源
添加中国的源：



    helm repo add stable https://burdenbear.github.io/kube-charts-mirror/


常用命令行

    helm list
    helm search
    helm search mysql --versions
    helm repo list
    helm serve &

monocular 安装
选择了Træfik-ingress

官网 https://docs.traefik.io/，以下按照 user-guide 进行安装，其实就是两个命令行：

    # 创建角色和rbac绑定
    $ kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-rbac.yaml
    
    # ServiceAccount, DaemonSet(直接绑定主机端口)，service
    $ kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml


就可以看到效果了：


    curl $(minikube ip)
    404 page not found

traefik

参照GitHub的教程进行安装：

参照GitHub的教程进行安装：

    Install
    
    Configuration
    Deployment
    Development
    config


我创建了一个自己的配置文件，使用 kubernetes helm 国内镜像的配置 custom.yaml

    $ cat > custom.yaml <<EOF
    ingress:
      hosts:
      - monocular.local
      annotations:
    traefik.frontend.rule.type: PathPrefixStrip
    kubernetes.io/ingress.class: traefik 
    api:
      config:
    repos:
      - name: monocular
    url: https://kubernetes-helm.github.io/monocular
    source: https://github.com/kubernetes-helm/monocular/tree/master/charts 
    EOF


### 添加 repo

    helm repo add monocular https://kubernetes-helm.github.io/monocular

### 安装pv

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: monocular-mongodb
    spec:
      capacity:
    storage: 8Gi
      accessModes:
    - ReadWriteOnce
      persistentVolumeReclaimPolicy: Recycle
      mountOptions:
    - hard
    - nfsvers=4
      nfs:
    path: /monocular
    server: 172.10.1.100
    
    $ kubectl apply -f monocular-pv.yml    

#### 安装 monocular

    helm install monocular/monocular --name monocular --set controller.hostNetwork=true -f custom.yaml


#### 如果出现错误，重新来过的话，清空重置：

    helm del --purge monocular
    kubectl delete pv monocular-mongodb

获得 ingress 地址：

    $ kubectl get ingress monocular-monocular
    
    NAME  HOSTS   ADDRESS   PORTS AGE
    monocular-monocular   monocular.local809s



现在可以访问页面请求响应。

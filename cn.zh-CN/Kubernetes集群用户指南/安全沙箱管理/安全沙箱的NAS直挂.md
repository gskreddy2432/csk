---
keyword: [NAS直挂, 安全沙箱, 解决IO性能]
---

# 安全沙箱的NAS直挂

您可以通过安全沙箱的NAS直挂功能，有效地解决IO性能问题。本文介绍安全沙箱NAS直挂的实现原理，并通过示例说明如何使用安全沙箱的NAS直挂功能。

-   [创建安全沙箱托管版集群](/cn.zh-CN/Kubernetes集群用户指南/安全沙箱管理/创建安全沙箱托管版集群.md)
-   [通过kubectl连接Kubernetes集群](/cn.zh-CN/Kubernetes集群用户指南/集群管理/连接集群/通过kubectl连接Kubernetes集群.md)

virtio-fs是一个共享的文件系统。容器服务Kubernetes版（ACK）安全沙箱使用virtio-fs可以把Volume、Secret、ConfigMap等共享到虚拟机GuestOS内，从而可以用原生的方式通过Volume挂载NAS。但这种方式下，NAS是挂载到主机上的，在容器内要经过virtio-fs读写主机上的NAS，由此会带来一些性能损耗。

安全容器提供了安全沙箱NAS直挂的功能。该功能会先卸载主机上的NAS挂载点，然后在GuestOS内挂载NAS，最后把NAS bind mount到容器内，从而能够在容器内直接读写NAS，达到接近原生的性能。

![直挂方案](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/9856076061/p185928.png)

## 实现原理

![挂载原理](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/0956076061/p95364.png)

实现安全沙箱的NAS直挂功能的流程说明如下。

|步骤序号|说明|
|----|--|
|①|Kubelet请求CSI-Plugin挂载NAS卷。|
|②|CSI-Plugin在主机挂载NAS。|
|③|Kubelet请求Kangaroo-Runtime创建容器。|
|④|Kangaroo-Runtime解析NAS挂载信息并传入GuestOS，同时卸载主机上的NAS。|
|⑤|Kangaroo-Runtime请求Agent创建容器。|
|⑥|Agent把NAS挂载到GuestOS内。|
|⑦|Agent把GuestOS上的NAS bind mount到容器内。|

## 示例

以下通过创建NAS实例及使用YAML模板创建资源对象举例说明如何在安全沙箱容器中挂载NAS卷。

1.  创建一个NAS实例。具体操作，请参见[创建通用型NAS文件系统]()。

    **说明：** NAS实例的VPC需要和集群的VPC一致。

    获取到NAS实例挂载点地址，如下图**挂载命令**中的file-system-id.region.nas.aliyuncs.com即为挂载点地址。

    ![NAS](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/0606659951/p95499.png)

    **说明：** file-system-id.region.nas.aliyuncs.com：表示挂载点地址。您可以在[NAS控制台](https://nas.console.aliyun.com/)中，单击模板文件系统名称，在**挂载使用**页面获取挂载点地址。

2.  执行以下命令，使用模板创建资源对象。

    ```
    cat <<EOF | kubectl create -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nas-pvc-csi
      namespace: default
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 5Gi
      selector:
        matchLabels:
          alicloud-pvname: nas-pv-csi
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      labels:
        alicloud-pvname: nas-pv-csi
      name: nas-pv-csi
    spec:
      accessModes:
        - ReadWriteMany
      capacity:
        storage: 5Gi
      csi:
        driver: nasplugin.csi.alibabacloud.com
        volumeAttributes:
          options: noresvport,nolock
          path: /csi
          server: ${nas-server-address}
          vers: "3"
        volumeHandle: nas-pv-csi
      persistentVolumeReclaimPolicy: Retain
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: deploy-nas-csi
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: busybox
      template:
        metadata:
          labels:
            app: busybox
          annotations:
            storage.alibabacloud.com/enable_nas_passthrough: "true"
        spec:
          runtimeClassName: runv
          containers:
            - name: busybox
              image: registry.cn-hangzhou.aliyuncs.com/acs/busybox:v1.29.2
              command: 
              - tail
              - -f
              - /dev/null
              volumeMounts:
                - name: nas-pvc
                  mountPath: "/data"
          restartPolicy: Always
          volumes:
            - name: nas-pvc
              persistentVolumeClaim:
                claimName: nas-pvc-csi
    EOF
    ```

    **说明：**

    -   您需要把模板中的`${nas-server-address}`替换成NAS实例挂载点地址。

        ```
        server: ${nas-server-address}
        ```

    -   默认Pod不启用NAS直挂功能，您需要在模板中通过Annotation开启NAS直通功能。

        ```
        annotations:
                storage.alibabacloud.com/enable_nas_passthrough: "true"
        ```

3.  执行以下命令，进入Pod内验证挂载点类型。

    ```
    kubectl get pods
    ```

    ```
    kubectl exec -it ${podid} sh
    ```

    ```
    mount | grep /data | grep nfs
    ```

    如果执行命令后有内容，表明NAS直挂功能成功。



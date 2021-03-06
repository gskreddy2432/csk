---
keyword: [Knative系统组件, Serving, 自动扩缩]
---

# 升级Serving组件

Serving组件用于管理Serverless工作负载，可以和事件结合并且提供基于请求驱动的自动扩缩的能力，而且在没有服务需要处理的时可以缩容到零个实例。本文介绍如何升级Serving组件。

-   您已部署Serving组件，请参见[部署组件](/intl.zh-CN/Kubernetes集群用户指南/Knative管理/Knative组件管理/部署组件.md)。
-   Kubernetes版本大于1.15.0。
-   仅支持Knative 0.11.0版本升级。

**说明：** 建议您在业务运行低谷时进行升级操作。

您可以升级Serving组件至0.14.0。

Knative Serving 0.14.0版本新增以下主要特性：

-   默认最小保留Knative服务的修订版本数为20，默认保留时间为48小时。
-   支持高可靠组件，包括controller、hpaautoscaler等。

## 操作步骤

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。

2.  在集群列表页面中，单击目标集群名称或者目标集群右侧**操作**列下的**详情**。

3.  在集群管理页左侧导航栏中，单击**Knative** \> **组件管理**。

4.  选择**Serving**组件，然后单击**升级**。

    ![serving组件](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3895659951/p127321.png)

    升级完成之后，显示执行结果信息。

    ![结果](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3895659951/p127322.png)



# 手把手教你玩转 一站式运维平台(CODO) - 8.使用云原生管理平台管理多地集群



## 集群接入

填入kubeconfig 即可完成接入

- 需要注意服务器和 kubeconfig 的 api 地址网络联通性
- 使用最高权限的 kubeconfig 导入

![image-20250802160413954](../images/image-20250802160413954.png)





## 凭证导出

权限配置好之后, 可以导出访问凭证, 导出的凭证用户流量会先进入到 灵息 平台, 由 灵息 平台代理到目标集群

从而可以无感实现

- 操作审计
- 权限控制

![image-20250802174831908](../images/image-20250802174831908.png)



## 权限配置

### 配置角色

一般来说不需要特别配置角色, 预置的管理员以及只读足够了

![image-20250802175113260](../images/image-20250802175113260.png)

### 用户组授权

**用户组授权需要需要先在 [后台管理 配置好角色](../3-admin/role-auth.md)**

![image-20250802174447211](../images/image-20250802174447211.png)



## 操作审计

![image-20250802174743181](../images/image-20250802174743181.png)


## 获取pod内存、CPU指标
若无法获取请部署metrics-scraper组件
yaml文件示例，亲测可用

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: metrics-scraper
  name: metrics-scraper
  namespace: kube-system
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: metrics-scraper
  name: metrics-scraper
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: metrics-scraper
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.4
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}

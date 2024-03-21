# 配置

## 1. OAuth2-Proxy 通过 OpenID Connect 方式接入 KubeSphere 4.x 认证

### 1.1 确认环境基础信息

以当前自试环境为例，KubeSphere Console 地址为 [http://172.31.73.114:30880](http://172.31.73.114:30880), OAuth2-Proxy 使用 `NodePort` 暴露，地址为 [http://172.31.73.114:32080](http://172.31.73.114:32080);因此:

```yaml
    # service redirect address
    redirect-url: http://172.31.73.114:32080/oauth2/callback
    # issuer address
    oidc-issuer-url: http://172.31.73.114:30880
```

### 1.2 更新 KubeSphere 配置

确认实际对外访问地址与 `configmap` kubesphere-config 中配置一致

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubesphere-config
  namespace: kubesphere-system
data:
  kubesphere.yaml: |
    authentication:
      issuer:
        host: "http://172.31.73.114:30880"     # 确认 issuer 地址
```

如不一致，可通过 `helm upgrade --set portal.hostname=xx.xx.xx.xx` 更新

```sh
helm upgrade --install -n kubesphere-system --create-namespace ks-core  https://charts.kubesphere.io/test/ks-core-0.6.13.tgz --debug --wait  --set global.tag=$(echo "nightly-$(TZ=Asia/Shanghai date +"%Y%m%d")") --set portal.hostname=172.31.73.114
```

增加 `secret` oauthclient-oauth2-proxy, 确认 redirectURIs 地址

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: oauthclient-oauth2-proxy
  namespace: kubesphere-system
  labels:
    config.kubesphere.io/type: oauthclient
type: config.kubesphere.io/oauthclient
stringData:
  configuration.yaml: |
    name: oauth2-proxy
    secret: oauth2-proxy
    grantMethod: auto
    trusted: true
    scopeRestrictions:
      - 'openid'
      - 'email'
      - 'profile'
    redirectURIs:
      - http://172.31.73.114:32080/oauth2/callback
```

### 1.3 更新 OAuth2-Proxy 扩展组件配置

更新 OAuth2-Proxy 扩展组件配置，确认 `redirect-url` `oidc-issuer-url` 地址及服务暴露方式

```yaml
oauth2-proxy:
  extraArgs: 
    # service redirect address
    redirect-url: http://172.31.73.114:32080/oauth2/callback
    # issuer address
    oidc-issuer-url: http://172.31.73.114:30880

  service:
    type: NodePort
    nodePort: 32080
```

## 2. openresty 代理未认证的服务

以接入默认的 AlertManager 服务为例，设置 subPath 为 `/alertmanager/`，endpoint 为 `http://alertmanager-main.monitoring.svc:9093/`， 即我们可以通过 [http://172.31.73.114:32080/alertmanager/](http://172.31.73.114:32080/alertmanager/) 经过鉴权后访问 AlertManager 服务。

```yaml
openresty:
  configs:
  - name: alertmanager
    description: KubeSphere 监控栈内部 Alertmanager 端点
    subPath: /alertmanager/
    endpoint: http://alertmanager-main.monitoring.svc:9093/
    additional:
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
```

同时通过 OAuth2-Proxy 认证登录后，我们会在首页看到 Alertmanager 服务的入口，点击即可访问。

| Service      | Description                             | Link                                                     |
|:------------ |:--------------------------------------- |:-------------------------------------------------------- |
| alertmanager | KubeSphere 监控栈内部 Alertmanager 端点 | [/alertmanager](http://172.31.73.114:32080/alertmanager) |

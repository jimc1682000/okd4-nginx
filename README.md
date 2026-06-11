# okd4-nginx

> Non-privileged nginx for OKD 4 / OpenShift
> 配合 [OKD4 Home Lab](https://jimc1682000.github.io/okd4_files/) 使用的示範 image

---

## 為什麼需要特殊處理

OKD/OpenShift 以任意 UID 執行 pod，預設 `nginx:stable` image 有兩個問題：

| 問題 | 原因 |
|---|---|
| 監聽 port 80（privileged port） | 非 root 無法 bind < 1024 |
| cache / pid / log 目錄不可寫 | 屬 root，group 無 write bit |

## Dockerfile

```dockerfile
FROM nginx:stable

# 開放 group 寫入，讓任意 UID 可寫 cache / pid / log
RUN chmod g+rwx \
      /var/cache/nginx \
      /var/run \
      /var/log/nginx

# port 80 → 8081（非 privileged）
RUN sed -i.bak \
  's/listen\(.*\)80;/listen 8081;/' \
  /etc/nginx/conf.d/default.conf

# 某些 nginx 版本需停用 user 指令（已跑為任意 UID 時 user 指令會報錯）
# RUN sed -i.bak 's/^user/#user/' /etc/nginx/nginx.conf

EXPOSE 8081
```

---

## 部署到 OKD4

### 方法一：`oc new-app`（快速測試）

```bash
oc new-app jimc1682000/okd4-nginx --name okd4-nginx
oc expose svc okd4-nginx --port=8081
oc get route okd4-nginx
```

### 方法二：YAML manifest（GitOps）

```bash
oc apply -f k8s.yaml
oc get route okd4-nginx
```

---

## 操作 pod

```bash
# 進入 shell
oc rsh $(oc get pod -l app=okd4-nginx -o name)

# 修改首頁（pod 內執行）
sed -i.bak 's/nginx/OKD4 Lab/' /usr/share/nginx/html/index.html
```

---

## 本機測試

```bash
docker build -t okd4-nginx:local .

# 以任意 UID 模擬 OpenShift 行為
docker run --user 123456 -p 8081:8081 okd4-nginx:local
curl http://localhost:8081
```

---

## 相關連結

- [OKD4 Home Lab 作品集](https://jimc1682000.github.io/okd4_files/)
- [okd4_files（叢集 config files）](https://github.com/jimc1682000/okd4_files)
- [Nginx on OpenShift 參考](https://torstenwalter.de/openshift/nginx/2017/08/04/nginx-on-openshift.html)

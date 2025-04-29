

kubectl apply -f etcd.yaml
kubectl apply -f apisix.yaml
kubectl apply -f apisix-dashboard.yaml

# kubectl get svc/apisix-dashboard -n default
NAME               TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
apisix-dashboard   NodePort   10.43.95.74   <none>        80:32394/TCP   12s


```config
    location / {
        proxy_pass  http://10.10.61.114:32394;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
```


## etcd backup & restore
```
docker network ls
docker ps

export ENDPOINT=docker-apisix-etcd-1:2379
docker run --rm  --network docker-apisix_apisix -v $(pwd)/snapshot:/tmp  docker-hub.zxzhao.com/anjia0532/etcd-development.etcd:v3.5.3 /bin/sh -c "/usr/local/bin/etcdctl --endpoints=${ENDPOINT} snapshot save /tmp/$(TZ=Asia/Shanghai date +%Y%m%d%H%M%S).db "
```


```
docker run --rm  --network docker-apisix_apisix -v $(pwd)/snapshot:/tmp  docker-hub.zxzhao.com/anjia0532/etcd-development.etcd:v3.5.3 /bin/sh -c "/usr/local/bin/etcdctl snapshot status /tmp/20250429150530.db"
```

docker network ls
docker ps
docker run --rm  --network docker-apisix_apisix -v $(pwd)/:/tmp  docker-hub.zxzhao.com/anjia0532/etcd-development.etcd:v3.5.3 /bin/sh -c "/usr/local/bin/etcdctl --endpoints=${ENDPOINT} snapshot restore /tmp/snapshot.db"

### restore k8s

```

kubectl  cp /tmp/20250429150530.db -n default  apisix-etcd-0:/tmp/

# apisix-etcd-0
kubectl exec -it -n default apisix-etcd-0 -- bash
cd /tmp 
rm -rf /bitnami/etcd/data/*
ETCDCTL_API=3 etcdctl snapshot restore 20250429150530.db \
--name apisix-etcd-0 \
--data-dir="/bitnami/etcd/data" \
--initial-cluster apisix-etcd-0=http://apisix-etcd-0.apisix-etcd-headless.default.svc.cluster.local:2380,apisix-etcd-1=http://apisix-etcd-1.apisix-etcd-headless.default.svc.cluster.local:2380,apisix-etcd-2=http://apisix-etcd-2.apisix-etcd-headless.default.svc.cluster.local:2380 \
--initial-advertise-peer-urls http://apisix-etcd-0.apisix-etcd-headless.default.svc.cluster.local:2380
```

```
kubectl run --rm -it etcd-restore \
  --image=docker.io/bitnami/etcd:3.5.1-debian-10-r31 \
  --namespace=default \
  --overrides='
{
  "apiVersion": "v1",
  "spec": {
    "securityContext": { "runAsUser": 0 },
    "containers": [{
      "name": "restorer",
      "image": "docker.io/bitnami/etcd:3.5.1-debian-10-r31",
      "command": ["sh", "-c", "echo Cleaning... && rm -rf /bitnami/etcd/* /bitnami/etcd/.* 2>/dev/null || true && echo Waiting for snapshot... && while [ ! -f /backup/snapshot.db ]; do sleep 1; done && echo Restoring... && etcdctl snapshot restore /backup/snapshot.db --name=apisix-etcd-0 --data-dir=/bitnami/etcd --initial-cluster apisix-etcd-0=http://localhost:2380 --initial-advertise-peer-urls http://localhost:2380 && echo Done && chown -R 1001:1001 /bitnami/etcd && ls -la /bitnami/etcd/"],
      "env": [
        { "name": "ETCDCTL_API", "value": "3" }
      ],
      "volumeMounts": [
        {
          "mountPath": "/bitnami/etcd",
          "name": "data"
        },
        {
          "mountPath": "/backup",
          "name": "backup"
        }
      ]
    }],
    "volumes": [
      {
        "name": "data",
        "persistentVolumeClaim": {
          "claimName": "data-apisix-etcd-0"
        }
      },
      {
        "name": "backup",
        "emptyDir": {}
      }
    ],
    "restartPolicy": "Never"
  }
}'
```


```
kubectl run --rm -it etcd-restore \
  --image=docker.io/bitnami/etcd:3.5.1-debian-10-r31 \
  --namespace=default \
  --overrides='
{
  "apiVersion": "v1",
  "spec": {
    "securityContext": { "runAsUser": 0 },
    "containers": [{
      "name": "restorer",
      "image": "docker.io/bitnami/etcd:3.5.1-debian-10-r31",
      "command": ["sh", "-c", "echo Cleaning... && rm -rf /bitnami/etcd/data/* /bitnami/etcd/data/.* 2>/dev/null || true && echo Waiting for snapshot... && while [ ! -f /backup/snapshot.db ]; do sleep 1; done && echo Restoring... && etcdctl snapshot restore /backup/snapshot.db --name=apisix-etcd-0 --data-dir=/bitnami/etcd/data --initial-cluster apisix-etcd-0=http://localhost:2380 --initial-advertise-peer-urls http://localhost:2380 && chown -R 1001:1001 /bitnami/etcd/data && echo Done && ls -la /bitnami/etcd/data"],
      "env": [
        { "name": "ETCDCTL_API", "value": "3" }
      ],
      "volumeMounts": [
        {
          "mountPath": "/bitnami/etcd/data",
          "name": "data"
        },
        {
          "mountPath": "/backup",
          "name": "backup"
        }
      ]
    }],
    "volumes": [
      {
        "name": "data",
        "persistentVolumeClaim": {
          "claimName": "data-apisix-etcd-0"
        }
      },
      {
        "name": "backup",
        "emptyDir": {}
      }
    ],
    "restartPolicy": "Never"
  }
}'
```



kubectl cp snapshot.db default/$(kubectl get pod -l run=etcd-restore -o jsonpath="{.items[0].metadata.name}"):/backup/snapshot.db



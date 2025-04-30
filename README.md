## Quick Start

```
kubectl apply -f etcd.yaml
kubectl apply -f apisix.yaml
kubectl apply -f apisix-dashboard.yaml
```

## Etcd backup & restore

### Docker

1. Check network

```
docker network ls
docker ps
```

2. Backup

```
docker run --rm \
  --network etcd_default \
  -v $(pwd)/backup:/backup \
  bitnami/etcd:latest \
  etcdctl \
    --endpoints=http://etcd:2379 \
    --insecure-skip-tls-verify=true \
    snapshot save /backup/snapshot.db
```

3. Validtion snapshot

```
docker run --rm \
  -v $(pwd)/backup:/backup \
  bitnami/etcd:latest \
  etcdctl snapshot status /backup/snapshot.db --write-out=table
```

4. Restore

```
sudo chown -R 1001:1001 ./backup
sudo rm -rf ./data && mkdir ./data && sudo chown -R 1001:1001 ./data

docker run --rm \
  -v $(pwd)/backup:/backup \
  -v $(pwd)/data:/bitnami/etcd/data \
  bitnami/etcd:latest \
  etcdctl snapshot restore /backup/snapshot.db \
    --data-dir=/bitnami/etcd/data
```

### K8s

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
      "command": ["sh", "-c", "echo Cleaning... && rm -rf /bitnami/etcd/data && mkdir -p /bitnami/etcd/data && echo Waiting for snapshot... && while [ ! -f /tmp/snapshot.db ]; do sleep 1; done && echo Restoring... && etcdctl snapshot restore /tmp/snapshot.db --data-dir=/bitnami/etcd/data && echo Done && chown -R 1001:1001 /bitnami/etcd/data && ls -la /bitnami/etcd/data"],
      "env": [
        { "name": "ETCDCTL_API", "value": "3" }
      ],
      "volumeMounts": [
        {
          "mountPath": "/bitnami/etcd/",
          "name": "data"
        }
      ]
    }],
    "volumes": [
      {
        "name": "data",
        "persistentVolumeClaim": {
          "claimName": "data-apisix-etcd-0"
        }
      }
    ],
    "restartPolicy": "Never"
  }
}'


kubectl cp snapshot.db default/$(kubectl get pod -l run=etcd-restore -o jsonpath="{.items[0].metadata.name}"):/tmp/snapshot.db
```


test

```bash
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





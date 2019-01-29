# k8s-EFK 国内安装，持久化存储
官网EFK基础配置
>> https://github.com/kubernetes/kubernetes.git

由于github上得镜像Dockerfile和实际有出入，手动编译无法执行，还是通过官方的镜像进行操作，但国内无法下载，所以通过hub.docker.com上同步的Google镜像仓库进行下载：
涉及镜像：
>> es: mirrorgooglecontainers/elasticsearch:v6.3.0

>> fluentd-es: mirrorgooglecontainers/fluentd-elasticsearch:v2.4.0

### 下面是官方yaml改动说明(namesapces不做过多说明，可自定义)
```
# es-statefulset.yaml 改动
        volumeMounts:
        - name: elasticsearch-logging
          mountPath: /data # 官方镜像默认数据存位置
	# 部分内置变量
        env:
        - name: "NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: cluster.name
          value: k8s-logs
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
	# 官方yaml文件未做持久化存储，禁用
#      volumes:
#      - name: elasticsearch-logging
#        emptyDir: {}
      # Elasticsearch requires vm.max_map_count to be at least 262144.
      # If your OS already sets up this number to a higher value, feel free
      # to remove this init container.
      initContainers:
      - image: alpine:3.6
        command: ["/sbin/sysctl", "-w", "vm.max_map_count=262144"]
        name: elasticsearch-logging-init
        securityContext:
          privileged: true
#StatefulSet 通过volumeClaimTemplates 挂载存储，可以是hostpath、nfs等
  volumeClaimTemplates:
  - metadata:
      name: elasticsearch-logging
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: efk # 重要内容，创建pv时的storageClassName字段，需一一匹配，切不需要创建PVC，不需要创建PVC，不需要创建PVC，重要的事情说三遍
      resources:
        requests:
          storage: 10Gi

```
#### ES 持久化存储配置(下面是hostpath测试)
这边是创建了3个ES节点，所以对应挂载3个PV
PV重点：storageClassName 需要与statefulset一致
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-logging-0
  labels:
    name: efk
spec:
  storageClassName: efk
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/es/0
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-logging-1
  labels:
    name: efk
spec:
  storageClassName: efk
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/es/1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-logging-2
  labels:
    name: efk
spec:
  storageClassName: efk
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/es/2
```
#### fluentd-es配置无需改动。注意镜像即可

#### kibana 配置，特别说明下，有点坑
```
#kibana-deployment.yaml
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch-logging:9200
#          - name: SERVER_BASEPATH  # 此变量是个小坑，非必要参数，这边最后通过Ingress访问Kibana服务即可
#            value: /api/v1/namespaces/kube-system/services/kibana-logging/proxy
``` 

# Create Cluster

```bash
eksctl create cluster \
  --name elk-cluster \
  --region ap-south-1 \
  --version 1.33 \
  --nodegroup-name workers \
  --node-type t3.large \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 5
```

```bash
kubectl get nodes
```

```bash
eksctl utils associate-iam-oidc-provider --region=ap-south-1 --cluster=elk-cluster --approve
```


# Install EBS driver
```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster elk-cluster \
  --region ap-south-1 \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

## Install addon
```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster elk-cluster \
  --region ap-south-1 \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```
```bash
kubectl get pods -n kube-system | grep ebs
```

# Create Namespace
```bash
kubectl create namespace logging
```

# Create Storage Class

```bash
kubectl apply -f storage-class/storage-class.yaml -n logging
```


# Add Elastic Helm Repo
```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```
# Install Elasticsearch

create file -- elasticsearch-values.yaml
```bash
replicas: 3

volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi

resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
```

```bash
helm install elasticsearch elastic/elasticsearch \
  -n logging \
  -f elasticsearch-values.yaml --version 7.17.3
```

# Get Elasticsearch Password

```bash
kubectl get secret -n logging
```

## Get password:
```bash
kubectl get secret elasticsearch-master-credentials \
  -n logging \
  -o jsonpath='{.data.password}' | base64 -d
```
user name --- elastic

## Install Kibana
create file kibana-values.yaml
``` bash
elasticsearchHosts: "https://elasticsearch-master:9200"

extraEnvs:
  - name: ELASTICSEARCH_USERNAME
    value: elastic

  - name: ELASTICSEARCH_PASSWORD
    value: "<PASSWORD>"
```

** REPLACES ELASTIC SEARCH PASSWORD

```bash
helm install kibana elastic/kibana \
  -n logging \
  -f kibana/kibana-values.yaml --version 7.17.3
  ```


## Port forward
```bash
kubectl port-forward svc/kibana-kibana 5601:5601 -n logging
```

## Open
```bash
http://localhost:5601
```


# Install Log stash

Create logstash-values.yaml
```yaml
logstashPipeline:
  logstash.conf: |
    input {
      beats {
        port => 5044
      }
    }

    output {
      elasticsearch {
        hosts => ["https://elasticsearch-master:9200"]
        user => "elastic"
        password => "<PASSWORD>"
        ssl => true
        ssl_certificate_verification => false
        index => "k8s-logs-%{+YYYY.MM.dd}"
      }
    }
```

## Install 
```bash
helm install logstash elastic/logstash \
  -n logging \
  -f logstash/logstash-values.yaml --version 7.17.3
```

## Verify
```bash
kubectl get pods -n logging
```

# Install Filebeat

Create filebeat-values.yaml
```bash
filebeatConfig:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log

    output.logstash:
      hosts: ["logstash-logstash:5044"]
```

## Install
```bash
helm install filebeat elastic/filebeat \
  -n logging \
  -f file-beat/filebeat-values.yaml --version 7.17.3
```

## Verify
```bash
kubectl get ds -n logging
```

# Better Test Application
Create logger.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
      - name: logger
        image: busybox
        command:
        - sh
        - -c
        - |
          while true
          do
            echo "User logged in $(date)"
            sleep 5
          done
```

```bash
kubectl apply -f logger.yaml
```

```bash
kubectl logs deployment/log-generator
```

# Verify Log Flow

## Check Filebeat
```bash
kubectl logs -n logging daemonset/filebeat
```

## Check Logstash
```bash
kubectl logs -n logging deployment/logstash-logstash
```

## Check Elasticsearch:
```bash
kubectl port-forward svc/elasticsearch-master 9200:9200 -n logging
```

##  Query
```bash
curl -k -u elastic:<PASSWORD> \
https://localhost:9200/_cat/indices?v
```


# Kibana setup
Login

Navigate -- 
Stack Management
    → Data Views
    → Create Data View

Enter:
    k8s-logs-*

Select:
    @timestamp

Save

Open:
    Analytics
        → Discover

You should now see:
    User logged in ...
    User logged in ...
    User logged in ...
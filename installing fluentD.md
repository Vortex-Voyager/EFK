# 📦 Fluentd Log Shipping to Elasticsearch on Kubernetes

This guide walks you through configuring Fluentd as a log shipper in your Kubernetes cluster, forwarding logs from your app pod to Elasticsearch.

---

## 1. Check Namespace

Ensure your namespace is set to `elastic-stack`:

```bash
kubectl config view --minify --output 'jsonpath={..namespace}'
```

If not, switch to it.

---

## 2. Lab Setup Overview

- Two pods are deployed: `app` (generates logs at `/log`) and `elasticsearch-0` (pre-configured from previous lab).
- No logs are being forwarded to Elasticsearch yet.
- To check logs in Elasticsearch, visit its UI and append:
  ```
  /_search?q=*:*&pretty
  ```

---

## 3. Fluentd Configuration Steps

### Step 1: Create Fluentd Directory

Create a directory for Fluentd configs:

```bash
mkdir -p /root/fluentd
```

### Step 2: ServiceAccount for Fluentd

Create `fluentd-sa.yaml` in `/root/fluentd`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: elastic-stack
  labels:
    app: fluentd
```

Apply:

```bash
kubectl apply -f fluentd-sa.yaml
```

### Step 3: ClusterRole for Fluentd

Create `fluentd-clusterrole.yaml` in `/root/fluentd`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
```

Apply:

```bash
kubectl apply -f fluentd-clusterrole.yaml
```

### Step 4: ClusterRoleBinding

Create `fluentd-clusterrolebinding.yaml` in `/root/fluentd`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: elastic-stack
```

Apply:

```bash
kubectl apply -f fluentd-clusterrolebinding.yaml
```

### Step 5: Fluentd Config File

Create `/root/fluentd/etc/fluent.conf` (create directories as needed):

```conf
<label @FLUENT_LOG>
  <match fluent.**>
    @type null
    @id ignore_fluent_logs
  </match>
</label>
<source>
  @type tail
  @id in_tail_container_logs
  path "/var/log/containers/*.log"
  pos_file "/var/log/fluentd-containers.log.pos"
  tag "kubernetes.*"
  exclude_path /var/log/containers/fluent*
  read_from_head true
  <parse>
    @type "/^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/"
    time_format "%Y-%m-%dT%H:%M:%S.%NZ"
    unmatched_lines
    expression ^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$
    ignorecase false
    multiline false
  </parse>
</source>
<match **>
  @type elasticsearch
  @id out_es
  @log_level "info"
  include_tag_key true
  host "elasticsearch.elastic-stack.svc.cluster.local"
  port 9200
  path ""
  scheme http
  ssl_verify false
  ssl_version TLSv1_2
  user
  password xxxxxx
  reload_connections false
  reconnect_on_error true
  reload_on_failure true
  log_es_400_reason false
  logstash_prefix "fluentd"
  logstash_dateformat "%Y.%m.%d"
  logstash_format true
  index_name "logstash"
  target_index_key
  type_name "fluentd"
  include_timestamp false
  template_name
  template_file
  template_overwrite false
  sniffer_class_name "Fluent::Plugin::ElasticsearchSimpleSniffer"
  request_timeout 5s
  application_name default
  suppress_type_name true
  enable_ilm false
  ilm_policy_id logstash-policy
  ilm_policy {}
  ilm_policy_overwrite false
  <buffer>
    flush_thread_count 8
    flush_interval 5s
    chunk_limit_size 2M
    queue_limit_length 32
    retry_max_interval 30
    retry_forever true
  </buffer>
</match>
```

Refer to the [official Fluentd documentation](https://docs.fluentd.org/configuration/config-file) for more details.

---

### Step 6: Fluentd DaemonSet

Create `fluentd.yaml` in `/root/fluentd`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: elastic-stack
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.14.1-debian-elasticsearch7-1.0
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.elastic-stack.svc.cluster.local"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
          - name: FLUENT_ELASTICSEARCH_SSL_VERIFY
            value: "false"
          - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
            value: /^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/
          - name:  FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX
            value: "fluentd"
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: configpath
          mountPath: /fluentd/etc
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: configpath
        hostPath:
          path: /root/fluentd/etc
```

Apply:

```bash
kubectl apply -f fluentd.yaml
```

---

## 4. Verifying Log Forwarding

To check if Fluentd is forwarding logs to Elasticsearch:

1. Go to the Elasticsearch UI and append:
   ```
   /_search?q=*:*&pretty
   ```
2. You should see logs with `pod_name` as `app`.
3. In your terminal, run:
   ```bash
   kubectl logs app
   ```
   Match these log lines with those in Elasticsearch.

> **Note:** Elasticsearch may limit the number of logs returned in a single request. Use the `size` parameter or scroll API for large result sets:
>
> `/search?q=*:*&size=10000&pretty`

---
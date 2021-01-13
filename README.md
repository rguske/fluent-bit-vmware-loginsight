# Leveraging Fluent Bit on Tanzu Kubernetes Grid to send VEBA logs to vRealize Log Insight

Corresponding Blog Post: https://rguske.github.io/post/leveraging-fluent-bit-on-tkg-to-send-veba-logs-to-vrli/ 

## <i class='fab fa-github fa-fw'></i> Repository

1. Clone the repository locally and change into the directory `fluent-bit-vmware-loginsight`: `git clone https://github.com/rguske/fluent-bit-vmware-loginsight.git && cd fluent-bit-vmware-loginsight`

### Adjustments

2. Make your adjustments to the following files to meet your requirements and environment specifications:

- `2-tkg-fluent-bit-configmap-cri.yaml`
- `3-tkg-fluent-bit-ds.yaml`

### Fluent Bit Configmap

The first adjustment you have to make is the replacement of your TKG clustername as well as the hostname of your vRealize Log Insight appliance.

File `2-tkg-fluent-bit-configmap-cri.yaml`:

- Replace the data for your TKG cluster in the `filter-record.conf: |` section
  - Note: for non TKG cluster, just delete the record

`filter-record.conf: |`

```code
[FILTER]
    Name                record_modifier
    Match               *
    Record tkg_instance tkg-veba
    Record tkg_cluster  tkg-veba
```

Enter your Log Insight server hostname in the `output-syslog.conf` section:

`output-syslog.conf: |`

```code
[OUTPUT]
    Name                 syslog
    Match                *
    Host                 vrli.jarvis.lab
    Port                 514
    Mode                 tcp
    Syslog_Format        rfc5424
    Syslog_Hostname_key  tkg_cluster
    Syslog_Appname_key   pod_name
    Syslog_Procid_key    container_name
    Syslog_Message_key   message
    syslog_msgid_key     msgid
    Syslog_SD_key        k8s
    Syslog_SD_key        labels
    Syslog_SD_key        annotations
    Syslog_SD_key        tkg
```

### Fluent Bit DaemonSet

Basically, it's not necessary to change the image for Fluent Bit but you could replace it with the one of your choice. I used the official Fluent Bit image which is at the time of writing this post version 1.6.9 -  `fluent/fluent-bit:1.6.9`. You could also use e.g. the VMware Fluent Bit Image in version 1.5.3: `registry.tkg.vmware.run/fluent-bit:v1.5.3_vmware.1`. I've validated the functionality of both images during my tests.

More images:

- Official: https://hub.docker.com/r/fluent/fluent-bit
- Bitnami: https://hub.docker.com/r/bitnami/fluent-bit
- VMware Registry: registry.tkg.vmware.run/fluent-bit:v1.5.3_vmware.1

File `3-tkg-fluent-bit-ds.yaml`:

- Make your edits at the `spec` section for the container:

```yaml
[...]

spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:1.6.9

[...]
```

## Deploy Fluent Bit on Kubernetes

3. Start with applying the `1-tkg-fluent-bit-preps.yaml` file to have the prerequisites (`Namespace`, `ServiceAccount`, etc.) done:

```shell
$ kubectl create -f 1-tkg-fluent-bit-preps.yaml

namespace/fluent-bit created
serviceaccount/fluent-bit created
clusterrole.rbac.authorization.k8s.io/fluent-bit-read created
clusterrolebinding.rbac.authorization.k8s.io/fluent-bit-read created
```

4. Next is to apply the configmap - `2-tkg-fluent-bit-configmap-cri.yaml` - for Fluent Bit:

```shell
kubectl create -f 2-tkg-fluent-bit-configmap-cri.yaml
configmap/fluent-bit-config created
```

1. As mentioned in the Prerequisites section before, Fluent Bit will be deployed as a DaemonSet by using the `3-tkg-fluent-bit-ds.yaml` file:

```shell
kubectl create -f 3-tkg-fluent-bit-ds.yaml && kubectl wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n fluent-bit
daemonset.apps/fluent-bit created
pod/fluent-bit-dd4ql condition met

kubectl get daemonsets.apps -n fluent-bit

NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluent-bit   2         2         2       2            2           <none>          95s

kubectl get pods -n fluent-bit
NAME               READY   STATUS    RESTARTS   AGE
fluent-bit-9m7qq   1/1     Running   0          108s
fluent-bit-dd4ql   1/1     Running   0          108s
```

More information about the successful deployment and as a first indication that everything is working fine can be grabbed from the logs of one of the pods.

```code
kubectl logs -n fluent-bit fluent-bit-dd4ql

Fluent Bit v1.6.9
* Copyright (C) 2019-2020 The Fluent Bit Authors
* Copyright (C) 2015-2018 Treasure Data
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

[2021/01/12 14:05:10] [ info] [engine] started (pid=1)
[2021/01/12 14:05:10] [ info] [storage] version=1.0.6, initializing...
[2021/01/12 14:05:10] [ info] [storage] in-memory
[2021/01/12 14:05:10] [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
[2021/01/12 14:05:11] [ info] [input:systemd:systemd.1] seek_cursor=s=f34691953f354fc19dbd06139503c160;i=156... OK
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] https=1 host=kubernetes.default.svc port=443
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] local POD info OK
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] testing connectivity with API server...
[2021/01/12 14:05:11] [ info] [filter:kubernetes:kubernetes.0] API server connectivity OK
[2021/01/12 14:05:11] [ info] [output:syslog:syslog.0] setup done for vrli.jarvis.lab:514
[2021/01/12 14:05:11] [ info] [http_server] listen iface=0.0.0.0 tcp_port=2020
[2021/01/12 14:05:11] [ info] [sp] stream processor started
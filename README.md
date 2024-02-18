# eks-kubecost Project

eks-cicd-dr is prototyping project for testing various architecture with kubecost and AWS EKS.

* [aws-terraform](https://github.com/ssup2-playground/eks-kubecost_aws-terraform) : Terraform for EKS clusters, Kubecost and AWS Resources

## Prometheus Architecture

<img src="/images/architecture-prom.png" width="600"/>

"Prometheus Architecture" is the most commonly used basic architecture when using kubecost. 

* Metrics required for cost calculation in Kubecost and cost metrics provided by Kubecost are collected by Prometheus.
* Kubecost retrieves cost metrics from Prometheus when visualizing cost.

### Prometheus Scrapper Setting for Kubecost

```yaml
    scrape_configs:
    - job_name: kubecost
      honor_labels: true
      scrape_interval: 1m
      scrape_timeout: 60s
      metrics_path: /metrics
      scheme: http
      dns_sd_configs:
      - names:
        - {{ template "cost-analyzer.serviceName" . }}
        type: 'A'
        port: 9003
    - job_name: kubecost-networking
      kubernetes_sd_configs:
        - role: pod
      relabel_configs:
      # Scrape only the the targets matching the following metadata
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
          action: keep
          regex:  kubecost
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
          action: keep
          regex:  network-costs
```

* "kubecost", "kubecost-networking" job must be set for Kubecost.

### Prometheus Recording Rules for Kubecost

```yaml
    rules:
      groups:
        - name: CPU
          rules:
            - expr: sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
              record: cluster:cpu_usage:rate5m
            - expr: rate(container_cpu_usage_seconds_total{container!=""}[5m])
              record: cluster:cpu_usage_nosum:rate5m
            - expr: avg(irate(container_cpu_usage_seconds_total{container!="POD", container!=""}[5m])) by (container,pod,namespace)
              record: kubecost_container_cpu_usage_irate
            - expr: sum(container_memory_working_set_bytes{container!="POD",container!=""}) by (container,pod,namespace)
              record: kubecost_container_memory_working_set_bytes
            - expr: sum(container_memory_working_set_bytes{container!="POD",container!=""})
              record: kubecost_cluster_memory_working_set_bytes
        - name: Savings
          rules:
            - expr: sum(avg(kube_pod_owner{owner_kind!="DaemonSet"}) by (pod) * sum(container_cpu_allocation) by (pod))
              record: kubecost_savings_cpu_allocation
              labels:
                daemonset: "false"
            - expr: sum(avg(kube_pod_owner{owner_kind="DaemonSet"}) by (pod) * sum(container_cpu_allocation) by (pod)) / sum(kube_node_info)
              record: kubecost_savings_cpu_allocation
              labels:
                daemonset: "true"
            - expr: sum(avg(kube_pod_owner{owner_kind!="DaemonSet"}) by (pod) * sum(container_memory_allocation_bytes) by (pod))
              record: kubecost_savings_memory_allocation_bytes
              labels:
                daemonset: "false"
            - expr: sum(avg(kube_pod_owner{owner_kind="DaemonSet"}) by (pod) * sum(container_memory_allocation_bytes) by (pod)) / sum(kube_node_info)
              record: kubecost_savings_memory_allocation_bytes
              labels:
                daemonset: "true"
```

* "CPU", "Savings" recording rules must be set for Kubecost.

## Prometheus AWS AMP Cost Architecture

<img src="/images/architecture-prom-ampcost.png" width="800"/>

"Prometheus AWS AMP Cost Architecture" is an architecture that centralizes only cost metrics in AWS AMP. It can be used when you want to visualize the cost of multiple AWS EKS clusters while optimizing the cost of using AWS AMP.

* Metrics required for cost calculation in Kubecost and cost metrics provided by Kubecost are collected by Prometheus.
* Kubecost retrieves cost metrics from Prometheus when visualizing cost.
* Prometheus writes cost metrics to AWS AMP

## Prometheus AWS AMP Architecture

<img src="/images/architecture-prom-amp.png" width="800"/>

"Prometheus AWS AMP Architecture" is the basic architecture for linking Kubecost and AWS AMP.

* Metrics required for cost calculation in Kubecost and cost metrics provided by Kubecost are collected by Prometheus.
* Prometheus writes collected metrics to AWS AMP.
* Kubecost retrieves cost metrics from AWS AMP when visualizing cost.

## ADOT Collector AWS AMP Architecture

<img src="/images/architecture-adot-amp.png" width="800"/>

"ADOT Collector AWS AMP Architecture" is an architecture in which ADOT collector replaces the metric collection role previously performed by Prometheus.

* Metrics required for cost calculation in Kubecost and cost metrics provided by Kubecost are collected by ADOT collector.
* ADOT collector writes collected metrics to AWS AMP.
* Kubecost retrieves cost metrics from AWS AMP when visualizing cost.

## AWS AMP Architecture

<img src="/images/architecture-amp.png" width="800"/>

"AWS AMP Architecture" is an agentless architecture that does not install agents to collect AWS EKS Cluster metrics using AWS AMP's Scrapper

* Metrics required for cost calculation in Kubecost and cost metrics provided by Kubecost are collected by AWS AMP Scrapper.
* Kubecost retrieves cost metrics from AWS AMP when visualizing cost.

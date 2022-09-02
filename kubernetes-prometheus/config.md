// The global configuration specifies parameters that are valid in all other configuration contexts. They also serve as defaults for other configuration sections.  
    global:  
// How frequently to scrape targets by default.Targets are the services which we wish to monitor.These can be also overwritten in individual scrape configs.This has to be tuned according to the size of clusters we dont want to overboard the data  
      scrape_interval: 5s  
// A list of scrape configurations. This is typically a section where we ask prometheus server to monitor particular URL's or metrics endpoints.In the general case, one scrape configuration specifies a single job. In advanced configurations, this may change. Targets can be static or dynamic.  
    scrape_configs:  
// The job name assigned to scraped metrics by default. This will be visible in the prometheus targets page  
// configuration responsible to scrape node exporter related data  
      - job_name: 'node-exporter'   
// service discovery: Service discovery (SD) enables you to provide that information to Prometheus from whichever database you store it in. Prometheus supports many common sources of service information, such as Consul, Amazon’s EC2, and Kubernetes out of the box  
// Kubernetes SD configurations allow retrieving scrape targets from Kubernetes' REST API and always staying synchronized with the cluster state.  
// List of Kubernetes service discovery configurations.  
        kubernetes_sd_configs:  
// Role: There can be multiple roles in this configs depending on what metrics you wish to scrape The endpoints role discovers targets from endpoints of a service.All the Service endpoints are scrapped if the service metadata is annotated with prometheus.io/scrape and prometheus.io/port annotations.  
          - role: endpoints  
//  relabel_configs: These relabeling steps are applied before the scrape occurs and only have access to labels added by Prometheus’ Service Discovery. They allow us to filter the targets returned by our SD mechanism, as well as manipulate the labels it sets.This way it gives us a granular control on what mertrics we wish to scrape using prometheus rather collecting all the data from metrics endpoints..All the relabelled clabelsa nd configs can be seen under service_discovery page from UI under each individual block  
  
// source_labels: It expects an array of one or more label names, which are used to select the respective label values  
  
// regex: A regular expression and is used to match the extracted value from source_labels  
  
// action:  There are multiple actions which are allowed in this configuration. Out of which the keep and drop actions allow us to filter out targets and metrics based on whether our label values match the provided regex.  
  
// In this particular block its recognizing the node exporter endpoint and scraping he metrics from node-exporter and keeping those metrics.  
        relabel_configs:  
        - source_labels: [__meta_kubernetes_endpoints_name] ## represents the name of the endpoints object.  
          regex: 'node-exporter' ## matches the regex  
          action: keep ## scrapes the metrics  
  
// configuration responsible to scrape kubernetes api server related data       
      - job_name: 'kubernetes-apiservers'  
        kubernetes_sd_configs:  
        - role: endpoints  
// Configures the protocol scheme used for requests. Prometheus talking to kubernetes api in a secured communication rather than http  
        scheme: https  
// In order to make theinternal secure tls communication it needs certs and token which are provided in below config  
        tls_config:  
// CA certificate to validate API server certificate with. This is available in each pod in the cluster by default   
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  
// token for authenticating to API server  
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  
        relabel_configs:  
// so the below block is responsible for scraping metrics from default namespace of service called kubernetes of endpoint https.Typically the defaulr kubernetes service endpoint when it resolves which consists of all api server related metrics  
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name] ## labels representing name of namespace,service and port  
          action: keep  
          regex: default;kubernetes;https   
		    
// configuration responsible to scrape kubernetes node related metrics like node ready status etc..    
      - job_name: 'kubernetes-nodes'  
        scheme: https  
        tls_config:  
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  
        kubernetes_sd_configs:  
// The node role discovers one target per cluster node( as in 3 nodes 3 targets) with the address defaulting to the Kubelet's HTTP port.  
  
// replacement: If the extracted value matches the given regex, then replacement gets populated by performing a regex replace  
  
// target_label: If the relabel action results in a value being written to some label, target_label defines to which label the replacement should be written  
  
        - role: node  
        relabel_configs:  
// the below block means all __meta_kubernetes_node_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_node_label_kubernetes_io_arch='amd64' to kubernetes_io_arch='amd64' i.e when you query in prometheus the field will be replaced as kubernetes_io_arch . Try executing kubelet_node_name in prometheus  
        - action: labelmap  
          regex: __meta_kubernetes_node_label_(.+)  
  
//  the below block means Endpoint adress within targets is by default replaced with kubernetes default service.It can be seen in targets page under kubernetes-nodes job..typically assiginign adress from where to get node metrics		    
        - target_label: __address__  
          replacement: kubernetes.default.svc:443  
		    
// the below block means that get the node name from the label __meta_kubernetes_node_name and replace it in __metrics_path__ with the given replacement  
eg : nodename is worker1 __metrics_path__ would be replaced by /api/v1/nodes/worker1/proxy/metrics    
  
        - source_labels: [__meta_kubernetes_node_name]  
          regex: (.+)  
          target_label: __metrics_path__  
          replacement: /api/v1/nodes/${1}/proxy/metrics       
  
// configuration responsible to scrape kubernetes pod related metrics       
      - job_name: 'kubernetes-pods'  
        kubernetes_sd_configs:  
// The pod role discovers all pods and exposes their containers as targets. For each declared port of a container, a single target is generated.   
        - role: pod  
        relabel_configs:  
		  
// This block means that scrape the metrics from  targets with label __meta_kubernetes_pod_annotation_prometheus_io_scrape equals 'true',which means the user added prometheus.io/scrape: true in the pods' annotation.  
  
// In my case nothing will be scrapped from this job cause I dont have any annotations on pods. This would be 0 on my service discovery tab  
  
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]  
          action: keep  
          regex: true  
		    
// This block means that if the user overrides the scrape path via pod annotations,its value will be put in __metrics_path__ and the metrics will be read from that path of the pod  
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]  
          action: replace  
          target_label: __metrics_path__  
          regex: (.+)  
// This block means that if the user overrides the port via pod annotations(application listening port to access metrics),if the user added prometheus.io/port in the pod annotation, use this port to replace the port in __address__.By default the __address__ label is set to the <host>:<port> address of the target.  
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]  
          action: replace  
          regex: ([^:]+)(?::\d+)?;(\d+)  
          replacement: $1:$2  
          target_label: __address__  
		    
// labelmap : The labelmap action is used to map one or more label pairs to different label names.  
  
// the below block means all __meta_kubernetes_pod_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_pod_label_app='armada-api' to app='armada-api' i.e when you query in prometheus the field will be replaced as app  
// this is done for easy understanding  
  
        - action: labelmap  
          regex: __meta_kubernetes_pod_label_(.+)  
		    
// This means if __meta_kubernetes_namespace matches .*(anything not empty), put its value in label kubernetes_namespace i.e when you query in prometheus the field will be replaced as kubernetes_namespace  
  
        - source_labels: [__meta_kubernetes_namespace]  
          action: replace  
          target_label: kubernetes_namespace  
		    
// This means if __meta_kubernetes_pod_name matches .*(anything not empty), put its value in label kubernetes_pod_name, i.e when you query in prometheus the field will be replaced as kubernetes_pod_name  
        - source_labels: [__meta_kubernetes_pod_name]  
          action: replace  
          target_label: kubernetes_pod_name  
  
// configuration responsible to scrape metrics from kube-state-metrics deployment.Metrics include data about kubernetes APi objects.Kube state metrics service exposes all the metrics on /metrics URI and the default metrics path is /metrics hence we dont explicitly mention it here  
  
// static configs: target to monitor.Here in this example, its listening to kubestate metrics  end point hence it can scrape  metrics. The default metrics path  is /metircs,It can be a comma seperated value in targets if there are multiple targets within same job  
        
      - job_name: 'kube-state-metrics'  
        static_configs:  
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']  
		    
// configuration responsible to scrape metrics from ca advisor..Typically the usage metrics  
      - job_name: 'kubernetes-cadvisor'  
        scheme: https  
        tls_config:  
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt  
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  
        kubernetes_sd_configs:  
        - role: node  
        relabel_configs:  
		  
// the below block means all __meta_kubernetes_node_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_node_label_kubernetes_io_arch='amd64' to kubernetes_io_arch='amd64' i.e when you query in prometheus the field will be replaced as kubernetes_io_arch..These renamed and replaced labels can be seen under service_discovery page from UI under each individual block  
  
        - action: labelmap  
          regex: __meta_kubernetes_node_label_(.+)  
		    
//  the below block means Endpoint adress within targets is by default replaced with kubernetes default service.It can be seen in targets page under ca advisor job..typically assiginign adress from where to get caadvisor metrics  
        - target_label: __address__  
          replacement: kubernetes.default.svc:443  
		    
// the below block means that get the node name from the label __meta_kubernetes_node_name and replace it in __metrics_path__ with the given replacement  
eg : nodename is worker1 __metrics_path__ would be replaced by /api/v1/nodes/worker1/proxy/metrics/cadvisor  
        - source_labels: [__meta_kubernetes_node_name]  
          regex: (.+)  
          target_label: __metrics_path__  
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor  
  
  
// configuration responsible to scrape metrics from kubernetes service endpoints when we use internal services directly  
  
      - job_name: 'kubernetes-service-endpoints'  
        kubernetes_sd_configs:  
        - role: endpoints  
        relabel_configs:  
  
// This block means that scrape the metrics from  targets with label __meta_kubernetes_service_annotation_prometheus_io_scrape equals 'true',which means the user added prometheus.io/scrape: true in the service' annotation.   
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]  
          action: keep  
          regex: true  
// This block means that replace the __scheme__(default http) accordingly if user mentioned anything in the service annotation prometheus.io/scheme in the service' annotation.   
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]  
          action: replace  
          target_label: __scheme__  
          regex: (https?)  
		  
// This is also for service annotation of prometheus, if the user overrides the scrape path, its value will be put in __metrics_path__  
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]  
          action: replace  
          target_label: __metrics_path__  
          regex: (.+)  
// This block means that if the user overrides the port via service annotations(application listening port to access metrics),if the user added prometheus.io/port in the service annotation, use this port to replace the port in __address__.By default the __address__ label is set to the <host>:<port> address of the target.  
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]  
          action: replace  
          target_label: __address__  
          regex: ([^:]+)(?::\d+)?;(\d+)  
          replacement: $1:$2  
  
// the below block means all __meta_kubernetes_service_label_(.+) will be changed to the (.+)  
// e.g. __meta_kubernetes_service_label_k8s_app='kube-dns' to k8s_app='kube-dns' i.e when you query in prometheus the field will be replaced as k8s_app..Try executing coredns_build_info metric in prometheus	  
  
        - action: labelmap  
          regex: __meta_kubernetes_service_label_(.+)  
		    
// the below block means all __meta_kubernetes_namespace will be changed to kubernetes_namespace  
// e.g. __meta_kubernetes_namespace='kube-system' to kubernetes_namespace='kube-system' i.e when you query in prometheus the field will be replaced as kubernetes_namespace	..Try executing coredns_build_info metric in prometheus	  
  
        - source_labels: [__meta_kubernetes_namespace]  
          action: replace  
          target_label: kubernetes_namespace  
		    
// the below block means all __meta_kubernetes_service_name will be changed to kubernetes_name  
// e.g. __meta_kubernetes_service_name='kube-dns' to kubernetes_name='kube-dns' i.e when you query in prometheus the field will be replaced as kubernetes_name	..Try executing coredns_build_info metric in prometheus	  
        - source_labels: [__meta_kubernetes_service_name]  
          action: replace  
          target_label: kubernetes_name  

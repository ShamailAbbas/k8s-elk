# k8s-elk

Implementing the ELK Stack (Elasticsearch, Logstash, and Kibana) for monitoring your Kubernetes (k8s) cluster is a great way to centralize and visualize logs. Below is a step-by-step guide to help you set up the ELK Stack on your Kubernetes cluster.

---

### **Step 1: Prerequisites**
1. **Kubernetes Cluster**: Ensure your Kubernetes cluster is up and running.
2. **kubectl**: Install and configure `kubectl` to interact with your cluster.
3. **Helm**: Install Helm, a package manager for Kubernetes, to simplify the deployment of the ELK Stack.
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```
4. **Storage Class**: Ensure you have a storage class configured for persistent volumes (PVs) if you want persistent storage for Elasticsearch.

---

### **Step 2: Add Helm Repositories**
Add the Elastic Helm charts repository to your Helm configuration:
```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

---

### **Step 3: Deploy Elasticsearch**
Elasticsearch is the core component of the ELK Stack, used for storing and indexing logs.

1. Create a namespace for the ELK Stack:
   ```bash
   kubectl create namespace logging
   ```

2. Deploy Elasticsearch using Helm:
   ```bash
   helm install elasticsearch elastic/elasticsearch \
     --namespace logging \
     --set replicas=1 \
     --set persistence.enabled=true \
     --set persistence.size=10Gi
   ```
   - Adjust `replicas` and `persistence.size` as per your requirements.
   - If you don't need persistent storage, set `persistence.enabled=false`.

3. Verify the deployment:
   ```bash
   kubectl get pods -n logging
   ```

---

### **Step 4: Deploy Kibana**
Kibana is the visualization layer for Elasticsearch.

1. Deploy Kibana using Helm:
   ```bash
   helm install kibana elastic/kibana \
     --namespace logging \
     --set service.type=ClusterIP
   ```

2. Expose Kibana (optional):
   - If you want to access Kibana externally, change `service.type` to `LoadBalancer` or `NodePort`.
   - Alternatively, use port-forwarding:
     ```bash
     kubectl port-forward svc/kibana-kibana 5601:5601 -n logging
     ```
   - Access Kibana at `http://localhost:5601`.

---

### **Step 5: Deploy Logstash (Optional)**
Logstash is used for processing and transforming logs before sending them to Elasticsearch. If you don't need log transformation, you can skip this step.

1. Deploy Logstash using Helm:
   ```bash
   helm install logstash elastic/logstash \
     --namespace logging \
     --set replicas=1
   ```

2. Configure Logstash pipelines to process logs as needed.

---

### **Step 6: Deploy Filebeat**
Filebeat is a lightweight shipper for forwarding logs to Elasticsearch or Logstash.

1. Deploy Filebeat using Helm:
   ```bash
   helm install filebeat elastic/filebeat \
     --namespace logging \
     --set daemonset.enabled=true \
     --set daemonset.filebeatConfig.inputs[0].type=container \
     --set daemonset.filebeatConfig.inputs[0].paths[0]="/var/log/containers/*.log"
   ```

2. Verify Filebeat is running:
   ```bash
   kubectl get pods -n logging -l app=filebeat
   ```

---

### **Step 7: Configure Kibana**
1. Access Kibana (as described in Step 4).
2. Go to **Stack Management** > **Index Patterns** and create an index pattern for Filebeat logs (e.g., `filebeat-*`).
3. Explore logs in the **Discover** section.

---

### **Step 8: Monitor Kubernetes Logs**
Filebeat will automatically collect logs from all containers in your cluster and send them to Elasticsearch. You can visualize and analyze these logs in Kibana.

---

### **Step 9: Advanced Configurations**
1. **Ingress**: Set up an Ingress controller to expose Kibana externally.
2. **Security**: Enable authentication and TLS for Elasticsearch and Kibana.
3. **Scaling**: Scale Elasticsearch and Logstash based on your cluster's workload.
4. **Custom Logs**: Modify Filebeat configurations to collect custom logs or metrics.

---

### **Step 10: Clean Up**
If you want to remove the ELK Stack:
```bash
helm uninstall elasticsearch -n logging
helm uninstall kibana -n logging
helm uninstall filebeat -n logging
helm uninstall logstash -n logging
kubectl delete namespace logging
```

---

This setup provides a basic ELK Stack for Kubernetes logging. You can further customize it based on your specific requirements. Let me know if you need help with any specific step!

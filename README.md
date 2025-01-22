 Let’s go through the **complete A-Z setup for the ELK stack (Elasticsearch, Logstash, Kibana) on your Kubernetes cluster running on a VirtualBox VM **. 

---

### **Step 1: Prerequisites**
1. **Kubernetes Cluster**: Ensure your Kubernetes cluster is up and running on your VirtualBox VM.
2. **kubectl**: Ensure `kubectl` is installed and configured to interact with your cluster.
3. **Helm**: Install Helm if you haven’t already.

   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   helm version
   ```

4. **Storage**: Ensure you have a directory for persistent storage on your VM (e.g., `/mnt/data/elasticsearch`).

---

### **Step 2: Create the `logging` Namespace**
Create a dedicated namespace for your ELK stack:

```bash
kubectl create namespace logging
```

---

### **Step 3: Set Up Persistent Storage**
Elasticsearch requires persistent storage to store its data. We’ll use **local persistent volumes** for this setup.

1. **Create a Directory for Persistent Storage**:
   SSH into your VirtualBox VM and create a directory for Elasticsearch data:

   ```bash
   sudo mkdir -p /mnt/data/elasticsearch
   sudo chmod 777 /mnt/data/elasticsearch
   ```

2. **Create a PersistentVolume (PV)**:
   Create a YAML file for the PersistentVolume. For example, `elasticsearch-pv.yaml`:

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: elasticsearch-pv
     labels:
       type: local
   spec:
     storageClassName: manual
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     hostPath:
       path: "/mnt/data/elasticsearch"
   ```

   Apply the PV:

   ```bash
   kubectl apply -f elasticsearch-pv.yaml
   ```

3. **Create a PersistentVolumeClaim (PVC)**:
   Create a YAML file for the PersistentVolumeClaim. For example, `elasticsearch-pvc.yaml`:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: elasticsearch-pvc
     namespace: logging
   spec:
     storageClassName: manual
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
   ```

   Apply the PVC:

   ```bash
   kubectl apply -f elasticsearch-pvc.yaml
   ```

4. **Verify PV and PVC**:
   Check that the PV and PVC are bound:

   ```bash
   kubectl get pv
   kubectl get pvc -n logging
   ```

---

### **Step 4: Add the Elastic Helm Charts Repository**
Elastic provides Helm charts to easily deploy the ELK stack.

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

---

### **Step 5: Deploy Elasticsearch**
Deploy Elasticsearch with persistent storage.

1. **Create a Helm values file for Elasticsearch**:
   Create a `values.yaml` file to customize the Elasticsearch Helm chart:

   ```yaml
   # values.yaml
   volumeClaimTemplate:
     accessModes: [ "ReadWriteOnce" ]
     storageClassName: "manual" # Match the StorageClass used in PVC
     resources:
       requests:
         storage: 10Gi
   ```

2. **Install Elasticsearch with Helm**:
   Use the `values.yaml` file to deploy Elasticsearch:

   ```bash
   helm install elasticsearch elastic/elasticsearch \
     --namespace logging \
     --values values.yaml
   ```

3. **Verify Elasticsearch Pods**:
   Check that the Elasticsearch pods are running:

   ```bash
   kubectl get pods -n logging
   ```

---

### **Step 6: Deploy Kibana**
Kibana doesn’t require persistent storage, so you can deploy it as-is:

```bash
helm install kibana elastic/kibana --namespace logging
```

---

### **Step 7: Deploy Logstash (Optional)**
If you need Logstash, deploy it similarly. Logstash typically doesn’t require persistent storage unless you’re using it for buffering.

```bash
helm install logstash elastic/logstash --namespace logging
```

---

### **Step 8: Verify Persistent Storage**
1. **Check Elasticsearch Data**:
   SSH into your VirtualBox VM and verify that data is being written to the persistent volume:

   ```bash
   ls -l /mnt/data/elasticsearch
   ```

2. **Test Data Persistence**:
   Delete the Elasticsearch pod and verify that the data persists after the pod restarts:

   ```bash
   kubectl delete pod elasticsearch-master-0 -n logging
   kubectl get pods -n logging
   ```

---

### **Step 9: Access Kibana**
Expose Kibana using port-forwarding or a NodePort service.

#### **Option 1: Port-Forwarding**
```bash
kubectl port-forward svc/kibana-kibana 5601:5601 -n logging
```

Access Kibana at `http://localhost:5601`.

#### **Option 2: Expose Kibana via NodePort**
Edit the Kibana service to use `NodePort`:

```bash
kubectl edit svc kibana-kibana -n logging
```

Change the `type` to `NodePort` and save the changes. Then, access Kibana using the IP of your Kubernetes node and the assigned port.

---

### **Step 10: Ingest Logs (Optional)**
You can use **Filebeat** or **Fluentd** to send logs to Elasticsearch. Here’s an example of deploying Filebeat:

1. **Install Filebeat with Helm**:
   ```bash
   helm install filebeat elastic/filebeat --namespace logging
   ```

2. **Configure Filebeat**:
   Modify the Filebeat configuration to collect logs from your desired sources.

---

### **Step 11: Configure Kibana**
1. Open Kibana in your browser (`http://localhost:5601`).
2. Go to **Stack Management** > **Index Patterns** and create an index pattern for your logs.
3. Explore the **Discover** tab to view your logs.

---

### **Step 12: Monitor and Maintain**
- **Monitor Pods**:
  Use `kubectl get pods -n logging` to monitor the status of your ELK stack components.
- **Scale Elasticsearch**:
  If needed, scale Elasticsearch by updating the Helm release:

  ```bash
  helm upgrade elasticsearch elastic/elasticsearch --namespace logging --set replicas=3
  ```

- **Backup Data**:
  Regularly back up your Elasticsearch data stored in the persistent volume.

---

### **Troubleshooting**
- **PVC Not Bound**:
  Ensure the StorageClass and PVC configurations match.
- **Pod Failing**:
  Check the pod logs for errors:

  ```bash
  kubectl logs <pod-name> -n logging
  ```

- **Insufficient Storage**:
  Increase the storage size in the PVC and PV definitions.

---

### **Conclusion**
You now have a fully functional ELK stack running in the `logging` namespace on your Kubernetes cluster with persistent storage. This setup ensures that your Elasticsearch data is retained even if pods are restarted or rescheduled. Let me know if you need further assistance! 🚀

## **Azure Kubernetes Deployment Guide**

### **1️⃣ Create Resource Group & AKS Cluster**

#### 🔹 (a) Create the Resource Group
```sh
az group create --name rg-greetinglab-ds-centralus --location centralus
```

#### 🔹 (b) Create the AKS Cluster with a Default Node Pool
```sh
az aks create \
  --resource-group rg-greetinglab-ds-centralus \
  --name aks-greetinglab-ds \
  --node-count 1 \
  --generate-ssh-keys
```

#### 🔹 (c) Add Two Additional Node Pools (Frontend & Backend)
```sh
az aks nodepool add \
  --resource-group rg-greetinglab-ds-centralus \
  --cluster-name aks-greetinglab-ds \
  --name frontendpool \
  --node-count 1 \
  --labels agentpool=frontendpool

az aks nodepool add \
  --resource-group rg-greetinglab-ds-centralus \
  --cluster-name aks-greetinglab-ds \
  --name backendpool \
  --node-count 1 \
  --labels agentpool=backendpool
```

#### ✅ Verify AKS Cluster
```sh
az aks show --resource-group rg-greetinglab-ds-centralus --name aks-greetinglab-ds --query "agentPoolProfiles[].{Name:name, Count:count}"
```

---

### **2️⃣ Create Azure Container Registry (ACR)**

#### 🔹 (a) Create ACR
```sh
az acr create \
  --resource-group rg-greetinglab-ds-centralus \
  --name acrgreetinglabds \
  --sku Basic \
  --admin-enabled true
```

#### ✅ Verify ACR Creation
```sh
az acr show --resource-group rg-greetinglab-ds-centralus --name acrgreetinglabds --query "loginServer" --output tsv
```

#### 🔹 (b) Retrieve ACR Credentials
```sh
ACR_NAME=acrgreetinglabds
ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query "username" --output tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" --output tsv)
```

#### 🔹 (c) Create `.dockerconfigjson` for Kubernetes Secret
```sh
cat <<EOF > config.json
{
  "auths": {
    "$ACR_NAME.azurecr.io": {
      "username": "$ACR_USERNAME",
      "password": "$ACR_PASSWORD",
      "auth": "$(echo -n "$ACR_USERNAME:$ACR_PASSWORD" | base64 -w 0)"
    }
  }
}
EOF
```

#### 🔹 (d) Encode & Store in Kubernetes Secret
```sh
cat config.json | base64 -w 0
```
**Paste the Base64 value inside `acr-secret.yaml`:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: acr-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <PASTE_BASE64_ENCODED_VALUE>
```

#### 🔹 (e) Apply the Secret
```sh
kubectl apply -f acr-secret.yaml
kubectl get secret acr-secret --output=yaml
```

---

### **3️⃣ Build & Push Docker Image to ACR**

#### 🔹 (a) Build the Docker Image
```sh
ACR_NAME=acrgreetinglabds
IMAGE_NAME=greetings-app
TAG=latest

docker build -t $ACR_NAME.azurecr.io/$IMAGE_NAME:$TAG .
```

#### 🔹 (b) Log in to ACR
```sh
az acr login --name acrgreetinglabds
```

#### 🔹 (c) Push the Image to ACR
```sh
docker push acrgreetinglabds.azurecr.io/greetings-app:latest
```

#### ✅ Verify Image in ACR
```sh
az acr repository list --name acrgreetinglabds --output table
az acr repository show-tags --name acrgreetinglabds --repository greetings-app --output table
```

#### 🔹 (d) Update `k8s-deployment.yaml`
```yaml
containers:
  - name: greetings-app
    image: acrgreetinglabds.azurecr.io/greetings-app:latest
```

---

### **4️⃣ Deploy Application in AKS**

#### 🔹 (a) Connect to AKS Cluster
```sh
az aks get-credentials --resource-group rg-greetinglab-ds-centralus --name aks-greetinglab-ds
kubectl get nodes
```

#### 🔹 (b) Deploy Redis First
```sh
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
kubectl get pods
kubectl get svc
```

#### 🔹 (c) Deploy the Greetings App
```sh
kubectl apply -f k8s-deployment.yaml
kubectl apply -f greetings-config.yaml
kubectl get pods
```

#### 🔹 (d) Expose the Greetings App
```sh
kubectl apply -f greetings-service.yaml
kubectl get svc
```

---

### **5️⃣ Access the Greetings App in Browser**
#### 🔹 (a) Get External IP
```sh
kubectl get svc
```
#### 🔹 (b) Open in Browser
```sh
http://<EXTERNAL-IP>
```
✔ Expected Output:
```
Greetings from DavitS 1 times!!!
```

---

### ✅ **Deployment Completed Successfully!** 🎉

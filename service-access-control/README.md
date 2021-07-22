### Service access control


Useful links:
- https://cloud.google.com/service-mesh/docs/security/authorization-policy-overview?authuser=1
- https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy?authuser=1&_ga=2.63111404.-2101012435.1612832431#enabling_network_policy_enforcement

#### 1. Create a GKE cluster
```

export PROJECT_ID=$(gcloud info --format='value(config.project)')
export CLUSTER_NAME="glau-asm-gke-cluster"
export CLUSTER_LOCATION=us-west1-a

./create_cluster.sh $CLUSTER_NAME $CLUSTER_LOCATION
```

#### 2. Install ASM on the GKE cluster
Download ASM installation script
```
curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.10 > install_asm
curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.10.sha256 > install_asm.sha256
sha256sum -c --ignore-missing install_asm.sha256
chmod +x install_asm
```
Install Anthos Service Mesh (ASM)
Please make sure you have all the required [GCP IAM permissions](https://cloud.google.com/service-mesh/docs/installation-permissions) before running the script below.
```
./install_asm \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME  \
  --cluster_location $CLUSTER_LOCATION  \
  --mode install \
  --output_dir ./asm-downloads \
  --enable_all
```

#### 3. Install RE operator bundle
```
kubectl create namespace redis
kubectl config set-context --current --namespace=redis

kubectl apply -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/v6.0.20-12/bundle.yaml
```  


#### 4. Install REC
```
kubectl apply -f - <<EOF
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseCluster
namespace: redis
metadata:
  name: rec
spec:
  nodes: 3
EOF
```


#### 5. Create a REDB
```
kubectl apply -f - <<EOF
apiVersion: app.redislabs.com/v1alpha1
kind: RedisEnterpriseDatabase
namespace: redis
metadata:
  name: redis-enterprise-database
spec:
  memorySize: 100MB
```  


#### 6. Deploy redisbank in "redis" namespace
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redisbank-deployment
spec:
  selector:
    matchLabels:
      app: redisbank-app
  replicas: 1
  template:
    metadata:
      labels:
        app: redisbank-app
    spec:
      containers:
      - name: redisbank-app
        image: gcr.io/central-beach-194106/redisbank:0.0.1
        imagePullPolicy: Always
        ports:
        - name: redisbank-web
          containerPort: 8080
        env:
        - name: SPRING_REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: redb-redis-enterprise-database
              key: service_name
        - name: SPRING_REDIS_PORT
          valueFrom:
            secretKeyRef:
              name: redb-redis-enterprise-database
              key: port
        - name: SPRING_REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redb-redis-enterprise-database
              key: password
EOF
```  
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: redisbank-service
spec:
  type: LoadBalancer
  selector:
    app: redisbank-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
EOF
```


#### 7. Copy REDB secret from "redis" ns to "default" ns
```
kubectl get secret redb-redis-enterprise-database -n redis  -o yaml | grep -v '^\s*namespace:\s' | kubectl apply --namespace=default -f -
```


#### 8. Deploy redisbank in "default" namespace
```
kubectl config set-context --current --namespace=redis

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redisbank-deployment
spec:
  selector:
    matchLabels:
      app: redisbank-app
  replicas: 1
  template:
    metadata:
      labels:
        app: redisbank-app
    spec:
      containers:
      - name: redisbank-app
        image: gcr.io/central-beach-194106/redisbank:0.0.1
        imagePullPolicy: Always
        ports:
        - name: redisbank-web
          containerPort: 8080
        env:
        - name: SPRING_REDIS_HOST
          value: redis-enterprise-database-headless.redis.svc.cluster.local
        - name: SPRING_REDIS_PORT
          valueFrom:
            secretKeyRef:
              name: redb-redis-enterprise-database
              key: port
        - name: SPRING_REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redb-redis-enterprise-database
              key: password
EOF
```  
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: redisbank-service
spec:
  type: LoadBalancer
  selector:
    app: redisbank-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
EOF
```


#### 9. Verify redisbank app in default ns can connect to the REDB
```
kubectl logs -f redisbank-deployment-<xxxxxxxxxxxxxx> -n default
```


#### 10. Create a service access control policy
The following policy is to block access from resources running in non "redis" namespace to the REDB
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: redis-enterprise-database-headless-deny
 namespace: redis
spec:
 selector:
   matchLabels:
     app: redis-enterprise
 action: DENY
 rules:
 - from:
   - source:
       notNamespaces: ["redis"]
```


#### 11. Verify redisbank-deployment-xxxx pod no longer connecting to the REDB
```
kubectl logs -f redisbank-deployment-<xxxxxxxxxxxxxx> -n default
```




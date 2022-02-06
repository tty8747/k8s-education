# Homework

## Using `kubectl -n kube-system describe ...`
```bash
for i in $(kubectl -n kube-system get po | grep -v NAME | awk -F' ' '{ print $1}' | xargs echo); do print $i; kubectl -n kube-system describe po $i | grep "Controlled By:"; done
coredns-64897985d-rfn9q
Controlled By:  ReplicaSet/coredns-64897985d
etcd-minikube
Controlled By:  Node/minikube
kube-apiserver-minikube
Controlled By:  Node/minikube
kube-controller-manager-minikube
Controlled By:  Node/minikube
kube-proxy-2cpl4
Controlled By:  DaemonSet/kube-proxy
kube-scheduler-minikube
Controlled By:  Node/minikube
metrics-server-68fbbb47dc-s76m7
Controlled By:  ReplicaSet/metrics-server-68fbbb47dc
metrics-server-7479797668-rtkw6
Controlled By:  ReplicaSet/metrics-server-7479797668
storage-provisioner
```

## Canary

Create namespace canary
```bash
kubectl create namespace canary
kubectl config set-context --current --namespace=canary
```

Create v1 app config
```bash
cat <<- EOF | tee ./v1.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
    location / {
        return 200 "hostname is \$hostname\n App version is v1\n";
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
EOF
```

Create v2 app config
```bash
cat <<- EOF | tee ./v2.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;
    location / {
        return 200 "hostname is \$hostname\n App version is v2\n";
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
EOF
```

Create configmap and apply it
```bash
kubectl create configmap nginx-v1 --from-file=v1.conf --dry-run=client -o yaml | grep -v creationTimestamp > cmv1.yml
kubectl create configmap nginx-v2 --from-file=v2.conf --dry-run=client -o yaml | grep -v creationTimestamp > cmv2.yml
kubectl apply -f cmv1.yml
kubectl apply -f cmv2.yml
```

Create deploy
```bash
kubectl create deployment nginx-v1 --image=nginx --replicas=1 --dry-run=client -o yaml | grep -v creationTimestamp > nginx-deployment-v1.yml
kubectl create deployment nginx-v2 --image=nginx --replicas=1 --dry-run=client -o yaml | grep -v creationTimestamp > nginx-deployment-v2.yml
```

Add lines to nginx-deplyment-v1.yml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-v1
  name: nginx-v1
spec:
  replicas: 8
  selector:
    matchLabels:
      app: nginx-v1
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx-v1
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: v1.conf
        resources: {}
      volumes:
      - name: app-conf
        configMap:
          name: nginx-v1
status: {}
```

Add lines to nginx-deplyment-v1.yml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-v2
  name: nginx-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-v2
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx-v2
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-conf2
          mountPath: /etc/nginx/conf.d
        resources: {}
      volumes:
      - name: app-conf2
        configMap:
          name: nginx-v2
status: {}
```

```bash
kubectl apply -f nginx-deployment-v1.yml
kubectl apply -f nginx-deployment-v2.yml
```

```bash
kubectl port-forward nginx-v1-646d74696c-6dzn6 8080:80
curl "http://localhost:8080"                 
hostname is nginx-v1-646d74696c-t289j
 App version is v1
```

Loadbalancer templates
```bash
kubectl expose deployment nginx-v1 --type=ClusterIP --dry-run=client -o yaml | grep -v creationTimestamp > nginx-v1-lb.yml
kubectl expose deployment nginx-v2 --type=ClusterIP --dry-run=client -o yaml | grep -v creationTimestamp > nginx-v2-lb.yml

kubectl apply -f nginx-v1-lb.yml
kubectl apply -f nginx-v2-lb.yml
```

Create L7 LoadBalancer for v1
```bash
cat <<-EOF | tee nginx-ingress-v1.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  annotations:
    # https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
             name: nginx-v1
             port: 
                number: 80
EOF
```

```bash
kubectl apply -f nginx-ingress-v1.yml
```

We get only v1
```bash
for i in {1..15}; do curl $(minikube ip); done
```

Create L7 LoadBalancer for v2
```bash
cat <<-EOF | tee nginx-ingress-v2.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-canary
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "canary"
    nginx.ingress.kubernetes.io/canary-by-header-pattern: "always|yes|true|ok"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
             name: nginx-v2
             port: 
                number: 80
EOF
```

> We can set percent, using these annotations. (More info: [link1](https://intl.cloud.tencent.com/ko/document/product/457/38413), [link2](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/))
> ```bash
> annotations:
>   kubernetes.io/ingress.class: "nginx"
>   nginx.ingress.kubernetes.io/canary: "true"
>   nginx.ingress.kubernetes.io/canary-weight: "10"
> spec:
> ```

```bash
kubectl apply -f nginx-ingress-v2.yml
```

Final part:
```bash
curl -H 'canary: ' $(minikube ip)
hostname is nginx-v1-646d74696c-whhhc
 App version is v1

curl -H 'canary: always' $(minikube ip)
hostname is nginx-v2-5895767dd8-j9blz
 App version is v2

curl -H 'canary: ok' $(minikube ip)
hostname is nginx-v2-5895767dd8-vxq75
 App version is v2


kubectl get all -n canary
NAME                            READY   STATUS    RESTARTS   AGE
pod/nginx-v1-646d74696c-6dzn6   1/1     Running   0          109s
pod/nginx-v1-646d74696c-724cv   1/1     Running   0          109s
pod/nginx-v1-646d74696c-7wxdp   1/1     Running   0          109s
pod/nginx-v1-646d74696c-8rp7h   1/1     Running   0          109s
pod/nginx-v1-646d74696c-ndlpb   1/1     Running   0          109s
pod/nginx-v1-646d74696c-q6ktf   1/1     Running   0          109s
pod/nginx-v1-646d74696c-vhmgj   1/1     Running   0          109s
pod/nginx-v1-646d74696c-zrjpw   1/1     Running   0          109s
pod/nginx-v2-5895767dd8-f4247   1/1     Running   0          106s
pod/nginx-v2-5895767dd8-pz6vv   1/1     Running   0          106s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/nginx-v1   ClusterIP   10.111.96.247   <none>        80/TCP    82s
service/nginx-v2   ClusterIP   10.108.120.26   <none>        80/TCP    78s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-v1   8/8     8            8           109s
deployment.apps/nginx-v2   2/2     2            2           106s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-v1-646d74696c   8         8         8       109s
replicaset.apps/nginx-v2-5895767dd8   2         2         2       106s


kubectl config set-context --current --namespace=default
kubectl delete namespace canary
namespacetodelete=canary
kubectl get namespace "$namespacetodelete" -o json \
  | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" \
  | kubectl replace --raw /api/v1/namespaces/"$namespacetodelete"/finalize -f - | jq '.'
```


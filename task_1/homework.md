# Homework

I use kvm on linux
```bash
minikube start --driver=kvm2
```

Get file for deployment
```bash
kubectl create deployment --replicas=2 nginx --image=nginx --dry-run=client -o yaml > deployment.yml
```

```bash
cat deployment.yml          
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

Use `deployment.yml` with kubectl
```
kubectl -f deployment.yml
```

Remove one of pods
```
kubectl get po | grep 85b98978db          
nginx-85b98978db-jpzt5   1/1     Running   0          30s
nginx-85b98978db-mwlpd   1/1     Running   0          61s

kubectl delete po nginx-85b98978db-jpzt5                     
pod "nginx-85b98978db-jpzt5" deleted

kubectl get po | grep 85b98978db        
nginx-85b98978db-2cnxf   0/1     ContainerCreating   0          4s
nginx-85b98978db-mwlpd   1/1     Running             0          82s

kubectl get po | grep 85b98978db
nginx-85b98978db-2cnxf   1/1     Running   0          8s
nginx-85b98978db-mwlpd   1/1     Running   0          86s
```

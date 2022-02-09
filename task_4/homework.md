# Homework

4.1 Create users deploy_view and deploy_edit. Give the user deploy_view rights only to view deployments, pods. Give the user deploy_edit full rights to the objects deployments, pods.

Create private keys
```bash
openssl genrsa -out deploy_view.key 2048
openssl genrsa -out deploy_edit.key 2048
```

Create a certificate signing requests
```bash
openssl req -new -key deploy_view.key \
-out deploy_view.csr \
-subj "/CN=deploy_view"

openssl req -new -key deploy_edit.key \
-out deploy_edit.csr \
-subj "/CN=deploy_edit"
```

Sign the CSRs in the Kubernetes CA.
```bash
openssl x509 -req -in deploy_view.csr \
-CA ~/.minikube/ca.crt \
-CAkey ~/.minikube/ca.key \
-CAcreateserial \
-out deploy_view.crt -days 500

openssl x509 -req -in deploy_edit.csr \
-CA ~/.minikube/ca.crt \
-CAkey ~/.minikube/ca.key \
-CAcreateserial \
-out deploy_edit.crt -days 500
```

Create users in kubernetes
```bash
kubectl config set-credentials deploy_view \
--client-certificate=deploy_view.crt \
--client-key=deploy_view.key

kubectl config set-credentials deploy_edit \
--client-certificate=deploy_edit.crt \
--client-key=deploy_edit.key
```

Set context for users
```bash
kubectl config set-context deploy_view \
--cluster=minikube --user=deploy_view

kubectl config set-context deploy_edit \
--cluster=minikube --user=deploy_edit
```

Apply roles
```bash
# https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/
kubectl api-resources --sort-by name -o wide | grep clusterroles

kubectl create clusterrole deploy_view --verb=get,list,watch --resource=deployments,pods --dry-run=client -o yaml | grep -v creationTimestamp | kubectl apply -f -
kubectl create clusterrole deploy_edit --verb='*' --resource=deployments,pods --dry-run=client -o yaml | grep -v creationTimestamp | kubectl apply -f -
```

Apply role bindings
```bash
kubectl api-resources --sort-by name -o wide | grep clusterrolebindings

kubectl create clusterrolebinding deploy_view --clusterrole=deploy_view --user=deploy_view --dry-run=client -o yaml | grep -v creationTimestamp | kubectl apply -f -
kubectl create clusterrolebinding deploy_edit --clusterrole=deploy_edit --user=deploy_edit --dry-run=client -o yaml | grep -v creationTimestamp | kubectl apply -f -

```

Examine rules
```bash
kubectl describe clusterrole deploy_view
Name:         deploy_view
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  pods              []                 []              [get list watch]
  deployments.apps  []                 []              [get list watch]

kubectl describe clusterrole deploy_edit
Name:         deploy_edit
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  pods              []                 []              [*]
  deployments.apps  []                 []              [*]

kubectl describe clusterrolebindings.rbac.authorization.k8s.io deploy_edit 
Name:         deploy_edit
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  deploy_edit
Subjects:
  Kind  Name         Namespace
  ----  ----         ---------
  User  deploy_edit  

kubectl describe clusterrolebindings.rbac.authorization.k8s.io deploy_view
Name:         deploy_view
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  deploy_view
Subjects:
  Kind  Name         Namespace
  ----  ----         ---------
  User  deploy_view  

kubectl config use-context deploy_view 
Switched to context "deploy_view".

kubectl config get-contexts           
CURRENT   NAME          CLUSTER            AUTHINFO      NAMESPACE
          deploy_edit   minikube           deploy_edit   
*         deploy_view   minikube           deploy_view   
          k8s_user      minikube           k8s_user      
          microk8s      microk8s-cluster   admin         
          minikube      minikube           minikube      default


kubectl config use-context deploy_view 
Switched to context "deploy_view".

kubectl get all
NAME            READY   STATUS             RESTARTS        AGE
pod/my-nginx    0/1     ImagePullBackOff   0               114s
pod/my-nginx2   1/1     Running            0               57s
pod/pod         0/1     CrashLoopBackOff   5 (2m43s ago)   5m58s
Error from server (Forbidden): replicationcontrollers is forbidden: User "deploy_view" cannot list resource "replicationcontrollers" in API group "" in the namespace "default"
Error from server (Forbidden): services is forbidden: User "deploy_view" cannot list resource "services" in API group "" in the namespace "default"
Error from server (Forbidden): daemonsets.apps is forbidden: User "deploy_view" cannot list resource "daemonsets" in API group "apps" in the namespace "default"
Error from server (Forbidden): replicasets.apps is forbidden: User "deploy_view" cannot list resource "replicasets" in API group "apps" in the namespace "default"
Error from server (Forbidden): statefulsets.apps is forbidden: User "deploy_view" cannot list resource "statefulsets" in API group "apps" in the namespace "default"
Error from server (Forbidden): horizontalpodautoscalers.autoscaling is forbidden: User "deploy_view" cannot list resource "horizontalpodautoscalers" in API group "autoscaling" in the namespace "default"
Error from server (Forbidden): cronjobs.batch is forbidden: User "deploy_view" cannot list resource "cronjobs" in API group "batch" in the namespace "default"
Error from server (Forbidden): jobs.batch is forbidden: User "deploy_view" cannot list resource "jobs" in API group "batch" in the namespace "default"
```

4.2 Create namespace prod. Create users prod_admin, prod_view. Give the user prod_admin admin rights on ns prod, give the user prod_view only view rights on namespace prod.


kubectl create namespace prod

Create necessary things
```bash
kubectl create namespace prod
users=(prod_admin prod_view)

for i in ${users[@]}; do \
  openssl genrsa -out $i.key 2048 \
	&& openssl req -new -key $i.key \
	-out $i.csr \
	-subj "/CN=$i" \
	&& openssl x509 -req -in $i.csr \
	-CA ~/.minikube/ca.crt \
	-CAkey ~/.minikube/ca.key \
	-CAcreateserial \
	-out $i.crt -days 500 \
	&& kubectl config set-credentials $i \
	--client-certificate=$i.crt \
	--client-key=$i.key \
	&& kubectl config set-context $i \
	--cluster=minikube --user=$i; \
done
```


Apply roles
```bash
# https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/
kubectl api-resources --sort-by name -o wide | grep -E "^roles"

kubectl create role prod_admin --verb='*' --resource='*' --namespace=prod --dry-run=client -o yaml | grep -v creationTimestamp > role_prod_admin.yml

cat <<- EOF | tee role_prod_admin.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prod_admin
  namespace: prod
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
EOF

kubectl apply -f role_prod_admin.yml

kubectl create role prod_view --verb=get,list,watch --resource='*' --namespace=prod --dry-run=client -o yaml | grep -v creationTimestamp > role_prod_view.yml

cat <<- EOF | tee role_prod_view.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prod_view
  namespace: prod
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - get
  - list
  - watch
EOF

kubectl apply -f role_prod_view.yml
```

Apply role bindings
```bash
kubectl api-resources --sort-by name -o wide | grep -E "^rolebindings"

kubectl create rolebinding prod_admin --role=prod_admin --user=prod_admin --namespace=prod --dry-run=client -o yaml | grep -v creationTimestamp | kubectl apply -f -
kubectl create rolebinding prod_view --role=prod_view --user=prod_view --namespace=prod --dry-run=client -o yaml | grep -v creationTimestamp | kubectl apply -f -
```

Examine roles and bindings
```bash
kubectl describe role {prod_admin,prod_view} -n prod
Name:         prod_admin
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *          []                 []              [*]


Name:         prod_view
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *          []                 []              [get list watch]

kubectl describe rolebindings {prod_admin,prod_view} -n prod
Name:         prod_admin
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  prod_admin
Subjects:
  Kind  Name        Namespace
  ----  ----        ---------
  User  prod_admin


Name:         prod_view
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  prod_view
Subjects:
  Kind  Name       Namespace
  ----  ----       ---------
  User  prod_view
```

4.3 Create a serviceAccount sa-namespace-admin. Grant full rights to namespace default. Create context, authorize using the created sa, check accesses.

Create service account

```bash
kubectl create serviceaccount sa-namespace-admin
serviceaccount/sa-namespace-admin created
```

Find `token` in secrets
```bash
kubectl get serviceaccounts sa-namespace-admin -o yaml | grep secrets -A 1
secrets:
- name: sa-namespace-admin-token-gn6fc

kubectl get secrets sa-namespace-admin-token-gn6fc -o yaml | grep token:
token: ZXlKaGJHY2lPaUpT...Q25rUVE=

TOKEN=$(kubectl get secrets sa-namespace-admin-token-gn6fc -o yaml | grep token: | awk -F' ' '{ print $2 }')

TOKEN=$(printf $TOKEN | base64 -d)
eyJhb...GciO
```

Create role
```bash
kubectl create role sa-admin-role --verb="*" --resource="*" --dry-run=client -o yaml

cat <<- EOF | tee sa-role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sa-admin-role
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
EOF
```

Bind our role with serviceaccount
```
kubectl create rolebinding sa-admin-rolebind --role=sa-admin-role --serviceaccount=default:sa-namespace-admin
```

Set context for our service account
```bash
kubectl config set-context sa-namespace-admin --cluster=minikube --namespace=default --user=sa-namespace-admin
```

Examine actions in our service account:

```bash
kubectl config set-credentials sa-namespace-admin --token=$TOKEN
kubectl config use-context sa-namespace-admin

list=(pods deployments replicasets ingresses roles)

for i in ${list[@]}; do \
  printf "  How I create %s? -> %s\n" "$i" "$(kubectl auth can-i create $i --namespace default)"; done
  How I create pods? -> yes
  How I create deployments? -> yes
  How I create replicasets? -> yes
  How I create ingresses? -> yes
  How I create roles? -> yes

for i in ${list[@]}; do \
  printf "  How I create %s? -> %s\n" "$i" "$(kubectl auth can-i create $i --namespace kube-system)"; done
  How I create pods? -> no
  How I create deployments? -> no
  How I create replicasets? -> no
  How I create ingresses? -> no
  How I create roles? -> no
```


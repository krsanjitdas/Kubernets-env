Following the guide will demonstrate how to create both a temporary token-based authentication and a permanent authentication.
For Temporary token
Step1-a: Create Namespace
#kubectl create namespace my-namespace
Step 1: Create a Service Account
Create a service account that will be associated with the token.

#kubectl create serviceaccount my-service-account --namespace my-namespace



Step 2: Assign Roles and Permissions
Attach the necessary RBAC permissions to the service account. For example, to allow the service account to read all pods in the namespace:
Use dry-run command to create yaml file
#kubectl create role pod-reader --verb=get,list,watch --resource=pods --namespace=my-namespace --dry-run=client -o yaml > pod-reader.yaml
Create a Role:

kind: Role
metadata:
  creationTimestamp: null
  name: pod-reader
  namespace: my-namespace
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch

Execute following command and modify accordingly as follow yaml file
 #kubectl create rolebinding pod-reader-binding --role=pod-reader --user=my-service-account --namespace=my-namespace --dry-run=client -o yaml > pod-reader-binding.yaml
Bind the Role to the Service Account:

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: pod-reader-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: my-namespace



Apply the YAML files:

#kubectl apply -f role.yaml
#kubectl apply -f rolebinding.yaml


Step 3: Generate a Token for the Service Account
Use the kubectl create token command (available from Kubernetes 1.24) to generate a token for the service account.

#kubectl create token my-service-account --namespace my-namespace

This command will output the token. Save it securely, as it will be needed to authenticate.

Step 4: Configure a Kubernetes Context
Set Up the Context Using kubectl config: Use the token and retrieved details to create a new context:



#kubectl config set-credentials my-user --token=<your-token>
#kubectl config set-context my-context \
  --cluster=<your-cluster-name> \
  --user=my-user \
  --namespace=my-namespace
#kubectl config use-context my-context


Step 5: Test the Context

If the service account has sufficient permissions, you should see a list of pods in the namespace. If not, adjust the RBAC policies accordingly.
And execute the following command to see the output is “yes” or “no” 

#kubectl auth can-i list pods --as=system:serviceaccount:my-namespace:my-service-account --namespace=my-namespace

Just copy the config file from ~/.kube to the another user’s home directory as follow

#Cp ~/.kube/config /home/iimc/.kube/

#Chown -R iimc:iimc /home/iimc/.kube

Just remove all other context except my-context then execute following command to set default context

#Kubectl config get-contexts 
See the available context 

#Kubectl config use-context my-context
Set default context 

Verify that the context is correctly configured by running:

kubectl get pods


If you experience with following error message thats mean the role and rollbinding need to check for permission related issue.

Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:my-network:sanjit-sa" cannot list resource "pods" in API group "" in the namespace "my-network"



Other information 

Starting with Kubernetes 1.24 and later versions (including 1.31.0), service account tokens are no longer automatically associated with secrets. Instead, Kubernetes uses Projected Service Account Tokens, which are short-lived tokens mounted directly into Pods. This change enhances security by avoiding long-lived secrets.

Why the Change?
Improved Security: Short-lived tokens reduce the impact of token theft.
Better Rotation: Tokens are automatically rotated without manual intervention.
Deprecation of Secret-Based Tokens: Secrets containing tokens (.secrets[0].name) are no longer created by default.


For permanent token 
1.a Create Namespace

#kubectl create namespace my-namespace

1.b Create a Service Account

#kubectl create sa my-legacy-service-account -n my-namespace --dry-run=client -o yaml  > my-legacy-service-account.yaml
Create a service account as usual:

apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-legacy-service-account
  namespace: my-namespace


#kubectl apply -f service-account.yaml


2. Manually Create a Secret for the Service Account
Since Kubernetes no longer auto-creates a secret for the service account, you need to create one manually:
#kubectl create secret generic my-legacy-token -n my-namespace --dry-run=client -o yaml

apiVersion: v1
kind: Secret
metadata:
  name: my-legacy-token
  namespace: my-namespace
  annotations:
    kubernetes.io/service-account.name: my-legacy-service-account
type: kubernetes.io/service-account-token




#kubectl apply -f secret.yaml


3. Verify the Secret is Bound to the Service Account
Check that the secret is created and linked to the service account:
#kubectl get secrets -n my-namespace
Look for the secret you created (my-legacy-token) and verify its association:
#kubectl describe secret my-legacy-token -n my-namespace
You should see an entry like this in the output:
Annotations:
  kubernetes.io/service-account.name: my-legacy-service-account


Use dry-run command to create yaml file
#kubectl create role pod-reader --verb=get,list,watch --resource=pods --namespace=my-namespace --dry-run=client -o yaml > pod-reader.yaml
Create a Role:

kind: Role
metadata:
  creationTimestamp: null
  name: pod-reader
  namespace: my-namespace
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch

Execute following command and modify accordingly as follow yaml file
 #kubectl create rolebinding pod-reader-binding --role=pod-reader --user=my-service-account --namespace=my-namespace --dry-run=client -o yaml > pod-reader-binding.yaml
Bind the Role to the Service Account:

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: pod-reader-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: my-legacy-service-account
  namespace: my-namespace




Apply the YAML files:
#kubectl apply -f role.yaml
#kubectl apply -f rolebinding.yaml



4. Retrieve the Token
Extract the token from the secret:
#token=$(kubectl get secret my-legacy-token -n default -o jsonpath='{.data.token}' | base64 --decode)

Save the token in variable
Step 5: Configure a Kubernetes Context
Set Up the Context Using kubectl config: Use the token and retrieved details to create a new context:
bash
Copy code

#kubectl config set-credentials my-user --token=<your-token>
#kubectl config set-context my-context \
  --cluster=<your-cluster-name> \
  --user=my-user \
  --namespace=my-namespace

#kubectl config use-context my-context

And execute the following command to see the output is “yes” or “no” 

#kubectl auth can-i list pods --as=system:serviceaccount:my-namespace:my-legacy-service-account --namespace=my-namespace


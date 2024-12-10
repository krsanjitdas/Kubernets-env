Following configuration work on > v1.24.0 

### **Step 1: Create a Service Account**

Create a service account that will be associated with the token.

bash  
Copy code  
`kubectl create serviceaccount my-service-account --namespace my-namespace`

---

### **Step 2: Assign Roles and Permissions**

Attach the necessary RBAC permissions to the service account. For example, to allow the service account to read all pods in the namespace:

Use dry-run command to create yaml file

kubectl create role `pod-reader` \--verb=get,list,watch \--resource=pods \--namespace=`my-namespace` \--dry-run=client \-o yaml \> pod-reader.yaml

Create a Role:  
yaml  
Copy code  
`apiVersion: rbac.authorization.k8s.io/v1`  
`kind: Role`  
`metadata:`  
  `name: pod-reader`  
  `namespace: my-namespace`  
`rules:`  
`- apiGroups: [""]`  
  `resources: ["pods"]`  
  `verbs: ["get", "list", "watch"]`

Execute following command and modify accordingly as follow yaml file

 kubectl create rolebinding pod-reader-binding \--role=pod-reader \--user=my-service-account \--namespace=my-namespace \--dry-run=client \-o yaml \> pod-reader-binding.yaml

Bind the Role to the Service Account:  
yaml  
Copy code  
`apiVersion: rbac.authorization.k8s.io/v1`  
`kind: RoleBinding`  
`metadata:`  
  `name: pod-reader-binding`  
  `namespace: my-namespace`  
`subjects:`  
`- kind: ServiceAccount  #### replace user to ServiceAccount`  
  `name: my-service-account`  
  `namespace: my-namespace`  
`roleRef:`  
  `kind: Role`    
  `name: pod-reader`  
  `apiGroup: rbac.authorization.k8s.io`

1. 

Apply the YAML files:

bash  
Copy code  
`kubectl apply -f role.yaml`  
`kubectl apply -f rolebinding.yaml`

---

### **Step 3: Generate a Token for the Service Account**

Use the `kubectl create token` command (available from Kubernetes 1.24) to generate a token for the service account.

bash  
Copy code  
`kubectl create token my-service-account --namespace my-namespace`

This command will output the token. Save it securely, as it will be needed to authenticate.

### 

### 

### **Step 4: Configure a Kubernetes Context**

**Set Up the Context Using `kubectl config`:** Use the token and retrieved details to create a new context:  
bash  
Copy code

`kubectl config set-credentials my-user --token=<your-token>`  
`kubectl config set-context my-context \`  
  `--cluster=<your-cluster-name> \`  
  `--user=my-user \`  
  `--namespace=my-namespace`  
`kubectl config use-context my-context`

1. 

---

### **Step 5: Test the Context**

If the service account has sufficient permissions, you should see a list of pods in the namespace. If not, adjust the RBAC policies accordingly.

And execute the following command to see the output is “yes” or “no” 

kubectl auth can-i list pods \--as=system:serviceaccount:my-namespace:my-service-account \--namespace=my-namespace

Just copy the config file from \~/.kube to the another user’s home directory as follow

Cp \~/.kube/config /home/iimc/.kube/

Chown \-R iimc:iimc /home/iimc/.kube

Just remove all other context except my-context then execute following command to set default context

Kubectl config get-contexts   
See the available context 

Kubectl config use-context my-context  
Set default context 

Verify that the context is correctly configured by running:

bash  
Copy code  
`kubectl get pods`

If you experience with following error message thats mean the role and rollbinding need to check for permission related issue.

Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:my-network:sanjit-sa" cannot list resource "pods" in API group "" in the namespace "my-network"

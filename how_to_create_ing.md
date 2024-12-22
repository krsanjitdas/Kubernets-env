How to Deploy Ingress to a Deployment
Ingress can be deployed with or without an external load balancer.

With an External Load Balancer: The ingress configuration integrates directly with the load balancer to manage external traffic.
Without an External Load Balancer: The NGINX Ingress Controller can be exposed using one of the following methods:
*) NodePort
*) Host Networking
*) MetalLB (recommended for production in bare-metal setups)

*) NodePort:
Let’s start with NodePort:
In this method, we use the worker node's IP address along with a specific port to access the application running in the container.

Step 1: We will create a .yaml file for the deployment. The following command is very helpful for generating a basic .yaml file, which you can edit afterward as needed.

#kubectl create deployment --image=nginx web-nodeport --port=80 --dry-run=client -o yaml > web-nodeport.yaml

After generating the .yaml file, verify its contents using the vim command.

vim web-nodeport.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-nodeport
  name: web-nodeport
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-nodeport
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-nodeport
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}

Execute the following command to create the deployment:
#kubectl create -f web-nodeport.yaml

Check the deployment 
#kubectl get pods

Step 2: To expose the deployment, we need to generate a YAML file for the service. Please note that the service will not be created until the deployment is successfully applied.

#kubectl expose deployment web-nodeport --port 80 --target-port 80  --type NodePort --dry-run=client -o yaml > web-nodeport-svc.yaml

After generating the .yaml file, Add the line for nodePort in the web-nodeport-svc.yaml file.

#vim web-nodeport-svc.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: web-nodeport
  name: web-nodeport-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30001       # add the line
  selector:
    app: web-nodeport
  type: NodePort
status:
  loadBalancer: {}

Execute the following command to create the service for the respective deployment:
#kubectl create -f web-nodeport-svc.yaml

Check the service 
#kubectl get svc

Step 3: Finally, create the ingress, where the backend service will be web-nodeport-svc, and web-nodeport-svc will be linked to the web-nodeport deployment.

Let’s generate the ingress YAML file using the following command, and then add ingressClassName: nginx in that yaml file. 

#kubectl create ingress web-ing --rule="nginxnode.iimcal.ac.in/=web-nodeport:80" --dry-run=client -o yaml >  web-ing.yaml

#vim  web-ing.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: web-ing
spec:
  ingressClassName: nginx                  ## Add the line
  rules:
  - host: nginxnode.iimcal.ac.in
    http:
      paths:
      - backend:
          service:
            name: web-nodeport-svc
            port:
              number: 80
        path: /
        pathType: Exact
status:
  loadBalancer: {}

Execute the following command to create the ingress for the respective service:
#kubectl create -f web-ing.yaml

Check the service 
#kubectl get ing

Now, check the application from a remote browser by typing nginxnode.iimcal.ac.in. Please note that the hostname must be available in your DNS entry or the hosts file. In this case, we tested by editing the hosts file.

Add the following line to the /etc/hosts file. The IP 172.16.1.114 is one of the worker nodes.

172.16.1.114 nginxnode.iimcal.ac.in
Step5.
Now test the service
curl nginxnode.iimcal.ac.in:30001

==============================NodePort===========================

*) Host Networking:
Let’s start with Host Networking:
In this method, the pod uses the node's network interfaces and IP addresses directly, which allows it to be accessible via the node's IP and ports.

Step 1: We will create a .yaml file for the deployment. The following command is very helpful for generating a basic .yaml file, which you can edit afterward as needed.


#kubectl create deployment --image=nginx web-hostport --port=80 --dry-run=client -o yaml > web-hostport.yaml

After generating the .yaml file, update it with hostNetwork and dnsPolicy in the yaml file using the vim command.

#vim web-hostport.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-hostport
  name: web-hostport
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-hostport
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-hostport
    spec:
      hostNetwork: true  # use for host networking
      dnsPolicy: ClusterFirstWithHostNet  # use for host networking
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}

Execute the following command to create the deployment:
#kubectl create -f web-hostport.yaml

Check the deployment 
#kubectl get pods

Step 2: The same way to expose the deployment, we need to generate a YAML file for the service. In this file you don't modify the svc file.
 

kubectl expose deployment web-hostport --port 80 --target-port 80  --dry-run=client -o yaml > web-host-svc.yaml

Edit the file web-host-svc.yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: web-host-svc
  name: web-host-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web-hostport
status:
  loadBalancer: {}


Execute the following command to create the service for the respective deployment:
#kubectl create –f web-host-svc.yaml

Check the service 
#kubectl get svc        # check the ip address it will get same if of node ip

Step 3: Finally, create the Ingress, where the backend service will be web-host-svc, and web-host-svc will be linked to the web-hostport deployment. The IP address for the service will be the same as the Node IP.

Let’s generate the ingress YAML file using the following command, and then add ingressClassName: nginx in that yaml file. 

Now create the Ingress file

kubectl create ingress web-ing --rule="nginxnode.iimcal.ac.in/=web-host-svc:80" --dry-run=client -o yaml >  web-ing.yaml

#vim web-ing.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: web-ing
spec:
  ingressClassName: nginx    # add the line
  rules:
  - host: nginxport.iimcal.ac.in
    http:
      paths:
      - backend:
          service:
            name: web-host-svc
            port:
              number: 80
        path: /
        pathType: Exact
status:
  loadBalancer: {}

Execute the following command to create the ingress for the respective service:
#kubectl create -f web-ing.yaml

Check the service 
#kubectl get ing


#Kubectl get svc                # check the ip address it will get same if of node ip

Now, check the application from a remote browser by typing nginxnode.iimcal.ac.in. Please note that the hostname must be available in your DNS entry or the hosts file. In this case, we tested by editing the hosts file.

Add the following line to the /etc/hosts file. The IP 172.16.1.114 is one of the worker nodes

Now add the following line in /etc/host file.
172.16.1.114 nginxport.iimcal.ac.in

Curl nginxport.iimcal.ac.in

==============================MetalLB===========================

Now we will see ingress with MetalLB. MetalLB is a load balancer for Kubernetes clusters, designed specifically for bare-metal (non-cloud) environments.It provides a way to expose Kubernetes services to external traffic with a load balancer feature.

MetalLB (recommended for production in bare-metal setups).

Step 1: asusal we will create a .yaml file for the deployment. The following command is very helpful for generating a basic .yaml file, which you can edit afterward as needed.

# kubectl create deployment web-load --image=nginx --port 80 --dry-run=client -o yaml > web-load.yaml
Edit the file  web-load.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-load
  name: web-load
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-load
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-load
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}

Execute the following command to create the deployment:
#kubectl create -f web-load.yaml

Check the deployment 
#kubectl get pods

Step 2: The same way to expose the deployment, we need to generate a YAML file for the service. 

kubectl expose deployment web-load --port 80 --target-port 80 --type LoadBalancer --dry-run=client -o yaml > web-load-svc.yaml

#vim web-load-svc.yaml

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: web-load-svc
  name: web-load-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web-load
  type: LoadBalancer
status:
  loadBalancer: {}

Execute the following command to create the service for the respective deployment:
#kubectl create –f web-load-svc.yaml

Check the service 
#kubectl get svc        # check the ip address it will get same if of node ip

Step 3: Finally, create the Ingress, where the backend service will be web-load-svc, and web-load-svc will be linked to the web-load deployment. The IP address for the service will get the same range ip address as the network of the Node IP.

Let’s generate the ingress YAML file using the following command, and then add ingressClassName: nginx in that yaml file. 

kubectl create ingress web-ing --rule="nginxnode.iimcal.ac.in/=web-load-svc:80" --dry-run=client -o yaml >  web-ing.yaml
Edit the file

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: web-ing
spec:
  ingressClassName: nginx    # add the line
  rules:
  - host: nginxload.iimcal.ac.in
    http:
      paths:
      - backend:
          service:
            name: web-load-svc
            port:
              number: 80
        path: /
        pathType: Exact
status:
  loadBalancer: {}

Execute the following command to create the ingress for the respective service:
#kubectl create -f web-ing.yaml

Check the service 
#kubectl get ing

root@master-node:/home/iimc/ing-with-nodeport# kubectl get svc
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
web-load            LoadBalancer   10.110.35.173   172.16.1.232   80:32161/TCP   2m12s

Now, check the application from a remote browser by typing nginxload.iimcal.ac.in. Please note that the hostname must be available in your DNS entry or the hosts file. In this case, we tested by editing the hosts file.

Add the following line to the /etc/hosts file. The IP 172.16.1.114 is one of the worker nodes

Now add the following line in /etc/host file.
172.16.1.232 nginxload.iimcal.ac.in

Curl nginxload.iimcal.ac.in




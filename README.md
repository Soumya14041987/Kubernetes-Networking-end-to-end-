# Cilium CLI install on Linux :-
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Hubble installation :-

HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}

# With Cilium successfully installed on your lab machine, proceed to install version 1.15.0 of Cilium.
cilium install --version 1.15.0 --wait

# Cilium Status
Execute cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium             Running: 1
                       cilium-operator    Running: 1
Cluster Pods:          2/2 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.15.0@sha256:9cfd6a0a3a964780e73a11159f93cc363e616f7d9783608f62af6cfdf3759619: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.15.0@sha256:e26ecd316e742e4c8aa1e302ba8b577c2d37d114583d6c4cdd2b638493546a79: 1

# To further test the Cilium installation which Cilium CLI command can be executed?

cilium connectivity test

# To observe network flows and events from a Hubble Server, which command can be executed?
hubble observe

# Your Security manager has come to you with a checklist of requirements for the network. For compliance, due to the sensitive data, the cluster must not allow for ANY Ingress or Egress as a default.

Communications are only allowed to be opened to pods where absolutely necessary. The current application consists of a website, a database, and, also for compliance reasons, a backup system.

The database must be able to accept communication from both the website and the backup system.
The website needs to be able to accept ingress on port 80.
The backup server needs to be able to egress to the NFS.
Identify your resources
The pods and the namespaces have been labeled as follows:

Website:
role: website

Database:
role: database

Backup System:
role: backup-system


# To meet the Security requirements, create a default policy for the database, website, and backup-system namespaces to deny all egress as a default. Follow the below naming convention for policies:
  Policy name: default-{namespace-name}-network-policy


# For namespace - website:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: default-website-network-policy
 namespace: website
spec:
 podSelector: {}
 policyTypes:
 - Egress
 egress: []

For namespace - database:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: default-database-network-policy
 namespace: database
spec:
 podSelector: {}
 policyTypes:
 - Egress
 egress: []

For namespace - backup-system:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: default-backup-network-policy
 namespace: backup-system
spec:
 podSelector: {}
 policyTypes:
 - Egress
 egress: []

#Under wesbite namespace now if I try to made curl 
 k exec -n website website -- curl kodekloud.com
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   167  100   167    0     0   2586      0 -<html>- --:--:-- --:--:--     0
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
-:--:-- --:--:-- --:--:--  2609

#Since the website will need to get data from the database, create a network policy with the following details:

Policy Name: allow-website-ingress-to-database
Policy action: Allow database pod to get ingress communication from website pod on port 3306

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-website-ingress-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      role: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          role: website
    - podSelector:
        matchLabels:
          role: website
    ports:
    - protocol: TCP
      port: 3306
  policyTypes:
  - Ingress

# The backup server will also need access to the database pod in order to take backups. Create another policy similar to the website to database ingress that allows the backup server to communicate with database with the following details:

Policy Name: allow-backup-ingress-to-database
Policy action: Allow database pod to get ingress communication from backup pod on port 3306

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backup-ingress-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: backup-system
        - podSelector:
            matchLabels:
              role: backup-system
      ports:
        - protocol: TCP
          port: 3306
  policyTypes:
    - Ingress


  cat default-backup-network-policy.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-backup-network-policy
  namespace: backup-system
spec:
  podSelector:
    matchLabels:
      app: backup-system
  egress:
    - to:
        - ipBlock:
            cidr: 10.1.2.3/32
      ports:
        - protocol: TCP
          port: 2049
  policyTypes:
    - Egress

# Create a service for the website based on the deployment in the website namespace. Use the following details for creating service:

Name of the service: website-service
Selectors: role: website
Port name: http
Port and TargetPort: 80


Ans :- apiVersion: v1
kind: Service
metadata:
 name: website-service
 namespace: website
spec:
 selector:
   role: website
 ports:
 - name: http
   protocol: TCP
   port: 80
   targetPort: 80


# Create a new pod in the website namespace and attach an interactive shell. Retrieve the value of the WEBSITE_SERVICE_SERVICE_HOST environment variable and select the correct from the options ?

Launch a new pod and attach interactive shell using the following command.
kubectl run test --image nginx -n website

kubectl exec -n website test -it -- bash
Print the WEBSITE_SERVICE_SERVICE_HOST environment variable
echo $WEBSITE_SERVICE_SERVICE_HOST

# In Kubernetes, a DNS record is automatically created for each service, following a specific format that includes the service name, namespace, and a default domain which is typically svc.cluster.local. This DNS naming convention ensures that each service can be uniquely identified across the cluster.

For a service named database-service in the database namespace, the full DNS name would be:

Note :-
In Kubernetes, a DNS record is automatically created for each service, following a specific format that includes the service name, namespace, and a default domain which is typically svc.cluster.local. This DNS naming convention ensures that each service can be uniquely identified across the cluster.

For a service named database-service in the database namespace, the full DNS name would be:

database-service.database.svc.cluster.local

The above can also be used with short DNS name as follows:
database-service.database


# Launch the pod website-deployment-{POD_HASH} in the website namespace and get the DNS nameserver and the search domains.Save the search details and nameserver details in file /root/dns.txt on controlplane.

Attach the interactive shell to the website-deployment pod as follows:
kubectl exec -n website website-deployment-65b6bf9ff6-9n8kq -it -- bash
DNS details will be available at file cat /etc/resolv.conf
root@website-deployment-65b6bf9ff6-9n8kq:/# cat /etc/resolv.conf 
search website.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
exit the container and save the details to the dns.txt file as follows:
echo "search website.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10 " > dns.txt


# Cilium is the underlying CNI being used on this cluster.When the Cilium CLI is installed locally, can we check the status of the network by issuing cilium-health status? #Using the Cilium CLI installed locally, get the current state of Cilium. Are there issues currently within Cilium that could be preventing network connectivity?

False

# What is not an additional step to troubleshoot the CNI?

Investugating the logs of kube-proxy

# It appears that the website pods in the website namespace cannot connect to the database pod in the database namespace. Looks like it just hangs when trying to connect. Open another terminal and execute a port forward to one of the website pods on ports 8000:5000 and execute the test-network.sh script. 
Investigate the network policies in the website namespace and delete a network policy that is no longer needed.

Port forward the deployment using following:
 kubectl port-forward -n website deployments/website-deployment 8000:5000
----
Forwarding from 127.0.0.1:8000 -> 5000
Forwarding from [::1]:8000 -> 5000

# We have also provided a test-network.sh script to check the connection. Open another terminal and execute the script.
bash test-network.sh 
curl: (28) Operation timed out after 3000 milliseconds with 0 bytes received
# As you can see the request is timed out, let's check if there are any network policies in website or database namespaces that is denying the connection.
kubectl describe netpol -n database
# We have a netpol that is denying all the egress; it might causing this issue. Delete the netpol and test the connection using the script again.
kubectl delete netpol -n website default-deny-egress

--- 
bash test-network.sh


# It looks like the connection is still off between the website and the database.Have a look at the database namespace and fully investigate the network policies. Edit the network policy that is not allowing ingress from the website.

1. Network policy in the database namespace is as follows:
kubectl describe netpol -n database 
Name:         allow-website-ingress-to-database
Namespace:    database
Created on:   2024-05-01 15:54:50 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     role=database
  Allowing ingress traffic:
    To Port: 3306/TCP
    From:
      NamespaceSelector: role=website
    From:
      PodSelector: role=website
  Not affecting egress traffic
  Policy Types: Ingress
2. As you can see, the above policy allows ingress from pods from namespace with label role:website . Let's check the labels of the website and database resources:

k get ns --show-labels | grep website
website           Active   36m   app=website,kubernetes.io/metadata.name=website
---
k get pods -n website --show-labels 
NAME                                  READY   STATUS    RESTARTS   AGE   LABELS
website-deployment-6d98cbcc89-dchlz   1/1     Running   0          37m   app=website,pod-template-hash=6d98cbcc89
---
k get pods -n database --show-labels 
NAME                                   READY   STATUS    RESTARTS   AGE   LABELS
database-deployment-866789c4b4-jm4wv   1/1     Running   0          38m   app=database,pod-template-hash=866789c4b4

3. As you can see, the labels are mismatched. Change the labels as follows and apply the policy again using the following manifest.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: allow-website-ingress-to-database
 namespace: database
spec:
 podSelector:
   matchLabels:
     app: database
 ingress:
 - from:
   - namespaceSelector:
       matchLabels:
         app: website
   - podSelector:
       matchLabels:
         app: website
   ports:
     - protocol: TCP
       port: 3306
 policyTypes:
 - Ingress


# There still seems to be a connection timeout between the website and the database. But at this point, we have validated that the network policies are once again configured properly. Since we have validated that egress from the website is working in previous steps using the test-network.sh script, let's assume that there a problem with the database. Remember that the database is also using a service. Investigate the service mysql-service in the database namespace and make the necessary changes. Note: Test the connection using bash test-network.sh

# There still seems to be a connection timeout between the website and the database. But at this point, we have validated that the network policies are once again configured properly. Since we have validated that egress from the website is working in previous steps using the test-network.sh script, let's assume that there a problem with the database. Remember that the database is also using a service. Investigate the service mysql-service in the database namespace and make the necessary changes.

Note: Test the connection using bash test-network.sh

# Inspect the mysql-service, the yaml definition of the service is as follows:

apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: database
spec:
  clusterIP: None
  clusterIPs:
  - None
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: mysql
    port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    role: database
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
# As you can see, there is a label mismatch for selector, change the labels to app: database. Change the manifest as follows and redeploy the service.

apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: database
spec:
  clusterIP: None
  clusterIPs:
  - None
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: mysql
    port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: database
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

# The database is now reachable internally, but we are still getting a timeout error. The only difference is we are now getting an error when trying to connect Unknown MySQL server host 'mysql.database.svc.cluster.local'. This hostname is actually configured as an environment variable in your website-deployment manifest as MYSQL_HOST, but it cannot be resolved. This is, in fact, incorrect. Modify this environment variable to the correct value in the website deployment.


# As the internal networking is now resolved and the website can connect to the database, we need to resolve the connection issue for external networking. Users are complaining that the URL they used to use to test the site is not responding. We originally configured this to be set up with a nodePort service type. Knowing this, we need to investigate the service used for the website. Make necessary changes to resolve the website nodeport type service website-service-nodeport.

Currently, the NodePort service is configured as follows:

apiVersion: v1
kind: Service
metadata:
  name: website-service-nodeport
  namespace: website
spec:
  type: NodePort
  ports:
    - name: http
      port: 5000
      targetPort: 5000
      nodePort: 30000
  selector:
    role: website
# But you can see the labels are misconfigured; the website pods use the labels app: website.

Change the manifest as follows and deploy the service.

apiVersion: v1
kind: Service
metadata:
  name: website-service-nodeport
  namespace: website
spec:
  type: NodePort
  ports:
    - name: http
      port: 5000
      targetPort: 5000
      nodePort: 30000
  selector:
    app: website

















                       

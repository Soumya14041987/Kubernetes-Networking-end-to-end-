# To get started, add the Cert-manager repo on helm in your terminal. Use the following artifactory link to get the instructions.

https://artifacthub.io/packages/helm/cert-manager/cert-manager?modal=install

Ans :- helm repo add cert-manager https://charts.jetstack.io

# Create a namespace called cert-manager for Cert-manager.
 k create ns cert-manager

# Install Cert-manager in the newly created cert-manager namespace using Helm, including all necessary Custom Resource Definitions (CRDs).
helm install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true

** Plaese note that installCRD flag is depreacted ,so use crds.enabled instead .

# Create an initial staging Issuer resource named letsencrypt-staging in the website namespace. Use the Staging server URL: https://acme-staging-v02.api.letsencrypt.org/directory. 
Email address: kodekloud-user@gmail.com
Name the secret key: letsencrypt-staging
and configure an http01 challenge on the Ingress website-ingress in the website namespace.

apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: website
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: kodekloud-user@gmail.com
    privateKeySecretRef:
          name: letsencrypt-staging
    solvers:
    - http01:
            ingress:
              name: website-ingress

 k apply -f letsencrypt-staging.yaml 
issuer.cert-manager.io/letsencrypt-staging created

# Now that we have a staging issuer, update the Ingress website-ingress to use the staging issuer for the domain companyx-website.com, and specify secret name as web-ssl.Ensure to add the necessary annotations and configure the TLS section with the host name and secret name.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: website-ingress
  namespace: website
  annotations:
    cert-manager.io/issuer: letsencrypt-staging
spec:
  tls:
  - hosts:
    - companyx-website.com
    secretName: web-ssl
  defaultBackend:
    service:
      name: website-service-nodeport
      port:
        name: http
  rules:
  - host: companyx-website.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: website-service-nodeport
            port:
              name: http
#  Our Ingress has now been updated with Cert-manager, and a certificate has been created as a secret. We can also check the status of our certificate by describing the Certificate custom resource      
              
kubectl describe certificate -n website


# If the certificate was successfully created, proceed to create a new Production Issuer. Create a production Issuer resource named letsencrypt-production in the website namespace.

Use the Staging server URL: https://acme-v02.api.letsencrypt.org/directory
Email address: kodekloud-user@gmail.com
Name the secret key: letsencrypt-production
and configure an http01 challenge on the Ingress website-ingress in the website namespace.

Syntax :-
If the certificate was successfully created, proceed to create a new Production Issuer.

Create a production Issuer resource named letsencrypt-production in the website namespace.

Use the Staging server URL: https://acme-v02.api.letsencrypt.org/directory
Email address: kodekloud-user@gmail.com
Name the secret key: letsencrypt-production
and configure an http01 challenge on the Ingress website-ingress in the website namespace.

Syntax :-
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-production
  namespace: website
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: kodekloud-user@gmail.com
    privateKeySecretRef:
          name: letsencrypt-production
    solvers:
    - http01:
            ingress:
              name: website-ingress

k apply -f letsencrypt-production.yaml 
issuer.cert-manager.io/letsencrypt-staging configured


Now that we have a tested staging and a have an production issuer, update the Ingress website-ingress to use the production issuer for the domain companyx-website.com.
Ensure to add the necessary annotations.

kubectl annotate ingress -n website website-ingress cert-manager.io/issuer=letsencrypt-production --overwrite

What is the purpose of Let’s Encrypt?

What protocol does Let's Encrypt primarily use to automate the process of obtaining and renewing SSL/TLS certificates?

ACME 

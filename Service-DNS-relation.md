

In Kubernetes, when you create a Service to expose one or more pods, Kubernetes automatically creates environment variables in each pod to hold information about the service. These environment variables allow pods to dynamically discover and communicate with each other without needing to hard-code IP addresses or service names.

How Environment Variables for Services are Created ?


When a service is created in Kubernetes, it gets assigned a fixed IP address (the ClusterIP) and a DNS name. Kubernetes then generates environment variables based on this service for all running pods in the namespace, allowing these pods to reference the service easily. Here’s how these environment variables are typically structured:

Service Name in Uppercase: The name of the service, converted to uppercase.

Service Name Suffixes:

_SERVICE_HOST: Contains the IP address of the service.

_SERVICE_PORT: Contains the port number that the service exposes.


For example, if you have a service named website-service in your Kubernetes cluster, Kubernetes generates the following environment variables for each pod in the same namespace:

WEBSITE_SERVICE_SERVICE_HOST: The IP address of the website-service.

WEBSITE_SERVICE_SERVICE_PORT: The port number on which website-service is listening.

Example

Let's assume you have a service named database-service in the database namespace, exposing a MySQL database. You might access this service from another pod in the same namespace using environment variables like this:

mysql -h $DATABASE_SERVICE_SERVICE_HOST -P $DATABASE_SERVICE_SERVICE_PORT -u user -p password




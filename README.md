# platomics-TA

1. Immutable Deployment for backend
   Description: Created an immutable backend deployment using the provided base manifest. Configured it so that only /tmp is writable.

- Deploy the manifest:
    kubectl apply -f backend-deployment.yaml
    #output
     deployment.apps/backend created

- Verify the deployment and its pod:
    kubectl get deployments -n project-plato
    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    backend   1/1     1            1           110s

    kubectl get pods -n project-plato
    NAME                      READY   STATUS    RESTARTS   AGE
    backend-99bccf8d9-f6bjh   1/1     Running   0          4m17s

    kubectl exec -it backend-7657567c46-bdnql -n project-plato -- sh
    # now Check /tmp directory is writable, but other directories are not
    # Inside the pod, run the following:
      touch /tmp/testfile       
      touch /root/testfile      

2. Deployments for 'db1' and 'db2'
   Description: Created two deployments db1 and db2 using the nginx:1.16.1-alpine image. Exposed db1 on port 6379 and db2 on port 5432.

 - Deploy the manifests:
   kubectl apply -f db-deployments.yaml
   # output
   deployment.apps/db1 created
   deployment.apps/db2 created

   kubectl get deployments -n project-plato
   backend   1/1     1            1           14m
   db1       1/1     1            1           75s
   db2       1/1     1            1           75s

   # Verify the db1 and db2 pods
   kubectl get pods -n project-plato -l app=db
   backend-99bccf8d9-f6bjh   1/1     Running   0          15m
   db1-84bbbcc84d-db2jp      1/1     Running   0          2m9s
   db2-6ff65497b4-khg5b      1/1     Running   0          2m9s

3. Services for db1 and db2
   Documentation for Verification:

  - Deploy the services:
    kubectl apply -f db-services.yaml
    service/db1 created
    service/db2 created

   I created custom nginx.conf files for both db1 and db2 to ensure that each service listens on the required ports (6379 for db1 and 5432 for db2). These configurations are managed via ConfigMaps in Kubernetes.
   Referenece: "nginx-db1.conf and nginx-db2.conf"

   - Create ConfigMap for db1 & db2:
     kubectl create configmap nginx-db1-config --from-file=nginx-db1.conf -n project-plato
     kubectl create configmap nginx-db2-config --from-file=nginx-db2.conf -n project-plato
   
   These ConfigMaps store the nginx.conf files and make them available to the db1 and db2 deployments.

   # Verify the ConfigMap Mount:
     For db1: kubectl exec -it db1-84bbbcc84d-db2jp  -n project-plato -- cat /etc/nginx/nginx.conf
     For db2: kubectl exec -it db2-6ff65497b4-khg5b -n project-plato -- cat /etc/nginx/nginx.conf
   # Ensure that db1 is exposed on port 6379 and db2 on port 5432
  - Verify the services:    
    kubectl get services -n project-plato
    NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    db1    ClusterIP   10.99.98.68      <none>        6379/TCP   99s
    db2    ClusterIP   10.104.105.130   <none>        5432/TCP   99s

4. Liveness and Readiness Probes
   Description: Configured LivenessProbe and ReadinessProbe for the backend deployment.
   Update backend-deployment.yaml to include the probes:

   Documentation for Verification: 

 - Check the probe functionality:
   # Check the backend pod status
   kubectl get pods -n project-plato -l app=backend

   # Describe the backend pod to check probe statuses
   kubectl describe pod backend-99bccf8d9-f6bjh -n project-plato

   # Simulate the readiness check by verifying db1 connectivity
   kubectl exec -it backend-99bccf8d9-f6bjh -n project-plato -- nc -zv db1 6379

5. NetworkPolicy
   Description: Created a NetworkPolicy np-backend that allows the backend pod to connect only to db1 on port 6379 and db2 on port 5432.
   
   # Verify the NetworkPolicy
   kubectl get networkpolicy np-backend -n project-plato

   # Check connectivity from backend to db1 (should succeed)
   kubectl exec -it backend-99bccf8d9-f6bjh -n project-plato -- nc -zv db1 6379

   # Check connectivity from backend to db2 (should succeed)
   kubectl exec -it backend-99bccf8d9-f6bjh -n project-plato -- nc -zv db2 5432

   # Attempt to connect to an unauthorized pod (should fail)
   kubectl exec -it backend-99bccf8d9-f6bjh -n project-plato -- nc -zv nginx-pod 8080

6. Secret for DB2 Deployment
   Description: Created a Kubernetes Secret with a username and password, then injected it into the db2 deployment.

 - Verification:
   # Verify the secret exists
   kubectl get secret db2-secret -n project-plato

   # Exec into the db2 pod and check for the secret
   kubectl exec -it db2-6ff65497b4-khg5b -n project-plato -- sh

   # Inside the pod, verify that the environment variables are set
   echo $DB_USERNAME    # Should output the decoded username
   echo $DB_PASSWORD    # Should output the decoded password


Bonus Task: 
  Helm Deployment of Postgres and kube-prometheus-stack
  Description: Deployed Postgres and kube-prometheus-stack using Helm. Configured Prometheus to scrape Postgres metrics.

  # Add the Helm repositories
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update

  # Deploy Postgres
  helm install my-postgres bitnami/postgresql --namespace monitoring --set primary.metrics.enabled=true

  # Deploy kube-prometheus-stack
  helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring

  # Verify the deployments
  kubectl get pods -n monitoring

  # Port-forward to access the Prometheus UI
  kubectl port-forward svc/prometheus-kube-prometheus-stack-prometheus 9090:9090 -n monitoring

  # In the Prometheus UI, check if Postgres metrics are being scraped
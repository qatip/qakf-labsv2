# Lab 4 - Networking
## 4.0 Day 2 Code sync
1. If running this lab on Day 2 of the course, artifacts expected from Day1 will not be present.
2. This section will bring your current K8S (Kubernetes) configuration to 'continue from end of day 1'
3. Create two new namespaces: `development` and `production`. 

<details><summary>show command</summary>
<p>

```bash
kubectl create namespace development
kubectl create ns production
```

</p>
</details>
<br/>
4. Ceate a file named `index.html` in your home directory. Its contents should be similar to the following, but feel free to customise the "welcome" message to suit yourself:

```html
<html><body><h1>Welcome to my home page!</h1></body></html>'
```

<details><summary>show command</summary>
<p>

```bash
echo '<html><body><h1>Welcome to my home page!</h1></body></html>' > ~/index.html
```

</p>
</details>
<br/>
5. Create a 'settings' ConfigMap in each namespace.

<details><summary>show command</summary>
<p>

```bash
kubectl create configmap settings --from-literal=colour=purple --namespace development
kubectl create configmap settings --from-literal=colour=green --namespace production
```

</p>
</details>
<br/>

6. Create a `secret` called `secrets` from a literal value with a key of `password` in the dev and prod namespaces, with different values for each password.

<details><summary>show command</summary>
<p>

```bash
kubectl create secret generic secrets --from-literal password=MySecretPhrase --namespace development
kubectl create secret generic secrets --from-literal password=ReallySecret --namespace production
```

</p>
</details>
<br/>
7. Create lab4frontend.yaml file referencing the configmap and secret volumes ready for later deployment

<details><summary>show YAML</summary>
<p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - image: public.ecr.aws/qa-wfl/qa-wfl/qakf/sfe:v1
        name: sfe
        env:
        - name: COLOUR
          valueFrom:
            configMapKeyRef:
              name: settings
              key: colour        
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: secret-volume
          mountPath: /data
      volumes:
      - name: secret-volume
        secret:
          secretName: secrets
```

</p>
</details>
<br/>

## 4.1 Explore CoreDNS
Objective: The goal of this lab is to demonstrate a 2-tier solution hosted in Kubernetes, a Frontend and a Backend. The Frontend will be a containerized web service with a default webpage that identifies its Pod name, namespace, configmap details, secrets details also the response it receives when communicating with its backend component. The Backend is a containerized application that simply returns it's version details when requested.

Within the Kubernetes cluster, the Frontend containers will not know which containers are running the backend app. The Backend containers must therefore be positioned behind a service with a fixed internal IP address, known as a Cluster IP, which must be registered in DNS. This service registration is automatic and namespace specific. The Frontend containers can therefore query DNS for service address resolution and obtain the service IP address for communication to a backend container within their namespace.

External traffic needs to be able to access the Frontend. Therefore, Frontend containers are also put behind a service which 
generates a service Cluster IP address. This is an internal IP, inaccessible to external clients. In front of this we could create a NodePort per node which would allow external traffic to be directed to the External IP address of a specific node in our cluster. A much better alternative would be an Ingress Controller. This allows a single External IP address to be used for all incoming traffic to all frontend offerings in the cluster. The Frontend to which access is required is identified by the creation of an Ingress behind the Ingress Controller. The format of URL requested by external clients identifies the particular Ingress required and thus the Ingress Controller directs the traffic to the appropriate service which directs it to an appropriate pod. This is similar to context switch load balancing in traditional networking.        

![Lab 4.1 final result](../diagrams/lab_4_coredns.png)
1. Lets start by creating the Backend tier and lets have 2 versions. Version 1 will be the current production version whilst Version 2 will be in devlopment.

2. Create a backend deployment in each namespace using the image `public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe` v2 in development and v1 in production.

<details><summary>show commands</summary>
<p>

```bash
kubectl create deploy backend --image=public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe:v1 -n production 
kubectl create deploy backend --image=public.ecr.aws/qa-wfl/qa-wfl/qakf/sbe:v2 -n development
```

</p>
</details>
<br/>

2. Expose them both as `clusterIP` services on `port` 80 with a `target-port` of 8080, giving them both a `name` of `backend`.

<details><summary>show commands</summary>
<p>

```bash
kubectl expose deployment backend --port 80 --target-port 8080 --name backend --namespace production 
kubectl expose deployment backend --port 80 --target-port 8080 --name backend -n development
```

</p>
</details>
<br/>

3. As the services are created they are automatically registered with DNS. So there will be a registration of 'backend' in each each namespace. A pod in the namepace can therefore request the service, by name, and DNS will identify the Cluster IP of the service in that namespace. Lets test this by creating a pod that we can connect to, in each workspace, and inside the container of which we can generate a service request. This will highlight how internal pods can easily find other internal pods without needing to know their IP addresses. So, we are going to use the `busybox` image to interact with DNS using nslookups.

4. Create a pod named `nettools` in both the `development` and `production` namespaces. Use the `busybox` image. We'll need it to run a `command` of `sleep infinity` otherwise it will immediately transition to a `completed` state.

<details><summary>show commands</summary>
<p>

```bash
kubectl run nettools --image=busybox -n production --command sleep infinity
kubectl run nettools --image=busybox -n development --command sleep infinity
```

</p>
</details>
<br/>

5. Use `kubectl exec` to execute the command `nslookup backend` on the two pods in the two different namespaces.

<details><summary>show commands</summary>
<p>

```bash
kubectl exec -it nettools -n production -- nslookup backend
kubectl exec -it nettools -n development -- nslookup backend
```

</p>
</details>
<br/>


Example output:

```
Server:         10.96.0.10
Address:        10.96.0.10:53

** server can't find backend.cluster.local: NXDOMAIN

Name:   backend.development.svc.cluster.local
Address: 10.107.29.209

** server can't find backend.svc.cluster.local: NXDOMAIN

** server can't find backend.cluster.local: NXDOMAIN


** server can't find backend.svc.cluster.local: NXDOMAIN

command terminated with exit code 1
```
<br/>

Note that CoreDNS tried a lot of variations of the name `backend` but one of the first was for `backend.development.svc.clsuter.local` in this case. When run against the production namespace, it'll look there. This is to illustrate that CoreDNS "knows" which namespace a pod is running in and returns the appropriate lookup.

5. So, given that assurance, we can now deploy our Frontend. The image used is coded to look for the backend and therefore this should 'just work'.

<details><summary>show command</summary>
<p>

```bash
kubectl apply -n production -f lab4frontend.yaml
kubectl apply -n development -f lab4frontend.yaml
```

</p>
</details>
<br/>

6. Get the new pods' details either from the dev or the prod namespaces. It doesn't matter which. Ask for `wide` output so you can see their IP addresses.

<details><summary>show command</summary>
<p>

```bash
kubectl -n production get pods --output wide
```

</p>
</details>
<br/>

Example output (modified):

```
NAME                          READY   STATUS    RESTARTS   AGE   IP               NODE
frontend-5b6dcf74cb-kvvvn     1/1     Running   0          14m   192.168.29.154   k8s-worker-0   lab4backend-676f7c57f-26cjm   1/1     Running   0          78m   192.168.230.25   k8s-worker-1   nettools                      1/1     Running   0          7m    192.168.230.30   k8s-worker-1
```

<br/>

7. **cURL** one of the frontend pods' IP addresses, targetting port 8080. You should see a v2 message in the dev namespace and a v1 message in production.

## 4.2 Expose the frontends

8. Expose both of the frontends in the two different namespaces. Expose them both as `clusterIP` services on `port` 80 with a `target-port` of 8080, giving them both a `name` of `frontend`.

<details><summary>show commands</summary>
<p>

```bash
kubectl expose deployment frontend --port 80 --target-port 8080 --name frontend --namespace production 
kubectl expose deployment frontend --port 80 --target-port 8080 --name frontend -n development
```

</p>
</details>
<br/>

9. Check that you can reach them both by finding their IP addresses and **cURL**ing them.

<details><summary>show command</summary>
<p>

```bash
kubectl get svc -A
curl dev-frontend-service-ip
curl prod-frontend-service-ip
```

</p>
</details>
<br/>

## 4.3 Install an ingress controller

![Lab 4.2 final result](../diagrams/lab_4_ingress.png)

Because we've exposed the back- and front-end services as `clusterIP`s, we can't currently reach them from "outside" the cluster (the host machine). That's fine for the backend services, but the frontend needs to be reachable. If we were running in a cloud, we could expose the two frontends with two load balancer services, or use the cloud vendor's ingress controller. We'll use a couple of ingress rules behind a single Nginx ingress controller.

10. Run the following command to install an Nginx Ingress Controller. A whole bunch of resources will be created. Helm is a package manager for Kubernetes, which we haven't covered yet, but we will, in the very next module.

```bash
helm install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

11. Check that the ingress service is running. Get a list of all services in all namespaces.

<details><summary>show command</summary>
<p>

```bash
kubectl get services --all-namespaces
```

</p>
</details>
<br/>

Example output (modified):

```
NAMESPACE       NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
development     backend                              ClusterIP      10.102.60.108    <none>        80/TCP                       14m
production      backend                              ClusterIP      10.105.142.21    <none>        80/TCP                       19m
ingress-nginx   ingress-nginx-controller             LoadBalancer   10.107.57.60     <pending>     80:31886/TCP,443:30765/TCP   72s
ingress-nginx   ingress-nginx-controller-admission   ClusterIP      10.104.181.164   <none>        443/TCP                      72s
```

<br/>

12. Make a note of the nodePort number of the `ingress-nginx-controller` service. Browse to yourIp:nodePort where yourIP is the External address of your Controller node. You should get a 404 File Not Found error because, whilst we are hitting the Ingress controller, we haven't configured any ingresses, rules indicating which service we are trying to communicate with behind the controller. We're going to sort that out now.



13. Create an ingress rule for the dev frontend using nip.io in the dev namespace. You might want to call the file `devingress.yaml`.

<details><summary>show command</summary>
<p>

devingress.yaml:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dev-ingress
  namespace: development
spec:
  ingressClassName: nginx
  rules:
  - host: dev.172.17.1.10.nip.io # make sure this IP address matches your node's IP address
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend
            port:
              number: 80
```

</p>
</details>
<br/>

14. Create the ingress.

<details><summary>show command</summary>
<p>

```bash
kubectl create -f devingress.yaml
```

</p>
</details>
<br/>

15. Point your web browser at *dev*.**your-ip***.nip.io*:**ingress-nodePort**, for example in this instance it's `dev.172.17.1.10.nip.io:31886` 

16. Create another ingress for the production namespace. It will be very similar to the devingress.yaml, but you need to make sure you change all the bits that need to change.

<details><summary>show command</summary>
<p>

prodingress.yaml:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prod-ingress    #change this from dev
  namespace: production #change this from dev
spec:
  ingressClassName: nginx
  rules:
  - host: prod.172.17.1.10.nip.io #change this from dev
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend
            port:
              number: 80
```

</p>
</details>
<br/>

17. Apply and test the prodingress.yaml file, similar to steps 15 and 16 above (just replace "prod" with "dev" in those commands).

18. That's it, you're done! Let your instructor know that you've finished the lab.

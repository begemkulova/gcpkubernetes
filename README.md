# gcpkubernetes by Bekaiym Egemkulova 
> [!NOTE]
> #### 👽 the app works perfectly with the HTTPS, yahoo! :>
> </detalis> 

## Awesome Cats Deployment - Application Guide
In the guidance below, you will see my example and my process of deploying this application. 
All Kubernetes yaml files can be found in the folder "kubernetes-yamls". 

## Prerequisites
### Repository Cloning

Clone the backend and frontend repositories locally:

```sh
$ git clone https://github.com/AntTechLabs/awesome_cats_backend.git
$ git clone https://github.com/AntTechLabs/awesome_cats_frontend.git
```

## Database Setup

### Google Cloud SQL

1. **Create a Cloud SQL Instance**
   - Navigate to the Google Cloud Console, search for Cloud SQL, and create a PostgreSQL instance.
   - Save the instance ID, password, and IP address.
   - Create or select a database named `awesome_cats_db`.
   - Configure network settings to allow connections from your Kubernetes cluster. Add the Network in the Connection section.

2. **Install PostgreSQL Locally**
   - Download and install the PostgreSQL version matching your Cloud SQL instance from [here](https://www.postgresql.org/download/).

3. **Connect to Cloud SQL via CLI**
   ```sh
   $ gcloud sql connect sql-instance-bekaiym --user=postgres 
   postgres=> \c awesome_cats_db
   ```

4. **Create Tables**
   ```sql
   CREATE TABLE login (
       id serial PRIMARY KEY,
       email text UNIQUE NOT NULL,
       hash VARCHAR(100) NOT NULL
   );

   CREATE TABLE users (
       id serial PRIMARY KEY,
       name VARCHAR(100),
       email text UNIQUE NOT NULL,
       score BIGINT DEFAULT 0,
       joined TIMESTAMP NOT NULL
   );
   ```

## Kubernetes Secret

Create a secret for your database credentials:

```sh
kubectl create secret generic db-secret \
  --from-literal=PGHOST='YOUR_INSTANCE_IP' \
  --from-literal=PGUSER='postgres' \
  --from-literal=PGDATABASE='awesome_cats_db' \
  --from-literal=PGPASSWORD='YOUR_PASSWORD'
```

## Dockerization

### Backend

1. **Create Dockerfile for Backend**
   ```Dockerfile
   FROM node:14 
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3000
   CMD ["node", "server.js"]
   ```

2. **Build and Push Docker Image**
   ```sh
   $ docker build -t gcr.io/YOUR_PROJECT_ID/awesome-cats-backend:v1 .
   $ docker push gcr.io/YOUR_PROJECT_ID/awesome-cats-backend:v1
   ```

### Frontend

1. **Create Dockerfile for Frontend**
   ```Dockerfile
   # Build
   FROM node:14 AS builder
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   RUN npm run build

   # Prod
   FROM nginx:alpine
   COPY --from=builder /app/build /usr/share/nginx/html  
   EXPOSE 80
   CMD ["nginx", "-g", "daemon off;"]
   ```

2. **Build and Push Docker Image**
   ```sh
   $ docker build -t gcr.io/YOUR_PROJECT_ID/awesome-cats-frontend:v1 .
   $ docker push gcr.io/YOUR_PROJECT_ID/awesome-cats-frontend:v1
   ```

## Kubernetes Deployments

### Backend Deployment and Service

1. **Create Backend Deployment**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: awesome-cats-backend
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: awesome-cats-backend
     template:
       metadata:
         labels:
           app: awesome-cats-backend
       spec:
         containers:
           - name: backend
             image: gcr.io/YOUR_PROJECT_ID/awesome-cats-backend:v1
             ports:
               - containerPort: 3000
             env:
               - name: PGHOST
                 valueFrom:
                   secretKeyRef:
                     name: db-secret
                     key: PGHOST
               - name: PGUSER
                 valueFrom:
                   secretKeyRef:
                     name: db-secret
                     key: PGUSER
               - name: PGPASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: db-secret
                     key: PGPASSWORD
               - name: PGDATABASE
                 valueFrom:
                   secretKeyRef:
                     name: db-secret
                     key: PGDATABASE
   ```

2. **Create Backend Service**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: backend-service
   spec:
     selector:
       app: awesome-cats-backend
     ports:
       - protocol: TCP
         port: 3000
         targetPort: 3000
     type: ClusterIP
   ```

3. **Apply Backend Deployment and Service**
   ```sh
   $ kubectl apply -f backend-deployment.yaml
   $ kubectl apply -f backend-service.yaml
   ```

### Frontend Deployment and Service

1. **Create Frontend Deployment**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: awesome-cats-frontend
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: awesome-cats-frontend
     template:
       metadata:
         labels:
           app: awesome-cats-frontend
       spec:
         containers:
           - name: frontend
             image: gcr.io/YOUR_PROJECT_ID/awesome-cats-frontend:v1
             ports:
               - containerPort: 80
   ```

2. **Create Frontend Service**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: frontend-service
   spec:
     selector:
       app: awesome-cats-frontend
     ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     type: ClusterIP
   ```

3. **Apply Frontend Deployment and Service**
   ```sh
   $ kubectl apply -f frontend-deployment.yaml
   $ kubectl apply -f frontend-service.yaml
   ```

## NGINX Ingress Controller and Ingress

1. **Install NGINX Ingress Controller**
   ```sh
   $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   $ helm repo update
   $ kubectl create namespace ingress-nginx
   $ helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx
   ```

2. **Create Ingress Resource**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress
     annotations:
       cert-manager.io/issuer: letsencrypt-prod
       nginx.ingress.kubernetes.io/ssl-redirect: "true"
       kubernetes.io/ingress.class: nginx
       nginx.ingress.kubernetes.io/use-regex: "true"
       nginx.ingress.kubernetes.io/rewrite-target: /$1
   spec:
     tls:
     - hosts:
       - app.bekaiym.biz
       secretName: bekaiym-biz-secret
     rules:
     - host: app.bekaiym.biz
       http:
         paths:
         - path: /?(.*)
           pathType: Prefix
           backend:
             service:
               name: frontend-service
               port:
                 number: 80
         - path: /api/?(.*)
           pathType: Prefix
           backend:
             service:
               name: backend-service
               port:
                 number: 3000
   ```

3. **Apply Ingress Resource**
   ```sh
   $ kubectl apply -f ingress.yaml
   ``` 

4. **Check Ingress Resource**
   ```sh
   $ kubectl get ingress
   ```

## Domain Configuration and SSL Certificate

1. **Configure Domain with Cloud DNS**
   - Set up DNS records to point to your NGINX Ingress Load Balancer IP.

2. **Install Cert-Manager**
   ```sh
   $ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
   ```

3. **Create ClusterIssuer and Certificate**
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       email: YOUR_EMAIL
       server: https://acme-v02.api.letsencrypt.org/directory
       privateKeySecretRef:
         name: letsencrypt-prod
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: bekaiym-biz-cert
   spec:
     secretName: bekaiym-biz-secret
     dnsNames:
       - app.bekaiym.biz
     issuerRef:
       name: letsencrypt-prod
       kind: ClusterIssuer
   ```

4. **Apply ClusterIssuer and Certificate**
   ```sh
   $ kubectl apply -f cluster-issuer.yaml
   $ kubectl apply -f certificate.yaml
   ```

## Verification

1. **Check Resources**
   ```sh
   $ kubectl get pods
   $ kubectl get svc
   $ kubectl get ingress
   ```

2. **Access Application**
   - Navigate to your domain, (in my case, it is `http://app.bekaiym.biz`) and verify your application is accessible.
   - Confirm SSL is working correctly by accessing `https://app.bekaiym.biz`.

## Troubleshooting

- **Check Logs**
  ```sh
  $ kubectl logs <POD_NAME>
  ```

- **Describe Resources**
  ```sh
  $ kubectl describe pod <POD_NAME>
  $ kubectl describe service <SERVICE_NAME>
  $ kubectl describe ingress <INGRESS_NAME>
  ```

---

## Conclusion

Now, you have successfully deployed the Awesome Cats application on GKE with an NGINX Ingress Controller, secure SSL certificates via Cert-Manager, and PostgreSQL as your database. 



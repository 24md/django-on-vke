# Deploy a Django application on Vultr Kubernetes Engine

The following are the prerequisites and the steps to deploy a Django application on the Vultr Kubernetes Engine.

## Prerequisites

* Provision a [Kubernetes cluster](https://my.vultr.com/kubernetes/add/).
* Provision a [Managed PostgreSQL database](https://my.vultr.com/databases/add/).
* Provision an [Object Storage resource](https://my.vultr.com/objectstorage/add/).

Execute all the commands mentioned below in the Django project directory.

## Vultr Object Storage as Static File Backend

Install the required Python packages.

```console
(venv) # pip install django-storages boto3
```

Edit `settings.py` file.

```console
(venv) # nano PROJECT_NAME/settings.py
```

Ensure that you have imported the `os` library in the `settings.py` file.

Add the following content under the static files configuration.

```python
DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"
STATICFILES_STORAGE = "storages.backends.s3boto3.S3StaticStorage"

AWS_S3_REGION_NAME = os.environ.get("OBJECT_STORAGE_REGION")
AWS_S3_ENDPOINT_URL = f"https://{AWS_S3_REGION_NAME}.vultrobjects.com"
AWS_S3_USE_SSL = True
AWS_STORAGE_BUCKET_NAME = os.environ.get("OBJECT_STORAGE_BUCKET_NAME")
AWS_ACCESS_KEY_ID = os.environ.get("OBJECT_STORAGE_ACCESS_KEY")
AWS_SECRET_ACCESS_KEY = os.environ.get("OBJECT_STORAGE_SECRET_KEY")
AWS_DEFAULT_ACL="public-read"
```

Create `.env` file.

```console
(venv) # nano .env
```

Add the following contents.

```file
OBJECT_STORAGE_REGION=
OBJECT_STORAGE_BUCKET_NAME=
OBJECT_STORAGE_SECRET_KEY=
OBJECT_STORAGE_ACCESS_KEY=
```

Export the environment variables.

```console
(venv) # export $(xargs < .env)
```

Apply the settings.

```console
(venv) # python manage.py collectstatic
```

## Vultr Managaed Database as Database Backend

Install the required Python packages.

```console
(venv) # pip install psycopg2-binary
```

Edit `settings.py` file.

```console
(venv) # nano PROJECT_NAME/settings.py
```

Replace the `DATABASES` variable with the following content.

```python
DATABASES = {
    "default": {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get("DATABASE_NAME"),
        'USER': os.environ.get("DATABASE_USER"),
        'PASSWORD': os.environ.get("DATABASE_PASS"),
        'HOST': os.environ.get("DATABASE_HOST"),
        'PORT': os.environ.get("DATABASE_PORT"),
        'OPTIONS': {'sslmode': 'require'}
    }
}
```

Update `.env` file.

```console
(venv) # nano .env
```

Add the following contents.

```file
DATABASE_HOST=
DATABASE_PORT=
DATABASE_USER=
DATABASE_PASS=
DATABASE_NAME=
```

Export the environment variables.

```console
(venv) # export $(xargs < .env)
```

Apply the settings.

```console
(venv) # python manage.py makemigrations
(venv) # python manage.py migrate
(venv) # python manage.py createsuperuser
```

## Build and Push the Docker Image

Install the required Python packages.

```console
(venv) # pip install gunicorn
```

Remove the old SQL lite database file. Edit the `settings.py` file and set the value of `DEBUG` to `False` and ensure that your domain name is present in the `ALLOWED_HOSTS` list.

Create a new `requirements.txt` file.

```console
(venv) # pip freeze > requirements.txt
```

Exclude the Virtual Environment directory.

```console
# echo "venv" > .dockerignore 
```

Create a new `Dockerfile`.

```console
# nano Dockerfile
```

Add the following contents.

```dockerfile
FROM python:3.8

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN pip install --upgrade pip 
COPY ./requirements.txt /app
RUN pip install -r requirements.txt

COPY . /app

EXPOSE 8000

CMD ["gunicorn", "todo_project.wsgi","-b", "0.0.0.0:8000"]
```

Create a new private repo on DockerHub.

Login, Build & Push the image to DockerHub.

```console
# docker login
# docker build -t DOCKERHUB_USERNAME/REPO_NAME .
# docker push DOCKERHUB_USERNAME/REPO_NAME
```

## Prepare the Kubernetes Cluster

Install the `kubectl` CLI tool.

```console
# curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
# sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Add the Vultr Kubernetes Engine Configuration.

```console
# mkdir ~/.kube
# nano ~/.kube/config
```

Download and paste the configuration by going to the Vultr Customer Portal > Kubernetes Cluster page and clicking on the "Download Configuration" button.

Install the `cert-manager` plugin.

```console
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml
```

Install the `ingress-nginx` controller.

```console
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
```

Create the Let's Encrypt `ClusterIssuer`.

```console
# nano ~/le_clusterissuer.yaml
```

Add the following contents.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "mayank2407@gmail.com"
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply the `ClusterIssuer` manifest file.

```console
# kubectl apply -f ~/le_clusterissuer.yaml
```

Create the secret resource for DockerHub authentication.

```console
# kubectl create secret docker-registry regcred --docker-username=DOCKERHUB_USER --docker-password=DOCKERHUB_PASS --docker-email=DOCKERHUB_EMAIL
```

## Deploy the Django Application on Vultr Kubernetes Engine

Map the domain name to the load balancer provisioned by `ingress-nginx` container using an A record.

Create the Django manifest file.

```console
# nano ~/django.yaml
```

Add the following contents.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: django-app
  template:
    metadata:
      labels:
        name: django-app
    spec:
      imagePullSecrets:
        - name: regcred
      containers:
        - name: django
          image: DOCKERHUB_USERNAME/REPO_NAME
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          envFrom:
          - secretRef:
              name: django-secrets

---
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8000
  selector:
    name: django-app

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - secretName: django-tls
      hosts:
        - YOUR_DOMAIN
  rules:
    - host: YOUR_DOMAIN
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-service
                port:
                  number: 80
```

Create the secret resource for fetching environment variables.

```console
# kubectl create secret generic django-secrets --from-env-file=.env
```

Apply the manifest file.

```console
# kubectl apply -f ~/django.yaml
```

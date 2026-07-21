---
name: k8s-gitlab-cicd-nextjs
description: Deploy Next.js applications to Kubernetes using GitLab CI/CD with Kaniko builds. Use when setting up Kubernetes deployment, GitLab CI/CD pipelines, or containerizing Next.js apps.
---

# Kubernetes + GitLab CI/CD for Next.js

Deploy Next.js applications to Kubernetes using GitLab CI/CD with Kaniko for container builds.

## File Structure

```
project/
├── Dockerfile
├── .gitlab-ci.yml
└── k8s/
    ├── app-deployment.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── ingress.yaml
    ├── mongo-deployment.yaml  # If using MongoDB
    └── mongo-pvc.yaml
```

## 1. Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
ARG NEXT_PUBLIC_BASE_PATH
ENV NEXT_PUBLIC_BASE_PATH=$NEXT_PUBLIC_BASE_PATH
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

**Note**: Ensure `next.config.ts` has `output: 'standalone'` for optimized Docker builds.

## 2. GitLab CI/CD (.gitlab-ci.yml)

```yaml
stages:
  - lint
  - build
  - deploy

variables:
  NAMESPACE: utena-${CI_PROJECT_NAME}
  BASE_PATH: ${CI_PROJECT_NAME}
  REGISTRY: ${CI_REGISTRY}
  REGISTRY_IMAGE: ${CI_REGISTRY_IMAGE}
  REGISTRY_USER: ${CI_REGISTRY_USER}
  REGISTRY_PASSWORD: ${CI_REGISTRY_PASSWORD}

.kaniko:
  image:
    name: gcr.io/kaniko-project/executor:v1.23.2-debug
    entrypoint: [""]
  before_script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"${REGISTRY}\":{\"username\":\"${REGISTRY_USER}\",\"password\":\"${REGISTRY_PASSWORD}\"}}}" > /kaniko/.docker/config.json

build:
  extends: .kaniko
  stage: build
  script:
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${REGISTRY_IMAGE}:latest"
      --build-arg NEXT_PUBLIC_BASE_PATH="/${BASE_PATH}"
      --cache=true
  only:
    - main

deploy:
  stage: deploy
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  script:
    - mkdir -p $HOME/.kube
    - echo -e "$KUBE_CONFIG" > $HOME/.kube/config
    - chmod 600 $HOME/.kube/config
    - kubectl create secret docker-registry gitlab-registry --docker-server=$REGISTRY --docker-username=$REGISTRY_USER --docker-password=$REGISTRY_PASSWORD -n $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
    - |
      find k8s -name "*.yaml" -type f | while read file; do
        envsubst < $file > $file.tmp && mv $file.tmp $file
      done
    - kubectl apply -f k8s/
    - kubectl rollout restart deployment/${CI_PROJECT_NAME} -n ${NAMESPACE}
    - kubectl rollout status deployment/${CI_PROJECT_NAME} -n ${NAMESPACE} --timeout=300s
  only:
    - main
```

## 3. Kubernetes Manifests

### app-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${CI_PROJECT_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${CI_PROJECT_NAME}
  template:
    metadata:
      labels:
        app: ${CI_PROJECT_NAME}
    spec:
      imagePullSecrets:
        - name: gitlab-registry
      containers:
        - name: ${CI_PROJECT_NAME}
          image: ${REGISTRY_IMAGE}:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: ${CI_PROJECT_NAME}-config
            - secretRef:
                name: ${CI_PROJECT_NAME}-secret
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: ${CI_PROJECT_NAME}
  namespace: ${NAMESPACE}
spec:
  selector:
    app: ${CI_PROJECT_NAME}
  type: ClusterIP
  ports:
    - port: 3000
      targetPort: 3000
```

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${CI_PROJECT_NAME}-config
  namespace: ${NAMESPACE}
data:
  NEXT_PUBLIC_BASE_PATH: "/${BASE_PATH}"
  # Add non-sensitive environment variables here
```

### secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ${CI_PROJECT_NAME}-secret
  namespace: ${NAMESPACE}
type: Opaque
stringData:
  # Add sensitive environment variables here
  # Values come from GitLab CI/CD Variables
```

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${CI_PROJECT_NAME}
  namespace: ${NAMESPACE}
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - ${HOSTNAME}
  rules:
    - host: ${HOSTNAME}
      http:
        paths:
          - path: /${BASE_PATH}
            pathType: Prefix
            backend:
              service:
                name: ${CI_PROJECT_NAME}
                port:
                  number: 3000
```

## 4. GitLab CI/CD Variables

Configure in **Settings > CI/CD > Variables**:

| Variable | Description | Protected | Masked |
|----------|-------------|-----------|--------|
| `KUBE_CONFIG` | Kubernetes kubeconfig | Yes | Yes |
| `HOSTNAME` | Application hostname | Yes | No |

Add application-specific secrets as needed.

## 5. MongoDB (Optional)

### mongo-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:latest
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: ${NAMESPACE}
spec:
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

### mongo-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  namespace: ${NAMESPACE}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Deploy Flow

```
1. Push to main branch
       ↓
2. Kaniko builds Docker image → GitLab Registry
       ↓
3. envsubst applies CI variables to k8s manifests
       ↓
4. kubectl apply -f k8s/
       ↓
5. kubectl rollout restart (pull new image)
       ↓
6. Application available at https://${HOSTNAME}/${BASE_PATH}
```

## Key Points

- **envsubst**: Replaces `${VARIABLE}` in YAML files with CI/CD variable values
- **Kaniko**: Builds Docker images without Docker daemon (rootless)
- **gitlab-registry secret**: Allows Kubernetes to pull images from GitLab Container Registry
- **Namespace**: Uses `utena-${CI_PROJECT_NAME}` pattern (customize as needed)

## Troubleshooting

```bash
# Check pods
kubectl get pods -n $NAMESPACE

# View logs
kubectl logs -f deployment/$APP_NAME -n $NAMESPACE

# Describe pod for events
kubectl describe pod -l app=$APP_NAME -n $NAMESPACE

# Check registry secret
kubectl get secret gitlab-registry -n $NAMESPACE
```

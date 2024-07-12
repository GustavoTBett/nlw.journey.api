## Projeto: Aplicação Containerizada com Deploy Automatizado em Kubernetes

Este projeto demonstra como criar, versionar, e automatizar o deploy de uma aplicação containerizada usando Docker, GitHub Actions, Kubernetes, Terraform e AWS.

### Sumário
1. [Pré-requisitos](#pré-requisitos)
2. [Estrutura do Projeto](#estrutura-do-projeto)
3. [Criando o Container da Aplicação](#criando-o-container-da-aplicação)
4. [Configurando o GitHub Actions](#configurando-o-github-actions)
5. [Setup do Kubernetes Local](#setup-do-kubernetes-local)
6. [Deploy no AWS com Terraform](#deploy-no-aws-com-terraform)
7. [CI/CD com GitHub Actions](#cicd-com-github-actions)

### Pré-requisitos
- Docker
- Git
- GitHub Account
- Docker Hub Account
- Kubernetes (Minikube ou similar)
- AWS Account
- Terraform

### Estrutura do Projeto

```
.
├── app-journey
|    └── .github
│       └── workflows
│         └── ci-cd.yml
|   └── deployment.yaml
│     └── service.yaml
|   └── src
│     └── app (código da aplicação)
|   └── Dockerfile
├── iac-cross
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── README.md
```

### Criando o Container da Aplicação

1. **Dockerfile**: Crie um arquivo Dockerfile na raiz do projeto.

```Dockerfile
FROM golang:1.22.4-alpine as builder

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download && go mod verify

COPY . .

RUN go build -o /bin/journey ./cmd/journey/journey.go

FROM scratch 

WORKDIR /app

COPY --from=builder /bin/journey .

EXPOSE 8080

ENTRYPOINT [ "./journey" ]
```

2. **Construir e testar o container**:

```sh
docker build -t my-app .
docker run -p 8080:8080 my-app
```

### Configurando o GitHub Actions

```yaml
name: CI

on: 
    push:
        branches:
        - main

jobs:
    build-and-push:
        if: ${{ !contains(github.event.head_commit.message, 'Update tag in values helm') }}
        name: "Build and Push"
        runs-on: ubuntu-latest

        steps:
        - name: Checkout
          uses: actions/checkout@v4

        - name: Setup Go
          uses: actions/setup-go@v5
          with:
            go-version: "1.22.x"
  
        - name: Run test
          run: go test

        - name: Generate SHA
          id: generate_sha
          run: |
            SHA=$(echo $GITHUB_SHA | head -c7)
            echo "sha=$SHA" >> $GITHUB_OUTPUT

        - name : Login to DockerHub
          uses: docker/login-action@v3
          with:
            username: ${{ secrets.DOCKER_USERNAME }}
            password: ${{ secrets.DOCKER_PASSWORD }}

        - name: Build and push
          uses: docker/build-push-action@v6
          with:
            context: .
            push: true
            tags: | 
                gustavotbett/nlw.journey.api:${{ steps.generate_sha.outputs.sha }}
                gustavotbett/nlw.journey.api:latest

        - name: Update image helm
          uses: fjogeleit/yaml-update-action@main
          with:
            branch: release
            targetBranch: main
            createPR: true
            valueFile: 'deploy/values.yaml'
            propertyPath: 'image.tag'
            value: ${{ steps.generate_sha.outputs.sha }}
            commitChange: true
            message: "[skip ci] Update tag in values helm"
```

### Setup do Kubernetes Local

1. **Instalar Minikube**:

```sh
# No Windows
choco install minikube

# No macOS
brew install minikube

# No Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

2. **Iniciar o cluster local**:

```sh
minikube start
```

3. **Configurar o Kubernetes para a aplicação**:

Crie os arquivos `deployment.yaml` e `service.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: journey-deployment
  labels:
    app: journey

spec:
  replicas: 10
  selector:
    matchLabels:
      app: journey
  template:
    metadata:
      labels:
        app: journey
    spec:
      containers:
        - name: api-journey
          image: gustavotbett/nlw.journey.api:aaf0c3e
          env:
            - name: JOURNEY_DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: db-username
            - name: JOURNEY_DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: db-password
            - name: JOURNEY_DATABASE_PORT
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: db-port
            - name: JOURNEY_DATABASE_HOST
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: db-host
            - name: JOURNEY_DATABASE_NAME
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: db-name
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

```yaml
apiVersion: v1
kind: Service

metadata:
  name: journey-service
  labels:
    app: journey

spec:
  selector:
    app: journey
  type: ClusterIP
  ports:
    - name: journey-service
      port: 80
      targetPort: 8080
      protocol: TCP
```

4. **Aplicar os arquivos de configuração**:

```sh
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### Deploy no AWS com Terraform

1. **Configurar o Terraform**: Crie os arquivos `main.tf`, `variables.tf` e `outputs.tf`.

2. **Executar o Terraform**:

```sh
cd terraform
terraform init
terraform apply
```

### CI/CD com GitHub Actions

1. **Adicionar Secrets**: Adicione `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `DOCKER_USERNAME`, e `DOCKER_PASSWORD` no repositório do GitHub (Settings > Secrets).

2. **Configurar o pipeline para deploy contínuo**: Já configurado no arquivo `main.yml`.

Com isso, qualquer commit na branch `main` irá disparar o pipeline do GitHub Actions que constrói e versiona a imagem Docker, publica no Docker Hub, e faz o deploy no Kubernetes local ou no AWS, conforme configurado.

### Conclusão
Este README guia a configuração completa de uma aplicação containerizada, automação de versionamento e deploy contínuo utilizando Docker, GitHub Actions, Kubernetes, Terraform e AWS.

# API Flash Sales - Infraestrutura DevOps

Projeto da atividade de **Criação de Infraestrutura DevOps**. O repositório contém a API fornecida e toda a infraestrutura necessária para executar, testar, observar e escalar a aplicação.

## Arquitetura

- **API Node.js + TypeScript** empacotada em imagem Docker multi-stage.
- **PostgreSQL 16** para persistência de dados.
- **Redis 7** para cache/controle de operações da aplicação.
- **Docker Compose** para ambiente local reproduzível.
- **Kubernetes** com 2 réplicas da API, probes, limites de recursos e HPA.
- **Prometheus** coleta `/metrics` da API.
- **Grafana** usa Prometheus como datasource provisionado automaticamente.
- **GitHub Actions** executa instalação, testes, build TypeScript e build da imagem Docker a cada push/PR na `main`.

## Pré-requisitos

Para o caminho mais simples, basta ter **Docker + Docker Compose**. Para testar Kubernetes, use também `kubectl` e um cluster local como Minikube ou Docker Desktop Kubernetes.

## Executar com Docker Compose

```bash
git clone <URL-DESTE-REPOSITORIO>
cd flash-sales-api
docker compose up --build -d
```

Serviços:

- API: `http://localhost:3000`
- Métricas: `http://localhost:3000/metrics`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3001` (usuário `admin`, senha `admin`)

Para encerrar:

```bash
docker compose down
```

Para remover também os volumes:

```bash
docker compose down -v
```

## Testar a API

Endpoints principais: **POST /checkout e POST /tickets/export**.

Exemplo:

```bash
curl -X POST http://localhost:3000/checkout \
  -H "Content-Type: application/json" \
  -d '{"eventId":"event1","userId":"user1","quantity":2}'
```

> No Flash Sales, o checkout depende da disponibilidade registrada no Redis pela lógica da aplicação. Os testes automatizados validam os cenários de sucesso e indisponibilidade.

## Testes e build local

```bash
npm ci
npm test
npm run build
```

## Kubernetes

Primeiro construa a imagem e disponibilize-a ao cluster. Em Minikube, por exemplo:

```bash
minikube start
eval $(minikube docker-env)
docker build -t flash-sales-api:latest .
```

Edite `k8s/api.yml` e substitua `CHANGE_ME_IMAGE:latest` por `flash-sales-api:latest`. Depois aplique:

```bash
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/config.yml
kubectl apply -f k8s/postgres.yml
kubectl apply -f k8s/redis.yml
kubectl apply -f k8s/api.yml
```

O HPA exige Metrics Server. Em Minikube:

```bash
minikube addons enable metrics-server
kubectl apply -f k8s/hpa.yml
```

Verifique o ambiente:

```bash
kubectl get all -n devops-atividade
kubectl get hpa -n devops-atividade
```

Para acessar a API localmente:

```bash
kubectl port-forward -n devops-atividade service/api 3000:80
```

## CI - GitHub Actions

O workflow `.github/workflows/ci.yml` roda automaticamente em `push` e `pull_request` para `main`. Ele executa:

1. `npm ci`
2. `npm test`
3. `npm run build`
4. `docker build`

Isso impede que alterações que quebrem testes, compilação ou containerização avancem sem detecção.

## Observabilidade

A API já expõe métricas Prometheus em `/metrics`. No Docker Compose, o Prometheus coleta essas métricas a cada 15 segundos e o Grafana já recebe o Prometheus como fonte de dados. Isso permite acompanhar volume de requisições e métricas da aplicação/Node.js.

## Justificativa da arquitetura

**Escalabilidade:** a API é stateless no container e utiliza PostgreSQL/Redis como serviços externos. No Kubernetes há duas réplicas e um HPA configurado entre 2 e 5 réplicas com alvo de 70% de CPU, permitindo expansão horizontal.

**Confiabilidade:** Docker torna o ambiente reproduzível; Compose usa healthchecks para PostgreSQL e Redis; Kubernetes utiliza readiness/liveness probes e requests/limits; o pipeline valida testes, compilação e criação da imagem antes da entrega.

**Segurança:** credenciais não são gravadas no código da aplicação. O `.env` é ignorado pelo Git e o Kubernetes separa configurações públicas em ConfigMap e credenciais em Secret. A imagem final usa somente dependências de produção e executa com o usuário não-root `node`.

**Automação:** o GitHub Actions automatiza a validação do código. Os manifestos e o Compose tratam a infraestrutura como código, tornando a implantação repetível e versionada.

**Observabilidade:** Prometheus + Grafana fornecem coleta e visualização de métricas, ajudando a detectar falhas, gargalos e comportamento anormal.

## Estrutura

```text
.github/workflows/ci.yml    # pipeline CI
k8s/                       # manifestos Kubernetes
monitoring/                # Prometheus e provisionamento Grafana
src/                       # código da API
 tests/                     # testes automatizados
Dockerfile                  # imagem da API
docker-compose.yml          # ambiente local completo
.env.example                # exemplo de configuração
README.md                   # documentação da entrega
```

## Banco de dados

Banco padrão do projeto: `flash_sales`. Em produção, substitua as credenciais de exemplo por segredos fortes e gerenciados fora do repositório.

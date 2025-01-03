# Melhorar a Pipeline de CI/CD com Jenkins

### Deploy Gradual (Canary/Blue-Green) 
### Implemente estratégias de deploy como Canary ou Blue-Green para validar novas versões em uma fração do tráfego antes de liberar para todos os usuários.

Você libera a nova versão (v2) para uma pequena porcentagem do tráfego (ex.: 5%) enquanto o restante continua na versão estável (v1). Conforme as métricas mostram sucesso, aumenta-se progressivamente o tráfego para a nova versão até substituí-la completamente.

Passos para Implementação:
1. Divisão de Tráfego:Configure o balanceador de carga (ex.: ingress ou service mesh como Istio) para direcionar parte do tráfego à nova versão.
2. Pipeline Jenkins:
    * Crie etapas no pipeline para:
        * Buildar a nova imagem Docker e fazer push para o registro.
        * Criar/atualizar um Deployment Canary no Kubernetes (apontando para a nova versão).
    * Monitore as métricas usando ferramentas como Prometheus.
    * Automatize a progressão do Canary ou volte para v1 se os limites de erro forem excedidos.
3. Exemplo de Arquitetura em Kubernetes:Crie dois Deployments:
    * deployment-v1 (versão estável).
    * deployment-canary (versão de teste).
4. Configure o balanceador de carga para gerenciar a divisão de tráfego. Exemplo com NGINX Ingress:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          service:
            name: service-v1
            port:
              number: 80
            weight: 95
      - path: /
        backend:
          service:
            name: service-canary
            port:
              number: 80
            weight: 5
```

## Blue-Green Deployment
Você mantém dois ambientes idênticos, Blue (atual) e Green (novo).

O Green recebe a nova versão e todo o tráfego é redirecionado para ele ao término do deploy.
Se houver falhas, reverte-se ao ambiente Blue.
Passos para Implementação:

Dois Ambientes no Kubernetes:

Crie dois namespaces ou dois sets de Deployments no mesmo namespace:
- namespace-blue
- namespace-green

Cada namespace contém as réplicas da versão correspondente.

### Pipeline Jenkins:

Crie etapas para:
- Buildar a imagem Docker e fazer push.
- Fazer deploy da nova versão no ambiente Green.
- Executar testes no ambiente Green (sanidade, integração, etc.).
- Redirecionar o tráfego para Green usando o balanceador de carga.

Configuração de Service no Kubernetes:
Alterne o seletor do Service para apontar ao novo Deployment (Green):
```
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: app-green
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

### Pipeline Jenkins para Canary/Blue-Green

Segue um exemplo genérico de pipeline para Jenkins em formato Declarative Pipeline:

```
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "seu-repositorio/app:${BUILD_NUMBER}"
    }
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
                sh 'docker push $DOCKER_IMAGE'
            }
        }
        stage('Deploy Canary') {
            steps {
                script {
                    // Aplica YAML do Canary
                    sh 'kubectl apply -f k8s/canary-deployment.yaml'
                }
            }
        }
        stage('Test Canary') {
            steps {
                script {
                    // Valida métricas no Prometheus
                    sh './scripts/check_metrics.sh'
                }
            }
        }
        stage('Promote Canary to Stable') {
            steps {
                script {
                    // Atualiza deploy principal
                    sh 'kubectl apply -f k8s/stable-deployment.yaml'
                }
            }
        }
        stage('Deploy Blue-Green') {
            steps {
                script {
                    // Alterna tráfego para o Green
                    sh 'kubectl apply -f k8s/blue-green-deployment.yaml'
                }
            }
        }
    }
}
```

## Rollback Inteligente:Configure pipelines para monitorar métricas críticas e reverter automaticamente para a versão anterior se elas saírem dos limites aceitáveis.

O Rollback Inteligente automatiza a reversão para uma versão anterior do sistema quando uma nova versão apresenta problemas, baseando-se em métricas e validações definidas previamente. Ele reduz o tempo de indisponibilidade e minimiza os impactos em produção.

Aqui está como implementar o rollback inteligente em sua estrutura com Jenkins e Kubernetes:

Conceito Básico
Métricas de Monitoramento: Define-se um conjunto de métricas críticas, como:

#### Taxa de erros HTTP (ex.: 5xx, 4xx).
- Latência média ou percentil (ex.: P95).
- Taxa de sucesso em transações ou chamadas downstream.
- Triggers de Rollback: São condições que, ao serem atendidas, acionam o rollback:

#### Taxa de erro acima de 5% em 5 minutos.
- Latência maior que 500ms.
- Métricas específicas para seu negócio, como falhas em cálculos de risco.

### Implementação Técnica com Jenkins e Kubernetes
Pipeline em Jenkins
Inclua uma etapa no pipeline para monitorar métricas e acionar rollback se necessário.

Exemplo simplificado de pipeline declarativo:

```
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "seu-repositorio/app:${BUILD_NUMBER}"
        KUBECTL_CMD = "kubectl" // ou configure como variável do ambiente
    }
    stages {
        stage('Deploy') {
            steps {
                // Faz o deploy da nova versão
                sh 'kubectl apply -f k8s/deployment-new.yaml'
            }
        }
        stage('Monitor Metrics') {
            steps {
                script {
                    // Executa script para verificar métricas
                    def isHealthy = sh(script: './scripts/check_metrics.sh', returnStatus: true)
                    if (isHealthy != 0) {
                        error "Métricas falharam. Iniciando rollback."
                    }
                }
            }
        }
        stage('Rollback') {
            when {
                expression { currentBuild.result == 'FAILURE' }
            }
            steps {
                // Reverte para a versão anterior
                sh 'kubectl apply -f k8s/deployment-stable.yaml'
            }
        }
    }
}
```

### Configuração de Monitoramento de Métricas

Com Prometheus e AlertManager

Crie Regras de Alertas: Configure alertas que identificam problemas com a nova versão:

```
groups:
  - name: service_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[1m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Alta taxa de erro detectada."
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m])) > 0.5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Latência alta detectada."
```

#### Webhook para Jenkins:

Configure o AlertManager para enviar notificações para Jenkins ou outro sistema, acionando o rollback automaticamente:

```
receivers:
  - name: 'jenkins'
    webhook_configs:
      - url: 'http://jenkins-url/rollback'
```

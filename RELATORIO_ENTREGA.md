# Relatório Técnico — Tech Challenge Fase 02

**POSTECH — Pós-graduação em Software Engineering**

---

## Informações do Grupo

| Campo | Informação |
|---|---|
| Integrantes | _(adicionar nomes, RMs e usuários Discord)_ |
| Repositório | _(adicionar link do repositório)_ |
| Vídeo | _(adicionar link do vídeo)_ |
| Badge Google Cloud | _(adicionar link do badge — opcional, +10 pts)_ |

---

## 1. Visão Geral da Arquitetura

Migramos o monolito ToggleMaster para 5 microsserviços independentes, cada um com seu próprio banco de dados, implantados no AWS EKS.

| Serviço | Linguagem | Banco de Dados |
|---|---|---|
| auth-service | Go | PostgreSQL (RDS) |
| flag-service | Python | PostgreSQL (RDS) |
| targeting-service | Python | PostgreSQL (RDS) |
| evaluation-service | Go | Redis (ElastiCache) |
| analytics-service | Python | DynamoDB + SQS |

- **auth-service** — responsável pela criação e validação de chaves de API. Toda requisição ao flag-service e targeting-service passa primeiro pelo auth.
- **flag-service** — API REST que gerencia as feature flags (criar, ativar, desativar).
- **targeting-service** — gerencia as regras de segmentação, definindo quais usuários veem quais flags.
- **evaluation-service** — o caminho crítico da aplicação. Avalia em tempo real se uma flag está ativa para um determinado usuário, armazena os resultados em cache no Redis e publica eventos de avaliação no SQS.
- **analytics-service** — consome as mensagens do SQS e grava os eventos de analytics no DynamoDB para rastreamento histórico.

Todo o tráfego externo entra por um único **Nginx Ingress Controller** com roteamento baseado em path:

| Path | Serviço |
|---|---|
| `/auth` | auth-service |
| `/flags` | flag-service |
| `/targeting` | targeting-service |
| `/evaluate` | evaluation-service |
| `/analytics` | analytics-service |

---

## 2. Infraestrutura Cloud

| Recurso | Serviço AWS | Finalidade |
|---|---|---|
| Cluster Kubernetes | EKS (eksctl) | Orquestração dos microsserviços |
| Imagens Docker | ECR (5 repositórios) | Registry privado por microsserviço |
| Banco auth-service | RDS PostgreSQL | Chaves de API e usuários |
| Banco flag-service | RDS PostgreSQL | Feature flags e regras |
| Banco targeting-service | RDS PostgreSQL | Regras de segmentação |
| Cache evaluation-service | ElastiCache Redis | Cache de resultados de avaliação |
| Eventos analytics-service | DynamoDB | Log imutável de eventos |
| Fila de mensagens | SQS | Canal entre evaluation e analytics |

---

## 3. Kubernetes — Recursos por Microsserviço

Cada microsserviço possui os seguintes manifests Kubernetes:

- **Namespace** — isolamento lógico por serviço
- **Deployment** — gerencia os pods, referencia imagem ECR
- **Service** — tipo ClusterIP para comunicação interna
- **Secret** — credenciais e endpoints (base64)
- **ConfigMap** — URLs internas e configurações não sensíveis

Recursos adicionais:

- **Ingress** — roteamento path-based via Nginx (namespace `ingress-nginx`)
- **ExternalName Services** — aliases DNS para roteamento cross-namespace
- **HPA** — autoscaling do evaluation-service por CPU
- **ScaledObject (KEDA)** — autoscaling do analytics-service por profundidade de fila SQS

---

## 4. Desafios Encontrados

### Node group não criado na primeira execução do eksctl

Ao rodar `eksctl create cluster`, a primeira tentativa falhou no meio da execução devido a um conflito no CloudFormation — uma stack de uma tentativa anterior ainda estava sendo deletada em background. A segunda execução criou o control plane, mas não o node group. Foi necessário rodar `eksctl create nodegroup` separadamente. Em seguida, os nodes ficaram com status `NotReady` porque o addon VPC CNI estava ausente — foi necessário instalar manualmente os 4 addons do EKS: `vpc-cni`, `coredns`, `kube-proxy` e `metrics-server`.

### Security group incorreto nas instâncias RDS

As instâncias RDS do flag-service e targeting-service foram criadas com o security group padrão da VPC, que não possuía regra de entrada para a porta 5432 a partir do CIDR da VPC. Os serviços travavam na inicialização por não conseguirem se conectar ao banco de dados. A solução foi adicionar uma regra de entrada para a porta 5432 a partir do CIDR `192.168.0.0/16` no security group correto e atualizar as instâncias RDS do flag-service e targeting-service para utilizá-lo.

### 503 no Ingress — roteamento entre namespaces

O Ingress foi implantado no namespace `ingress-nginx`, mas os serviços estavam em seus próprios namespaces (`auth-service`, `flag-service`, etc.). O Ingress do Kubernetes não consegue rotear diretamente para serviços em namespaces diferentes. A solução foi criar **ExternalName services** dentro do namespace `ingress-nginx`, que funcionam como aliases DNS apontando para os serviços reais nos outros namespaces.

### Campo incorreto no manifesto do HPA

Ao escrever o manifesto do HPA, foi utilizado o campo `averageUtilizationPercentage`, que está incorreto — o campo correto é `averageUtilization`. O Kubernetes rejeitou o manifesto silenciosamente e o HPA nunca funcionou até que o nome do campo fosse identificado e corrigido.

---

## 5. Observações

### Single NAT Gateway

O cluster EKS foi configurado com um único NAT Gateway em vez de um por zona de disponibilidade. Em produção, essa configuração não é recomendada — o NAT Gateway se torna um **ponto único de falha**: se aquela AZ cair, todo o tráfego das subnets privadas perde o acesso à internet. Essa escolha foi feita deliberadamente para reduzir os custos de infraestrutura neste projeto acadêmico, com uma economia de aproximadamente U$65 por mês.

---

## 6. Escalabilidade — HPA vs KEDA

Foram utilizadas duas estratégias de escalabilidade distintas, cada uma escolhida com base na natureza do serviço.

### evaluation-service → HPA (baseado em CPU)

O evaluation-service é uma API síncrona — os usuários a chamam diretamente e aguardam a resposta. Sua carga é diretamente proporcional ao uso de CPU: mais requisições equivalem a mais CPU. O HPA foi configurado para escalar a partir de 70% de utilização de CPU, pois a CPU é um sinal direto e imediato de carga neste tipo de serviço.

### analytics-service → KEDA (profundidade da fila SQS)

O analytics-service é um consumidor assíncrono — ele processa mensagens de uma fila, não requisições HTTP diretas. A CPU seria uma métrica inadequada aqui: se a fila tiver 10.000 mensagens mas o pod estiver ocioso aguardando o próximo lote, a CPU estará em 0% e o HPA nunca escalaria. O KEDA resolve isso monitorando a **fonte real de trabalho** — a profundidade da fila SQS. Quando as mensagens se acumulam, o KEDA escala os pods para processá-las. Quando a fila esvazia, o KEDA escala os pods até **zero**, o que é impossível com o HPA. Zero pods em estado ocioso significa zero custo computacional.

---

## 7. Bancos de Dados — RDS vs ElastiCache vs DynamoDB

### RDS PostgreSQL — 3 instâncias (auth-service, flag-service, targeting-service)

Utilizado para dados relacionais estruturados que exigem consistência, relacionamentos e consultas complexas. O auth-service armazena chaves de API e usuários. O flag-service armazena flags com suas regras. O targeting-service armazena as configurações de segmentação. São as entidades de negócio centrais da aplicação — precisam de transações, chaves estrangeiras e schemas estruturados.

### ElastiCache Redis — evaluation-service

Utilizado como cache para os resultados de avaliação de flags. Quando o evaluation-service verifica se uma flag está ativa para um usuário, ele consulta primeiro o Redis. Se o resultado estiver em cache, retorna instantaneamente sem precisar consultar o PostgreSQL — resposta em sub-milissegundo. O Redis é um armazenamento em memória: os dados vivem na RAM e são perdidos ao reiniciar, mas isso é aceitável para um cache. O objetivo é velocidade, pois esse é o caminho crítico da aplicação.

### DynamoDB — analytics-service

Utilizado para armazenamento de eventos imutáveis em escala. Cada avaliação de flag gera um evento que o analytics-service grava no DynamoDB. Esses eventos nunca são atualizados ou deletados — apenas gravados e posteriormente consultados. O DynamoDB é totalmente serverless (sem instâncias para gerenciar, com escala automática), não possui schema fixo, o que se adapta bem a eventos que podem evoluir com o tempo, e suporta alto volume de escrita com baixo custo.

# Desafio Kubernetes: Infraestrutura Declarativa e Resiliente

Este projeto é a solução para um desafio de Kubernetes, onde é implementada uma infraestrutura declarativa e resiliente para uma API PostgREST integrada a um banco de dados PostgreSQL. Utilizando manifestos YAML e um namespace isolado, foram configurados o armazenamento persistente (PVC), o gerenciamento de credenciais via Secrets, a comunicação interna por Services e o escalonamento automático (HPA).

## Estrutura do Repositório

```text
.
├── k8s/                          
│   ├── 0-namespace.yaml
│   ├── 1-postgres-configmap.yaml
│   ├── 2-postgres-secret.yaml.example #Apenas um modelo de como fazer seu secret, você deve preencher com suas credenciais
│   ├── 3-postgres-pvc.yaml
│   ├── 4-postgres-deployment.yaml
│   ├── 5-postgres-service.yaml
│   ├── 6-postgrest-deployment.yaml
│   ├── 7-postgrest-service.yaml
│   └── 8-hpa.yaml
├── docs/                          
│   ├── reflexoes.md      
│   ├── validacoes-e-evidencias.md 
│   └── imgs/                      
│       ├── evidenciaInfraestrutura.png
│       ├── evidenciaAPI.png
│       └── evidenciaPersistencia.png
└── README.md                      
```
## Arquitetura e Componentes

Seguem os componentes que fazem parte da arquitetura e seus papéis.

* **Namespace:** Isolamento lógico de todos os recursos da aplicação.

* **PersistentVolumeClaim:** Armazenamento persistente para manter os dados mesmo após a recriação dos Pods.

* **ConfigMap & Secret:** Separação entre configurações de ambiente e credenciais sensíveis.

* **Deployments:** Gerenciamento declarativo e autorrecuperação dos Pods, com health checks realizados por liveness e readiness probes.

* **Services:** Comunicação interna por meio de DNS estático e acesso estável aos Pods, além da exposição da API quando necessário.

* **HorizontalPodAutoscaler:** Escalonamento automático das réplicas da API PostgREST com base no consumo de CPU.

## Funcionamento da Infraestrutura

### Banco de Dados (Persistente)
O PostgreSQL é implantado no namespace desafio-kubernetes através de um Deployment com réplica única. O estado dos dados é preservado de forma desacoplada do ciclo de vida dos Pods por meio de um PersistentVolumeClaim (PVC). As credenciais de acesso ficam armazenadas em um Secret e as rotinas de inicialização/tabelas em um ConfigMap. Para garantir que outros componentes encontrem o banco de forma estável mesmo se o Pod for recriado com outro IP, foi exposto o postgres-service (ClusterIP), disponibilizando resolução de nome via DNS interno.

### API REST (Stateless & Escalável)
A API PostgREST consulta e disponibiliza os dados do banco em formato JSON via requisições HTTP. Por ser uma aplicação sem estado, a API pode ser escalada em múltiplas réplicas sem riscos de concorrência ou corrupção de dados. O gerenciamento de saúde dos Pods é feito por probes, o livenessProbe reinicia containers em caso de travamento e o readinessProbe garante que o tráfego só seja direcionado a Pods totalmente inicializados.

### Autoescala e Resiliência (HPA & Self-Healing)
A infraestrutura conta com um HorizontalPodAutoscaler (HPA) vinculado ao Deployment da API. Com base nas métricas coletadas pelo metrics-server, o HPA ajusta dinamicamente a quantidade de réplicas da API entre 1 e 5 de acordo com o consumo de CPU. Em caso de falha de hardware ou exclusão manual de qualquer Pod, a malha do Kubernetes detecta o desvio do estado desejado e recria automaticamente os containers necessários.

## Como Executar o Projeto Localmente

### Pré-requisitos

-   Cluster Kubernetes ativo (Minikube, k3s, k3d ou Docker Desktop).
-   Ferramenta de linha de comando `kubectl` instalada.
-   `metrics-server` habilitado no cluster para a coleta de métricas do
    HPA.

### 1. Confirmar que o cluster local está ativo

Abra o PowerShell e verifique se o nó do Kubernetes está pronto:

``` powershell
kubectl get nodes
```

A saída deve exibir o nó (ex.: `docker-desktop`) com o status `Ready`.

### 2. Clonar o repositório e configurar credenciais

Clone o projeto e entre na pasta raiz:

``` powershell
git clone https://github.com/JosephLMedeiros/Desafio-Kubernetes.git
cd Desafio-Kubernetes
```
Após isso, crie o arquivo de Secret a partir do modelo e configure suas credenciais:
```
Copy-Item "k8s/2-postgres-secret.yaml.example" -Destination "k8s/2-postgres-secret.yaml"
```
### 3. Subir a infraestrutura

Aplique todos os manifestos contidos na pasta `k8s/`:

``` powershell
kubectl apply -f k8s/
```

O Kubernetes irá criar os recursos definidos nos manifestos:

-   Namespace
-   ConfigMap e Secret
-   PersistentVolumeClaim
-   Deployment e Service do PostgreSQL
-   Deployment e Service da API PostgREST
-   HorizontalPodAutoscaler

### 4. Confirmar que os recursos subiram

Verifique o estado dos componentes no namespace:

``` powershell
kubectl get all -n desafio-kubernetes
```

Os Pods do PostgreSQL e da API PostgREST devem aparecer com status
`1/1 Running`.

### 5. Expor e testar a API

Em uma janela do terminal, inicie o redirecionamento de porta para
acessar o Service da API via `localhost`:

``` powershell
kubectl port-forward svc/postgrest-service 3000:3000 -n desafio-kubernetes
```

Mantenha esse terminal aberto e utilize outra janela do PowerShell para
enviar as requisições.

#### Inserir uma nova tarefa (POST)

``` powershell
Invoke-RestMethod -Uri "http://localhost:3000/tarefas" -Method Post -ContentType "application/json" -Body '{"titulo": "Testar Integracao API e Banco"}'
```

#### Consultar tarefas cadastradas (GET)

``` powershell
Invoke-RestMethod -Uri "http://localhost:3000/tarefas"
```

Se o `POST` for aceito e o `GET` retornar os dados cadastrados, a
integração entre a API e o PostgreSQL está funcionando.

### 6. Testar a persistência dos dados

Para verificar se os dados continuam disponíveis após a recriação do Pod
do PostgreSQL:

#### 1. Insira um registro de confirmação

``` powershell
Invoke-RestMethod -Uri "http://localhost:3000/tarefas" -Method Post -ContentType "application/json" -Body '{"titulo": "Teste de Persistencia PVC"}'
```

#### 2. Delete o Pod do PostgreSQL

``` powershell
kubectl delete pod -l app=postgres -n desafio-kubernetes
```

#### 3. Verifique a recriação automática do Pod

``` powershell
kubectl get pods -n desafio-kubernetes
```

O Deployment deverá criar um novo Pod para substituir o anterior.

#### 4. Consulte a API novamente

``` powershell
Invoke-RestMethod -Uri "http://localhost:3000/tarefas"
```

O registro inserido anteriormente deverá continuar presente, comprovando
a persistência dos dados através do PVC.

### 7. Testar o escalonamento automático (HPA)

Em uma janela do terminal, acompanhe o comportamento do
HorizontalPodAutoscaler:

``` powershell
kubectl get hpa -n desafio-kubernetes -w
```

Em outra janela, crie um Pod temporário para gerar requisições contínuas
à API:

``` powershell
kubectl run carga-teste --image=curlimages/curl -n desafio-kubernetes --rm -it -- sh
```

Dentro do Pod, execute:

``` sh
while true; do curl -s http://postgrest-service:3000/tarefas > /dev/null; done
```

Com o aumento do consumo de CPU, o HPA poderá aumentar a quantidade de
réplicas da API conforme os limites configurados.

Para interromper o teste:

``` text
Ctrl+C
exit
```

Após a redução da carga, o HPA poderá reduzir gradualmente a quantidade
de réplicas.

### 8. Limpar o ambiente

Para remover todos os recursos criados pelo projeto:

``` powershell
kubectl delete namespace desafio-kubernetes
```

Isso removerá o Namespace e os recursos associados, incluindo Pods,
Services, Deployments, HPA e PVC.

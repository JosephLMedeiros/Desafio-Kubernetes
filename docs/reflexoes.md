# Respostas às Reflexão Técnicas

### 1. Ao deletar um Pod avulso, ele volta sozinho? Por quê? Isso te diz algo sobre por que raramente criamos Pods diretamente.
Não, um Pod criado de forma avulsa não volta sozinho se for deletado.

Quando um Pod é criado diretamente, ele não possui um Controller (como um Deployment ou ReplicaSet) associado para gerenciar o seu estado desejado, então o cluster não consegue recriá-lo caso ele morra.

Por isso, em ambientes de produção a norma é a utilização de Controllers, já que eles possuem estratégias automatizadas que garantem o funcionamento correto do Reconciliation Loop e habilitam recursos essenciais como atualizações sem interrupção (Rolling Updates), Rollbacks, controle de versão declarativo e escalonamento automático (HPA).

### 2. Qual a diferença entre montar um PVC e um emptyDir? O que aconteceria com os dados em cada caso ao deletar o Pod?
A diferença principal entre eles está na dependência do volume a um Pod e em seus ciclos de vida.

O emptyDir é um diretório temporário criado no próprio nó e seu ciclo de vida está estritamente atrelado ao do Pod correspondente. Dessa forma, caso o Pod seja deletado, reiniciado ou falhe, o emptyDir é removido e todos os dados armazenados nele são permanentemente perdidos.

Já o PersistentVolumeClaim (PVC) é um elemento de armazenamento persistente que funciona de forma totalmente desacoplada dos Pods, com sua existência independendo dos Pods. Assim, quando um ou mais Pods associados ao PVC são deletados, o volume permanece intacto no cluster, permitindo que um novo Pod recriado reanexe esse mesmo armazenamento e acesse os dados  já existentes.

### 3. Ao inspecionar o Secret com -o yaml , o valor aparece "embaralhado". Isso é criptografia de verdade ou apenas codificação? O que isso significa para a segurança real?
O valor exibido no Secret é apenas uma codificação em Base64, e não criptografia. É uma transformação reversível sem necessidade de chave, onde qualquer pessoa com acesso ao YAML pode decodificar o valor para texto limpo com o comando básico ``base64 -d``. Ele serve apenas para padronizar o formato dos dados para evitar problemas de sintaxe no YAML.

Isso significa que Secret no Kubernetes apenas não garante a proteção de dados sensíveis, é necessário em ambientes reais medidas adicionais, como a utilização de criptografia em repouso no etcd, restringir o acesso através de políticas de RBAC e manter essas credenciais em cofres externos, apenas os injetando conforme a necessidade.

### 4. Por que usamos o nome do Service do Postgres na string de conexão, em vez do IP do Pod? O que aconteceria com a conexão se você usasse o IP e o Pod do banco fosse recriado?
Toda vez que um Pod é recriado, normalmente é atribuído a ele um novo IP. Portanto, se a API fosse configurada utilizando diretamente o IP do Pod do banco de dados, quando esse Pod fosse recriado por uma falha, atualização ou manutenção, a conexão deixaria de funcionar, pois o endereço utilizado pela aplicação não apontaria mais para o novo Pod.

Por isso, utilizamos o nome do Service na string de conexão. O Service fornece um nome DNS estável (`postgres-service`) e um IP virtual fixo (ClusterIP), além de encaminhar as requisições para os Pods correspondentes. Assim, mesmo que o Pod do PostgreSQL seja recriado e receba outro IP, a API continua acessando o banco pelo mesmo nome do Service.

### 5. Quantos componentes tiveram que funcionar em conjunto para esse dado sobreviver? (PVC, Deployment, Service, Secret, a API...) O que isso mostra sobre como o Kubernetes coordena as peças?
Foram necessários 6 componentes centrais:
* **Secret e ConfigMap**, que forneceram as credenciais e as configurações/scripts de inicialização.
* **PVC**, que mantém os dados do PostgreSQL persistidos, mesmo que o Pod seja recriado.
* **Deployment do PostgreSQL**, que garante o estado desejado e recria o Pod caso ele falhe.
* **Service do PostgreSQL**, que mantém um ponto de acesso estável por meio do DNS interno.
* **Deployment da API PostgREST**, que mantém a API em execução e permite que ela se conecte ao banco para disponibilizar os dados ao cliente.

Isso mostra como o Kubernetes coordena diversos recursos de forma desacoplada, em que cada componente possui uma responsabilidade específica, mas existem relações entre eles para formar o sistema completo. O Kubernetes monitora continuamente a diferença entre o estado atual e o estado desejado e tenta corrigi-la automaticamente. Dessa forma, se um Pod falhar, ele pode ser recriado sem que os dados persistidos sejam perdidos e a comunicação possa ser restabelecida automaticamente por meio do Service.

### 6. Qual a diferença prática entre liveness e readiness? Por que escalar a API para várias réplicas é seguro, mas escalar o banco desse jeito (com o mesmo PVC) não seria?
A diferença entre livenessProbe e readinessProbe está na ação tomada quando ocorre uma falha. O livenessProbe verifica se a aplicação continua funcionando, se falhar, o Kubernetes reinicia o container. Já o readinessProbe verifica se a aplicação está pronta para receber requisições, se falhar, o Pod é temporariamente removido do tráfego do Service, mas não é reiniciado.

Escalar a API é seguro porque ela é stateless, ou seja, suas réplicas podem processar requisições independentemente. Já o PostgreSQL é stateful e mantém seus dados no volume. Colocar várias réplicas do banco usando o mesmo PVC não é seguro, pois elas poderiam acessar e modificar os mesmos arquivos simultaneamente, causando conflitos e até corrupção dos dados. Para escalar bancos, seria necessário utilizar mecanismos próprios de replicação e volumes separados para cada instância.

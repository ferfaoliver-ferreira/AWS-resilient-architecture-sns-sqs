# Passo a Passo Técnico: Implementação Prática

Abaixo está o detalhamento prático de todas as etapas executadas no Console da AWS para a construção e validação desta arquitetura resiliente.

---

## 🛠️ Fase 1: Configuração do Buffer Durável e Isolamento de Erros (Amazon SQS)

O primeiro passo consiste em preparar o ambiente de filas. Criamos a fila de falhas antes da principal, pois a fila principal precisa referenciar o ARN da DLQ em sua política de redrive.

### 1. Criação da Dead-Letter Queue (DLQ)
1. No console do **Amazon SQS**, clique em **Criar fila**.
2. Mantenha o tipo como **Padrão**.
3. No campo **Nome**, defina como `minha-dlq-lab`.
4. Deixe as demais configurações de ciclo de vida e criptografia como padrão e clique em **Criar fila** no final da página.
5. Copie e reserve o **ARN** gerado para esta fila (ele será necessário no próximo passo).

### 2. Criação da Fila Principal e Vínculo de Redrive
1. Novamente no painel do SQS, clique em **Criar fila** (Tipo: **Padrão**).
2. Defina o **Nome** como `minha-fila-principal-lab`.
3. Configure o campo **Tempo limite de visibilidade** (*Visibility Timeout*) para **30 segundos** (tempo padrão que garante que uma mensagem lida fique invisível para outros consumidores temporariamente).
4. Role a página até a seção **Fila de mensagens mortas (Dead-letter queue)** e marque a opção **Habilitada**.
5. Em **Escolher fila**, selecione **Inserir ARN da fila SQS** e cole o ARN da `minha-dlq-lab` criado anteriormente.
6. No campo **Contagem máxima de recebimentos** (*maxReceiveCount*), defina o valor como **3**. 
   > *Isso significa que se uma mensagem for lida 3 vezes da fila principal e não for explicitamente deletada (confirmando o sucesso), o SQS entenderá que houve uma falha sistêmica crônica e a moverá para a DLQ.*
7. Clique em **Criar fila**.

---

## 📢 Fase 2: Configuração do Barramento de Eventos (Amazon SNS)

Com as filas prontas, criamos o componente responsável por receber as mensagens de um serviço publicador e distribuí-las de forma assíncrona.

### 1. Criação do Tópico SNS
1. Acesse o console do **Amazon SNS** e clique em **Tópicos** no menu lateral.
2. Clique em **Criar tópico**.
3. Selecione o tipo **Padrão** (adequado para cenários de alta vazão e distribuição em massa/fan-out).
4. No campo **Nome**, insira `meu-topico-sns-lab` e clique em **Criar tópico**.

### 2. Criação da Assinatura (Subscription)
1. Dentro da página do tópico criado, vá até a aba **Assinaturas** e clique em **Criar assinatura**.
2. No campo **Protocolo**, escolha **Amazon SQS**.
3. No campo **Endpoint**, insira o ARN exato da fila principal (`minha-fila-principal-lab`).
4. Clique em **Criar assinatura**. 
   > *A partir deste momento, qualquer mensagem publicada no Tópico SNS será automaticamente replicada e entregue como uma nova mensagem dentro da fila SQS principal.*

---

## 🔒 Fase 3: Restrição de Acesso e Segurança (IAM Resource Policy)

Para impedir que agentes maliciosos ou serviços não autorizados injetem dados na nossa fila, editamos a política de acesso baseada em recursos do SQS para aplicar o privilégio mínimo.

1. Volte ao console do **Amazon SQS** e clique na sua fila principal (`minha-fila-principal-lab`).
2. Acesse a aba **Política de acesso** e clique em **Editar**.
3. No editor JSON, remova a permissão genérica padrão e configure uma regra estrita:
   - Defina o efeito como `Allow`.
   - O `Principal` deve ser estritamente limitado ao serviço do SNS (`sns.amazonaws.com`).
   - A `Action` permitida deve ser apenas `sqs:SendMessage`.
   - Adicione uma cláusula de condição (`Condition`) garantindo que o `aws:SourceArn` seja estritamente igual ao ARN do seu `meu-topico-sns-lab`.
4. Clique em **Salvar**.

---

## 🧪 Fase 4: Validação da Arquitetura e Teste de Redrive (DLQ)

A última fase valida se os tempos e limites de mensageria estão se comportando como o esperado no ecossistema distribuído.

### 1. Publicação do Evento via SNS
1. No console do **Amazon SNS**, entre no tópico `meu-topico-sns-lab`.
2. Clique no botão **Publicar mensagem**.
3. No corpo da mensagem, insira um texto estruturado de teste (ex: `{"id_pedido": 1024, "status": "criado"}`).
4. Clique em **Publicar mensagem**.

### 2. Simulação de Falhas Consecutivas e Transbordo para DLQ
1. Retorne ao painel do **Amazon SQS** e selecione a fila `minha-fila-principal-lab`. Note que o contador de **Mensagens disponíveis** agora marca **1**.
2. Clique em **Enviar e receber mensagens** e clique no botão **Sondar mensagens** (*Sondar/Poll for messages*).
3. A mensagem enviada pelo SNS aparecerá na lista. Clique nela para inspecionar os detalhes:
   - Veja que o contador **Contagem de recebimentos** (*Receive count*) marcará **1**.
4. **O teste de resiliência:** Feche os detalhes da mensagem **sem clicar em Excluir**.
5. Aguarde o relógio correr por **30 segundos** (tempo que definimos para o *Visibility Timeout*). Durante esse intervalo, a mensagem fica oculta para simular que um worker está tentando processá-la.
6. Passados os 30 segundos, clique novamente em **Sondar mensagens**. Abra o card. O contador mudará para **Receive count = 2**. Repita o processo mais uma vez para que ele atinja **Receive count = 3**.
7. Após o terceiro ciclo expirar, clique em sondar novamente. 
8. **Resultado esperado:** A mensagem sumirá completamente da fila principal, indicando que ela atingiu o limite de tolerância estabelecido na *Redrive Policy*.
9. Vá até o menu do SQS, abra a fila `minha-dlq-lab` e clique em **Enviar e receber mensagens** ➔ **Sondar mensagens**.
10. A mensagem original estará estacionada com sucesso na DLQ, isolada e pronta para auditoria dos desenvolvedores, sem ter causado gargalos ou travamentos no barramento principal!

# Passo a Passo Técnico: Implementação Prática

Abaixo está o detalhamento prático de todas as etapas executadas no Console da AWS para a construção e validação desta arquitetura resiliente.
<img width="621" height="472" alt="Captura de tela 2026-05-13 222515" src="https://github.com/user-attachments/assets/3f72582f-3f61-4a04-a5e6-c6c11d2a0a02" />

---

## 🛠️ Fase 1: Configuração do Buffer Durável e Isolamento de Erros (Amazon SQS)

O primeiro passo consiste em preparar o ambiente de filas. Criamos a fila de falhas antes da principal, pois a fila principal precisa referenciar o ARN da DLQ em sua política de redrive.

### 1. Criação da Dead-Letter Queue (DLQ)
1. No console do **Amazon SQS**, clique em **Criar fila**.
2. Mantenha o tipo como **Padrão**.
3. No campo **Nome**, defina como `minha-dlq-lab`.
4. Deixe as demais configurações de ciclo de vida e criptografia como padrão e clique em **Criar fila** no final da página.
5. Copie e reserve o **ARN** gerado para esta fila.

![Configuração Inicial da Fila DLQ](images/Captura%20de%20tela%202026-05-13%20213252.png)
*Visualização do painel de criação e definição do tipo de fila padrão.*

![Confirmação de Criação da DLQ](images/Captura%20de%20tela%202026-05-13%20213514.png)
*Fila de mensagens mortas criada com sucesso no console do SQS.*

---

### 2. Criação da Fila Principal e Vínculo de Redrive
1. Novamente no painel do SQS, clique em **Criar fila** (Tipo: **Padrão**).
2. Defina o **Nome** como `minha-fila-principal-lab`.
3. Configure o campo **Tempo limite de visibilidade** (*Visibility Timeout*) para **30 segundos**.

![Configurando Detalhes da Fila Principal](images/Captura%20de%20tela%202026-05-13%20213537.png)
*Definição de nomenclatura e tempos base para a fila principal do sistema.*

4. Role a página até a seção **Fila de mensagens mortas (Dead-letter queue)** e marque a opção **Habilitada**.
5. Em **Escolher fila**, selecione **Inserir ARN da fila SQS** e cole o ARN da `minha-dlq-lab` criado anteriormente.
6. No campo **Contagem máxima de recebimentos** (*maxReceiveCount*), defina o valor como **3**.
7. Clique em **Criar fila**.

![Ativação da Política de Redrive Policy](images/Captura%20de%20tela%202026-05-13%20213635.png)
*Viculação da fila principal com a DLQ e parametrização do limite máximo de 3 tentativas.*

![Filas Criadas e Prontas](images/Captura%20de%20tela%202026-05-13%20213914.png)
*Listagem do SQS exibindo ambas as filas ativas e disponíveis para a arquitetura.*

---

## 📢 Fase 2: Configuração do Barramento de Eventos (Amazon SNS)

Com as filas prontas, criamos o componente responsável por receber as mensagens de um serviço publicador e distribuí-las de forma assíncrona.

### 1. Criação do Tópico SNS
1. Acesse o console do **Amazon SNS** e clique em **Tópicos** no menu lateral.
2. Clique em **Criar tópico**.
3. Selecione o tipo **Padrão** (adequado para cenários de alta vazão e distribuição em massa/fan-out).
4. No campo **Nome**, insira `meu-topico-sns-lab` e clique em **Criar tópico**.

![Criação do Tópico SNS](images/Captura%20de%20tela%202026-05-13%20214128.png)
*Parametrização do tópico Standard no console do Amazon SNS.*

![Tópico Criado com Sucesso](images/Captura%20de%20tela%202026-05-13%20214222.png)
*Painel de gerenciamento exibindo os detalhes gerais e ARN do novo tópico.*

---

### 2. Criação da Assinatura (Subscription)
1. Dentro da página do tópico criado, vá até a aba **Assinaturas** e clique em **Criar assinatura**.
2. No campo **Protocolo**, escolha **Amazon SQS**.
3. No campo **Endpoint**, insira o ARN exato da fila principal (`minha-fila-principal-lab`).
4. Clique em **Criar assinatura**.

![Configurando a Assinatura SQS](images/Captura%20de%20tela%202026-05-13%20214231.png)
*Mapeamento do protocolo e vinculação do endpoint correspondente à fila consumidora principal.*

![Assinatura Ativada no Tópico](images/Captura%20de%20tela%202026-05-13%20214550.png)
*Visualização da assinatura vinculando com sucesso o barramento ao buffer do SQS.*

---

## 🔒 Fase 3: Restrição de Acesso e Segurança (IAM Resource Policy)

Para impedir que agentes maliciosos ou serviços não autorizados injetem dados na nossa fila, editamos a política de acesso baseada em recursos do SQS para aplicar o privilégio mínimo.

1. Volte ao console do **Amazon SQS** e clique na sua fila principal (`minha-fila-principal-lab`).
2. Acesse a aba **Política de acesso** e clique em **Editar**.
3. No editor JSON, remova a permissão genérica padrão e configure uma regra estrita limitando a ação `sqs:SendMessage` ao serviço `sns.amazonaws.com` com a condição de origem do seu ARN do SNS.
4. Clique em **Salvar**.

![Acessando Configurações de Segurança da Fila](images/Captura%20de%20tela%202026-05-13%20215437.png)
*Aba de políticas de acesso nativa da fila principal antes das alterações de privilégio mínimo.*

![Estruturação da Política JSON Customizada](images/Captura%20de%20tela%202026-05-13%20215547.png)
*Edição manual da IAM Resource Policy restringindo os métodos de publicação ao SNS de origem.*

---

## 🧪 Fase 4: Validação da Arquitetura e Teste de Redrive (DLQ)

A última fase valida se os tempos e limites de mensageria estão se comportando como o esperado no ecossistema distribuído.

### 1. Publicação do Evento via SNS
1. No console do **Amazon SNS**, entre no tópico `meu-topico-sns-lab`.
2. Clique no botão **Publicar mensagem**.
3. No corpo da mensagem, insira um texto estruturado de teste e envie.

![Publicando Evento de Teste no SNS](images/Captura%20de%20tela%202026-05-13%20220154.png)
*Disparo de payload estruturado através do console do barramento para simular um microsserviço produtor.*

---

### 2. Simulação de Falhas Consecutivas e Transbordo para DLQ
1. Retorne ao painel do **Amazon SQS** e selecione a fila `minha-fila-principal-lab`. Note que o contador de **Mensagens disponíveis** agora marca **1**.
2. Clique em **Enviar e receber mensagens** e clique no botão **Sondar mensagens** (*Poll for messages*).

![Sondagem Inicial de Mensagens Disponíveis](images/Captura%20de%20tela%202026-05-13%20220849.png)
*Identificação da mensagem retida no buffer da fila principal aguardando processamento.*

3. A mensagem enviada pelo SNS aparecerá na lista. Clique nela para inspecionar os detalhes e verificar que o contador **Contagem de recebimentos** (*Receive count*) marcará **1**.

![Primeira Tentativa de Recebimento](images/Captura%20de%20tela%202026-05-13%20220931.png)
*Inspeção dos metadados da mensagem apontando o início do ciclo de leitura (Receive count = 1).*

4. Feche os detalhes da mensagem **sem clicar em Excluir**. Aguarde o relógio correr por **30 segundos** (tempo do *Visibility Timeout*).
5. Passados os 30 segundos, clique novamente em **Sondar mensagens** para que ele atinja o segundo ciclo de leitura.

![Segunda Tentativa de Recebimento](images/Captura%20de%20tela%202026-05-13%20220947.png)
*Validação do comportamento assíncrono com o incremento do contador de tentativas após expiração da visibilidade (Receive count = 2).*

6. Repita o processo de leitura mais uma vez até estourar o limite parametrizado de tentativas na fila principal.

![Atingindo o Limite Máximo de Erros](images/Captura%20de%20tela%202026-05-13%20222515.png)
*Monitoramento do evento na fila principal indicando o estouro do limite máximo de tentativas toleradas.*

7. Após o ciclo expirar, clique em sondar novamente. A mensagem sumirá da fila principal e será deslocada.
8. Vá até o menu do SQS, abra a fila `minha-dlq-lab` e realize a sondagem. A mensagem original estará estacionada de forma segura na DLQ pronta para auditoria!

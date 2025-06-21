# Monitoramento de Eventos no S3 com SNS e SQS

[cite_start]Este repositório contém a documentação e os artefatos do laboratório prático "Notificações de Upload/Exclusão no S3 com SNS e SQS"[cite: 1], realizado na [Escola da Nuvem](https://www.escoladanuvem.org/).

O projeto implementa uma solução serverless e orientada a eventos para notificar e registrar uploads e exclusões de arquivos em um bucket Amazon S3, utilizando um padrão de arquitetura *Fan-out*.

**Instrutor:** João Gaioso
**Aluno:** Artur Costa ([@arturcosta86](https://github.com/arturcosta86))

---

## 🎯 Objetivos

O principal desafio deste laboratório foi configurar uma infraestrutura na AWS para:

* [cite_start]**Receber notificações por e-mail:** Alertar administradores sempre que um arquivo for carregado (`upload`) ou excluído (`delete`) de um bucket S3.
* [cite_start]**Registrar eventos em uma fila:** Manter um log de todos esses eventos em uma fila SQS para fins de auditoria, análise posterior ou integração com outros sistemas.

---

## 🛠️ Arquitetura da Solução

A solução utiliza uma integração entre três serviços da AWS, seguindo o fluxo abaixo:

1.  Um **arquivo é carregado ou excluído** do `Bucket S3`.
2.  [cite_start]O S3 gera uma **Notificação de Evento**.
3.  Essa notificação é publicada em um **Tópico SNS**.
4.  O SNS, utilizando o padrão **Fan-out**, distribui a mensagem para todos os seus inscritos (subscriptions):
    * [cite_start]Um **E-mail** é enviado para o endereço de e-mail cadastrado.
    * [cite_start]Uma **mensagem** é enviada para a `Fila SQS`.

---

## 📄 Políticas JSON (O "Código" da Integração)

A comunicação segura entre os serviços AWS (S3, SNS, SQS) é controlada por políticas de acesso baseadas em recursos (JSON). Abaixo estão os modelos das políticas utilizadas.

### Política do Tópico SNS (Permite S3 publicar no SNS)

Esta política é anexada ao tópico SNS para permitir que o serviço S3 publique mensagens nele.

```json
{
    "Version": "2008-10-17",
    "Id": "_default_policy_ID",
    "Statement": [
        {
            "Sid": "default_statement_ID",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": [
                "SNS:Publish",
                "SNS:RemovePermission",
                "SNS:SetTopicAttributes",
                "SNS:DeleteTopic",
                "SNS:ListSubscriptionsByTopic",
                "SNS:GetTopicAttributes",
                "SNS:Receive",
                "SNS:AddPermission",
                "SNS:Subscribe"
            ],
            "Resource": "ARN_DO_SEU_TOPICO_SNS",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceOwner": "SEU_ID_DE_CONTA_AWS"
                }
            }
        },
        {
            "Sid": "AllowS3ToPublishToTopic",
            "Effect": "Allow",
            "Principal": {
                "Service": "s3.amazonaws.com"
            },
            "Action": "SNS:Publish",
            "Resource": "ARN_DO_SEU_TOPICO_SNS",
            "Condition": {
                "ArnLike": {
                    "aws:SourceArn": "ARN_DO_SEU_BUCKET_S3"
                }
            }
        }
    ]
}
```
*[Baseado no modelo do laboratório]* 

### Política da Fila SQS (Permite SNS enviar para SQS)

Esta política é anexada à fila SQS para permitir que o tópico SNS envie mensagens para ela.

```json
{
    "Version": "2012-10-17",
    "Id": "default_policy_ID",
    "Statement": [
        {
            "Sid": "default_statement_ID",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": [
                "SQS:SendMessage",
                "SQS:ReceiveMessage",
                "SQS:DeleteMessage",
                "SQS:GetQueueAttributes",
                "SQS:SetQueueAttributes",
                "SQS:ListQueues"
            ],
            "Resource": "ARN_DA_SUA_FILA_SQS"
        },
        {
            "Sid": "AllowSNSToSendMessageToSQS",
            "Effect": "Allow",
            "Principal": {
                "Service": "sns.amazonaws.com"
            },
            "Action": "sqs:SendMessage",
            "Resource": "ARN_DA_SUA_FILA_SQS",
            "Condition": {
                "ArnEquals": {
                    "aws:SourceArn": "ARN_DO_SEU_TOPICO_SNS"
                }
            }
        }
    ]
}
```
[cite_start]*[Baseado no modelo do laboratório]* 

---

## ✅ Evidências de Execução

A seguir estão as capturas de tela que comprovam o funcionamento da solução.

**1. Confirmação da Inscrição (Subscription)**
O primeiro passo para receber as notificações é confirmar a inscrição do endpoint de e-mail no tópico SNS.

![Confirmação da Inscrição no Tópico SNS](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Confirma%C3%A7%C3%A3o%20do%20Subscription.png)

**2. Recebimento da Notificação de Teste do S3**
Ao criar a notificação no S3, um evento de teste é enviado para o tópico SNS, validando a integração inicial.

![E-mail com o evento de teste do S3](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Notifica%C3%A7%C3%A3o%20Evento%20no%20e-mail%20Upload%20no%20S3.png)

**3. Notificação de Upload de Objeto (ObjectCreated)**
E-mail recebido após o upload de um novo arquivo no bucket S3, contendo o JSON com os detalhes do evento `ObjectCreated:Put`.

![E-mail com notificação de upload](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Notifica%C3%A7%C3%A3o%20Evento%20no%20e-mail%20-%20Arquivo%20Delete%20no%20S3.png)

**4. Visão Geral das Notificações por E-mail**
Caixa de entrada mostrando a chegada de todas as notificações: confirmação de inscrição e os eventos de S3.

![Caixa de entrada com as notificações](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Notifica%C3%A7%C3%A3o%20Eventos%20no%20e-mail.png)

---

## 🚀 Como Replicar

1.  [cite_start]**Criar Recursos:** Crie um bucket no S3 [cite: 9][cite_start], um tópico padrão no SNS [cite: 14] [cite_start]e uma fila padrão no SQS [cite: 18][cite_start], todos na mesma região.
2.  [cite_start]**Configurar Inscrições (Subscriptions):** Inscreva seu e-mail no tópico SNS e confirme a inscrição.
3.  [cite_start]**Aplicar Políticas:** Edite as políticas de acesso do tópico SNS e da fila SQS com os JSONs fornecidos, substituindo os placeholders pelos ARNs e ID da sua conta.
4.  [cite_start]**Criar Notificação no S3:** Nas propriedades do bucket, crie uma notificação de evento para todos os eventos de criação e remoção de objetos, definindo o tópico SNS como destino.
5.  [cite_start]**Inscrever Fila SQS:** Inscreva o ARN da sua fila SQS no mesmo tópico SNS, habilitando o "raw message delivery".
6.  [cite_start]**Testar:** Faça o upload e depois exclua um arquivo do seu bucket S3 para verificar o recebimento das notificações por e-mail e as mensagens na fila SQS.
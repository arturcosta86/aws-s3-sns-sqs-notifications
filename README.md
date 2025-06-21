# Monitoramento de Eventos no S3 com SNS e SQS

Este repositório contém a documentação e os artefatos do laboratório prático "Notificações de Upload/Exclusão no S3 com SNS e SQS", realizado na Escola da Nuvem.

O projeto implementa uma solução serverless e orientada a eventos para notificar e registrar uploads e exclusões de arquivos em um bucket Amazon S3, utilizando um padrão de arquitetura *Fan-out*.

**Instrutor:** João Gaioso ([@Gaiosojoao](https://github.com/Gaiosojoao)) 
**Aluno:** Artur Costa ([@arturcosta86](https://github.com/arturcosta86))

---

## 🎯 Objetivos

O principal desafio deste laboratório foi configurar uma infraestrutura na AWS para:

* Receber notificações por e-mail sempre que um arquivo for carregado (`upload`) ou excluído (`delete`) de um bucket S3.
* Registrar eventos em uma fila para fins de auditoria, análise posterior ou integração com outros sistemas.

---

## 🛠️ Arquitetura da Solução

A solução utiliza uma integração entre três serviços da AWS, seguindo o fluxo abaixo:

1.  Um **arquivo é carregado ou excluído** do `Bucket S3`.
2.  O S3 gera uma **Notificação de Evento**.
3.  Essa notificação é publicada em um **Tópico SNS**.
4.  O SNS, utilizando o padrão **Fan-out**, distribui a mensagem para todos os seus inscritos (subscriptions):
    * Um **E-mail** é enviado para o endereço de e-mail cadastrado.
    * Uma **mensagem** é enviada para a `Fila SQS`.

---

## 📄 Políticas JSON (O "Código" da Integração)

A comunicação segura entre os serviços AWS é controlada por políticas de acesso em JSON. Abaixo estão os modelos das políticas utilizadas no laboratório.

### Política do Tópico SNS (Permite S3 publicar no SNS)

```json
{
    "Version": "2008-10-17",
    "Statement": [
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

### Política da Fila SQS (Permite SNS enviar para SQS)

```json
{
    "Version": "2012-10-17",
    "Statement": [
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

---

## ✅ Evidências de Execução

A seguir estão as capturas de tela que comprovam o funcionamento da solução, na sequência em que os eventos ocorrem no laboratório.

**1. Confirmação da Inscrição (Subscription)**
* **Descrição:** Prova de que a inscrição do e-mail no tópico SNS foi confirmada com sucesso, habilitando o recebimento de notificações.

![Confirmação da Inscrição no Tópico SNS](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Confirma%C3%A7%C3%A3o%20do%20Subscription.png)

**2. Recebimento da Notificação de Teste do S3**
* **Descrição:** E-mail recebido com o evento de teste (`s3:TestEvent`), enviado automaticamente pelo S3 ao criar a notificação, validando a integração com o SNS.

![E-mail com o evento de teste do S3](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Notifica%C3%A7%C3%A3o%20Evento%20no%20e-mail%20Upload%20no%20S3.png)

**3. Notificação de Upload de Objeto (ObjectCreated)**
* **Descrição:** Notificação real de um evento de upload (`ObjectCreated:Put`) recebida por e-mail após carregar um arquivo no bucket.

![E-mail com notificação de upload](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Notifica%C3%A7%C3%A3o%20Evento%20no%20e-mail%20-%20Arquivo%20Delete%20no%20S3.png)

**4. Visão Geral das Notificações na Caixa de Entrada**
* **Descrição:** Visão geral da caixa de e-mails, mostrando a sequência de notificações recebidas (confirmação e eventos do S3).

![Caixa de entrada com as notificações]([Artur%20-%20Notifica%C3%A7%C3%A3o%20Eventos%20no%20e-mail.png](https://github.com/arturcosta86/aws-s3-sns-sqs-notifications/blob/main/Artur%20-%20Notifica%C3%A7%C3%A3o%20Eventos%20no%20e-mail.png))
````

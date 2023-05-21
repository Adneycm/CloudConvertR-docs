# Implementação

Nessa página da documentação nós vamos entender como o código terraform fora implementado para a construção do projeto.

## Terraform

Terraform é uma ferramenta de infraestrutura como código (IaC) para provisionar e gerenciar recursos de forma automatizada e escalável. Gerenciado pela empresa HashiCorp, o terraform possui integração com diversos providers, incluindo a AWS que será o provider utilizado nesse projeto. Toda a infraestrutura da aplicação está implementada em terraform:


![terraform-architecture](assets/terraform-architecture.png)


A construção de um componente em terraform é feita da seguinte maneira:
```terraform linenums="1"
resource "provider_instance" "example" {
  name = "name-example"
  ...
}
```

O código Terraform do projeto está localizado em um único arquivo, ```terraform/main.tf```. Neste guia, vamos analisar detalhadamente como cada componente dentro do código está configurado e como cada um deles se interliga com os demais.

#### S3

O Amazon S3 (Simple Storage Service) é um serviço de armazenamento de objetos altamente escalável fornecido pela AWS. Armazena arquivos e oferece alta disponibilidade, escalabilidade automática e recursos de segurança avançados. É amplamente utilizado para armazenamento de dados, backup e hospedagem de sites.

No nosso projeto nós fazemos a utilização de dois buckets S3, um como input de arquivos markdown, e outro como output do função de conversão Lambda, que armazenará os arquivos HTML.

![S3](assets/S3.png)


Para a criação desses buckets fora construído o seguinte código:

```terraform linenums="1"
resource "aws_s3_bucket" "input-bucket-cloudconvertr" {
  bucket = "input-bucket-cloudconvertr"
}

resource "aws_s3_bucket" "output-bucket-cloudconvertr" {
  bucket = "output-bucket-cloudconvertr"
}
```

Para que o upload de um novo arquivo consiga realizar o _trigger_ das etapas subsequentes, nós precisamos ser alertados quando a ação de upload seja realizada. Portanto, fora adicionado a parte de código abaixo:

```terraform linenums="1"
resource "aws_s3_bucket_notification" "bucket_notification" {
  bucket = aws_s3_bucket.input-bucket-cloudconvertr.id

  topic {
    topic_arn  = aws_sns_topic.sns-CloudConvertR.arn
    events     = ["s3:ObjectCreated:Put", "s3:ObjectCreated:Post"]
  }
}
```


Observe que na linha '2' fora especificado o bucket que queremos monitorar, e na linha 6 as ações em que esse alerta deve ser disparado, que são as ações de Put e Post. Com isso o nosso bucket já está configurado para ativar os processos seguintes.

```terraform hl_lines="2 6" linenums="1"
resource "aws_s3_bucket_notification" "bucket_notification" {
  bucket = aws_s3_bucket.input-bucket-cloudconvertr.id

  topic {
    topic_arn  = aws_sns_topic.sns-CloudConvertR.arn
    events     = ["s3:ObjectCreated:Put", "s3:ObjectCreated:Post"]
  }
}
```

Com isso, nosso código para a configuração do S3 ficaria assim:

```terraform linenums="1"
resource "aws_s3_bucket" "input-bucket-cloudconvertr" {
  bucket = "input-bucket-cloudconvertr"
}

resource "aws_s3_bucket" "output-bucket-cloudconvertr" {
  bucket = "output-bucket-cloudconvertr"
}

resource "aws_s3_bucket_notification" "bucket_notification" {
  bucket = aws_s3_bucket.input-bucket-cloudconvertr.id

  topic {
    topic_arn  = aws_sns_topic.sns-CloudConvertR.arn
    events     = ["s3:ObjectCreated:Put", "s3:ObjectCreated:Post"]
  }
}
```

#### SNS

O Amazon Simple Notification Service (Amazon SNS) é um serviço gerenciado que fornece entrega de mensagens de editores para assinantes (também conhecido comoProdutoreseConsumidores). Os editores se comunicam de maneira assíncrona com os assinantes produzindo e enviando mensagens para um tópico, que é um canal de comunicação e um ponto de acesso lógico. Os clientes podem se inscrever no tópico SNS e receber mensagens publicadas usando um tipo de endpoint compatível, como Amazon Kinesis Data Firehose, Amazon SQS,AWS Lambda, HTTP, e-mail, notificações push móveis e mensagens de texto móveis (SMS).

Para a construção do SNS, fora preciso a construção de uma policy. Uma policy é uma série de configurações que permite fornecer acessos a componentes de maneira regrada. Para o caso do SNS, sua policy permite que a notificação de upload do bucket S3 de input, possa escrever no tópico do SNS.

![SNS](assets/SNS.png)

No código abaixo podemos ver a policy criada. Nas linhas '10' e '11' podemos ver a ação permitida ("SNS:Publish" -> ação de publicar menssagem no tópico SNS) e a qual recurso estamos permitindo essa ação seja feita ("sns-CloudConvertR" -> Tópico SNS que está conectando o nosso bucket de input ao SQS).

```terraform hl_lines="10 11" linenums="1"
data "aws_iam_policy_document" "topic" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["s3.amazonaws.com"]
    }

    actions   = ["SNS:Publish"]
    resources = ["arn:aws:sns:*:*:sns-CloudConvertR"]

    condition {
      test     = "ArnLike"
      variable = "aws:SourceArn"
      values   = [aws_s3_bucket.input-bucket-cloudconvertr.arn]
    }
  }
}
```

Observer que a policy possui um campo de "condition", o que demonstra que essa ação só pode ser performada caso a condição seja satisfeita. Nessa caso a condição está restringindo os elementos que podem realizar essa ação, no caso desse projeto, ao bucket de input de arquivos

```terraform hl_lines="16" linenums="1"
data "aws_iam_policy_document" "topic" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["s3.amazonaws.com"]
    }

    actions   = ["SNS:Publish"]
    resources = ["arn:aws:sns:*:*:sns-CloudConvertR"]

    condition {
      test     = "ArnLike"
      variable = "aws:SourceArn"
      values   = [aws_s3_bucket.input-bucket-cloudconvertr.arn]
    }
  }
}
```

Com a policy criada podemos criar o nosso tópico SNS e integrar essas permissões a sua policy:

```terraform linenums="1"
resource "aws_sns_topic" "sns-CloudConvertR" {
  name   = "sns-CloudConvertR"
  policy = data.aws_iam_policy_document.topic.json
}
```

Outra funcionalidade implementada foi a notificação por email quando um arquivo novo é inserido no projeto. Para que essa informação consiga chegar ao email, é necessário inscrever o email nesse tópico SNS. Para isso fora implementado o código abaixo:
```terraform linenums="1"
variable "email" {
  type = string
}

resource "aws_sns_topic_subscription" "email_notification" {
  topic_arn = aws_sns_topic.sns-CloudConvertR.arn
  protocol  = "email"
  endpoint  = var.email
}
```

Com isso nosso código para configuração do SNS fica assim:

```terraform linenums="1"
data "aws_iam_policy_document" "topic" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["s3.amazonaws.com"]
    }

    actions   = ["SNS:Publish"]
    resources = ["arn:aws:sns:*:*:sns-CloudConvertR"]

    condition {
      test     = "ArnLike"
      variable = "aws:SourceArn"
      values   = [aws_s3_bucket.input-bucket-cloudconvertr.arn]
    }
  }
}

resource "aws_sns_topic" "sns-CloudConvertR" {
  name   = "sns-CloudConvertR"
  policy = data.aws_iam_policy_document.topic.json
}

variable "email" {
  type = string
}

resource "aws_sns_topic_subscription" "email_notification" {
  topic_arn = aws_sns_topic.sns-CloudConvertR.arn
  protocol  = "email"
  endpoint  = var.email
}
```

#### SQS

O Amazon Simple Queue Service (SQS) permite que você envie, armazene e receba mensagens entre componentes de software em qualquer volume, sem perder mensagens ou precisar que outros serviços estejam disponíveis.

Nesse projeto o SQS está responsável por receber as novas notificações do tópico SNS e repassá-las para a função Lambda.

![SQS](assets/SQS.png)

Para a criação da fila SQS fora implementado o seguinte código:

```terraform linenums="1"
resource "aws_sqs_queue" "sqs-CloudConvertR" {
  name = "sqs-CloudConvertR"
}
```

Também fora implementado uma policy para permitir que o tópico SNS conseguisse enviar mensagens para a fila SQS. Nas linhas '11' e '12' está especificado a ação permitida e o recurso para o qual essa ação pode ser aplicada, respectivamente. Na linha '17' podemos verificar o recurso a quem essas permissões estão sendo fornecidas, no caso do nosso projeto, o recurso é o tópico SNS.

```terraform hl_lines="11 12 17" linenums="1"
data "aws_iam_policy_document" "sqs-policy" {
  statement {
    sid    = "First"
    effect = "Allow"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.sqs-CloudConvertR.arn]

    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_sns_topic.sns-CloudConvertR.arn]
    }
  }
}
```

Para adicionar a policy ao SQS:
```terraform linenums="1"
resource "aws_sqs_queue_policy" "sqs-policy" {
  queue_url = aws_sqs_queue.sqs-CloudConvertR.id
  policy    = data.aws_iam_policy_document.sqs-policy.json
}
```
Com a permissão de receber mensagens do tópico SNS fornecida, precisamos inscrever a fila SQS no próprio tópico (um recurso só receberá as mensagens do SNS caso esteja inscrito no mesmo). Para isso é necessário implementar:
```terraform linenums="1"
resource "aws_sns_topic_subscription" "sqs_notification" {
  topic_arn = aws_sns_topic.sns-CloudConvertR.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.sqs-CloudConvertR.arn
}
```
Com isso, nosso código para a construção do SQS ficará assim:

```terraform linenums="1"

resource "aws_sqs_queue" "sqs-CloudConvertR" {
  name = "sqs-CloudConvertR"
}

data "aws_iam_policy_document" "sqs-policy" {
  statement {
    sid    = "First"
    effect = "Allow"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.sqs-CloudConvertR.arn]

    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_sns_topic.sns-CloudConvertR.arn]
    }
  }
}

resource "aws_sqs_queue_policy" "sqs-policy" {
  queue_url = aws_sqs_queue.sqs-CloudConvertR.id
  policy    = data.aws_iam_policy_document.sqs-policy.json
}

resource "aws_sns_topic_subscription" "sqs_notification" {
  topic_arn = aws_sns_topic.sns-CloudConvertR.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.sqs-CloudConvertR.arn
}
```


#### Lambda Function

O AWS Lambda é um serviço de computação sem servidor e orientado a eventos que permite executar código para praticamente qualquer tipo de aplicação ou serviço de backend sem provisionar ou gerenciar servidores.

A função Lambda, dentro do nosso projeto, tem a funcionalidade de realizar a conversão dos arquivos em markdown para HTML. Ela receberá uma mensagem vinda do SQS que conterá, no seu corpo da mensagem, o nome do arquivo que foi realizado o upload e o nome do bucket proveniente. Posteriormente analisaremos detalhadamente o código implementado em Python para realizar essa conversão.

![Lambda](assets/Lambda.png)

```terraform
data "aws_iam_policy_document" "lambda_policy" {
  statement {
    effect = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_role" {
  name               = "lambda_role"
  assume_role_policy = data.aws_iam_policy_document.lambda_policy.json
}

resource "aws_iam_role_policy_attachment" "lambda_sqs_role_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_s3_role_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_role_policy_attachment" "lambda_basic_execution_role_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}


data "archive_file" "zip_python_code" {
  type = "zip"
  source_dir  = "${path.module}/python/"
  output_path = "${path.module}/python/lambda-CloudConvertR.zip"
}

resource "aws_lambda_function" "lambda-test-cp" {
  filename      = "${path.module}/python/lambda-CloudConvertR.zip"
  function_name = "lambda-CloudConvertR"
  role          = aws_iam_role.lambda_role.arn
  handler       = "lambda-CloudConvertR.CloudConvertR"   # <nome_do_arquivo.py>.<nome_da_função_dentro_do_arquivo>
  runtime       = "python3.8"
}

resource "aws_lambda_event_source_mapping" "event_source_mapping" {
  event_source_arn = aws_sqs_queue.sqs-CloudConvertR.arn
  function_name    = aws_lambda_function.lambda-test-cp.arn
}
```



## Python

- Python

``` py hl_lines="2 4" linenums="1"
def bubble_sort(items):
    for i in range(len(items)):
        for j in range(len(items) - 1 - i):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
```
<div align="center">

# 🔄 AWS Step Functions — Orquestração de Workflows na Nuvem

[![AWS](https://img.shields.io/badge/AWS-Step%20Functions-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/step-functions/)
[![Lambda](https://img.shields.io/badge/AWS-Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white)](https://aws.amazon.com/lambda/)
[![DynamoDB](https://img.shields.io/badge/AWS-DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white)](https://aws.amazon.com/dynamodb/)
[![SNS](https://img.shields.io/badge/AWS-SNS-FF4F8B?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/sns/)
[![SQS](https://img.shields.io/badge/AWS-SQS-FF4F8B?style=for-the-badge&logo=amazonsqs&logoColor=white)](https://aws.amazon.com/sqs/)
[![S3](https://img.shields.io/badge/AWS-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)](https://aws.amazon.com/s3/)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![Status](https://img.shields.io/badge/Status-Concluído-success?style=for-the-badge)]()

<p>Laboratório prático de orquestração de workflows serverless com AWS Step Functions, integrando Lambda, S3, DynamoDB, SNS e SQS para automatizar validação e processamento de arquivos.</p>

</div>

---

## 📑 Índice

- [Sobre o Projeto](#-sobre-o-projeto)
- [Arquitetura](#-arquitetura)
- [Serviços Utilizados](#-serviços-utilizados)
- [Pré-requisitos](#-pré-requisitos)
- [Passo a Passo Completo](#-passo-a-passo-completo)
- [Erros Encontrados e Soluções](#-erros-encontrados-e-soluções)
- [Arquitetura e Decisões Técnicas](#-arquitetura-e-decisões-técnicas)
- [Insights e Aprendizados](#-insights-e-aprendizados)
- [Estrutura do Repositório](#-estrutura-do-repositório)
- [Referências](#-referências)

---

## 📌 Sobre o Projeto

Este repositório documenta o laboratório prático de **AWS Step Functions**, explorando a orquestração visual de workflows serverless com integração entre múltiplos serviços AWS.

O cenário simulado foi um **pipeline de validação e processamento de arquivos**:
- Arquivos com extensões válidas (`.csv`, `.json`, `.txt`) são processados e salvos no DynamoDB, com mensagem enfileirada no SQS
- Arquivos com extensões inválidas disparam uma notificação de erro por e-mail via SNS

---

## 🏗️ Arquitetura

```
┌──────────────────────────────────────────────────────────────────────┐
│                        AWS Step Functions                            │
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────────┐  │
│  │   Lambda    │───▶│   Choice    │───▶│         Lambda           │  │
│  │  validar-   │    │  (Válido?)  │Sim │      processar-          │  │
│  │  arquivo    │    │             │    │      arquivo             │  │
│  └─────────────┘    └──────┬──────┘    └────────────┬─────────────┘  │
│                            │ Não                    │               │
│                            ▼                        ▼               │
│                     ┌────────────┐          ┌─────────────┐         │
│                     │    SNS     │          │     SQS     │         │
│                     │  Notifica  │          │    Fila     │         │
│                     │   Erro     │          │ Processam.  │         │
│                     └────────────┘          └─────────────┘         │
└──────────────────────────────────────────────────────────────────────┘
                                                       │
                                                       ▼
                                              ┌──────────────┐
                                              │   DynamoDB   │
                                              │  Metadados   │
                                              └──────────────┘
```

**Fluxo de execução:**
```
Input ──▶ ValidarArquivo ──▶ VerificarValidade ──┬──▶ ProcessarArquivo ──▶ EnviarParaFila ──▶ [END]
                                                 │
                                                 └──▶ NotificarErro ──▶ [END]
```

---

## ☁️ Serviços Utilizados

| Serviço | Papel no Workflow |
|---|---|
| **AWS Step Functions** | Orquestrador central — coordena todos os serviços via state machine |
| **AWS Lambda** | Executa lógica de validação e processamento (serverless, Python 3.12) |
| **Amazon S3** | Bucket de entrada para os arquivos a serem processados |
| **Amazon DynamoDB** | Persiste metadados dos arquivos processados com sucesso |
| **Amazon SNS** | Envia notificação por e-mail quando um arquivo inválido é detectado |
| **Amazon SQS** | Fila para desacoplamento — recebe mensagens de arquivos processados |
| **Amazon CloudWatch** | Logs e monitoramento das execuções da state machine |
| **AWS IAM** | Controle de permissões entre todos os serviços |

---

## ✅ Pré-requisitos

- Conta AWS ativa (Free Tier suficiente)
- Permissões de administrador ou acesso aos serviços: Step Functions, Lambda, S3, DynamoDB, SNS, SQS, IAM, CloudWatch
- E-mail válido para receber notificações do SNS
- Conhecimento básico de JSON e Python

---

## 📖 Passo a Passo Completo

### 1. Bucket S3

1. Acesse **S3** no console AWS → **Create bucket**
2. Nome: `lab-stepfunctions-arquivos` *(deve ser globalmente único)*
3. Região: `us-east-1` — **use a mesma região para todos os serviços**
4. Mantenha as configurações padrão → **Create bucket**

> ⚠️ **Atenção:** usar regiões diferentes entre os serviços é uma das causas mais comuns de `ResourceNotFoundException` neste lab.

---

### 2. Tabela DynamoDB

1. Acesse **DynamoDB** → **Create table**
2. Table name: `lab-arquivos-metadata`
3. Partition key: `arquivo_id` → tipo **String**
4. Mantenha as configurações padrão → **Create table**
5. Aguarde o status mudar para **Active** antes de continuar

---

### 3. Funções Lambda

#### 3.1 — `validar-arquivo`

1. Acesse **Lambda** → **Create function** → **Author from scratch**
2. Function name: `validar-arquivo` | Runtime: **Python 3.12** → **Create function**
3. Cole o código abaixo no editor → **Deploy**

```python
import json

def lambda_handler(event, context):
    arquivo = event.get("arquivo", "")
    extensoes_validas = [".csv", ".json", ".txt"]

    valido = any(arquivo.endswith(ext) for ext in extensoes_validas)

    return {
        "arquivo": arquivo,
        "valido": valido,
        "mensagem": "Arquivo válido" if valido else "Extensão não permitida"
    }
```

#### 3.2 — `processar-arquivo`

1. Crie outra função: `processar-arquivo` | Runtime: **Python 3.12** → **Create function**
2. Cole o código abaixo → **Deploy**

```python
import json
import boto3
import uuid

def lambda_handler(event, context):
    # region_name explícito evita o erro ResourceNotFoundException
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    table = dynamodb.Table('lab-arquivos-metadata')

    arquivo_id = str(uuid.uuid4())
    table.put_item(Item={
        'arquivo_id': arquivo_id,
        'arquivo': event.get('arquivo', ''),
        'status': 'processado'
    })

    return {
        "arquivo_id": arquivo_id,
        "status": "processado com sucesso"
    }
```

3. Vá em **Configuration → Permissions** → clique na **IAM role**
4. **Add permissions → Attach policies** → adicione **AmazonDynamoDBFullAccess**

---

### 4. Tópico SNS

1. Acesse **SNS** → **Topics** → **Create topic**
2. Type: **Standard** | Name: `lab-notificacao-erro` → **Create topic**
3. Dentro do tópico: **Create subscription** → Protocol: **Email** → seu e-mail → **Create subscription**
4. Confirme a inscrição pelo e-mail recebido *(verifique spam)*
5. Copie o **ARN** do tópico — será usado na state machine

---

### 5. Fila SQS

1. Acesse **SQS** → **Create queue**
2. Type: **Standard** | Name: `lab-fila-processamento` → **Create queue**
3. Copie a **URL** e o **ARN** da fila — serão usados na state machine

---

### 6. State Machine

1. Acesse **Step Functions** → **Create state machine**
2. **Write your workflow in code** | Type: **Standard**
3. Substitua o JSON abaixo trocando todos os placeholders pelos seus valores reais:

```json
{
  "Comment": "Workflow de validação e processamento de arquivos",
  "StartAt": "ValidarArquivo",
  "States": {
    "ValidarArquivo": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGIAO:CONTA:function:validar-arquivo",
      "Next": "VerificarValidade"
    },
    "VerificarValidade": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.valido",
          "BooleanEquals": true,
          "Next": "ProcessarArquivo"
        }
      ],
      "Default": "NotificarErro"
    },
    "ProcessarArquivo": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGIAO:CONTA:function:processar-arquivo",
      "Next": "EnviarParaFila"
    },
    "EnviarParaFila": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "https://sqs.REGIAO.amazonaws.com/CONTA/lab-fila-processamento",
        "MessageBody.$": "States.JsonToString($)"
      },
      "End": true
    },
    "NotificarErro": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:REGIAO:CONTA:lab-notificacao-erro",
        "Message.$": "$.mensagem"
      },
      "End": true
    }
  }
}
```

> ⚠️ **Substitua:** `REGIAO` (ex: `us-east-1`) e `CONTA` (número de 12 dígitos da sua conta AWS). Sempre copie os ARNs diretamente do console — nunca os digite manualmente.

4. **Next** | Name: `lab-workflow-arquivos` | Permissions: **Create new role** → **Create state machine**
5. Após criar: **Configuration** → clique na **IAM role** → adicione **AmazonSNSFullAccess** e **AmazonSQSFullAccess**

---

### 7. Testes de Execução

#### ✅ Cenário 1 — Arquivo Válido

Clique em **Start execution** com o input:

```json
{ "arquivo": "relatorio.csv" }
```

Caminho esperado: `ValidarArquivo → VerificarValidade → ProcessarArquivo → EnviarParaFila`

- Verifique em **DynamoDB → Explore items** se o registro foi criado
- Verifique em **SQS → Poll for messages** se a mensagem chegou na fila

#### ❌ Cenário 2 — Arquivo Inválido

Clique em **Start execution** com o input:

```json
{ "arquivo": "foto.exe" }
```

Caminho esperado: `ValidarArquivo → VerificarValidade → NotificarErro`

- Verifique o e-mail cadastrado no SNS — a notificação deve chegar em instantes

---

### 8. Monitoramento

| Local | O que observar |
|---|---|
| **Step Functions → Executions** | Histórico, status e duração de cada execução |
| **Graph view** | Visualização do caminho percorrido no fluxo |
| **Event history** | Cada transição de estado com timestamps |
| **CloudWatch Logs** | Ativar em *Logging* nas configurações da state machine |
| **DynamoDB → Explore items** | Registros persistidos com sucesso |
| **SQS → Poll for messages** | Mensagens enfileiradas |

---

## 🐛 Erros Encontrados e Soluções

Dois erros ocorreram durante a execução do laboratório. Estão documentados aqui como referência.

---

### Erro 1 — `ResourceNotFoundException` no DynamoDB

**Mensagem de erro:**
```json
{
  "errorMessage": "An error occurred (ResourceNotFoundException) when calling the PutItem operation: Requested resource not found",
  "errorType": "ResourceNotFoundException"
}
```

**Diagnóstico:** A Lambda `processar-arquivo` não conseguiu localizar a tabela DynamoDB. Duas causas possíveis:

| Causa | Como identificar | Solução |
|---|---|---|
| Nome da tabela diferente no código | Comparar o nome no código com o exibido em DynamoDB → Tables | Copiar o nome exato e colar no código |
| Lambda e DynamoDB em regiões diferentes | Verificar o seletor de região para cada serviço | Adicionar `region_name` explicitamente no `boto3.resource()` |

**Solução aplicada:**

```python
# Antes (sem região explícita — causa o erro se os serviços estiverem em regiões diferentes)
dynamodb = boto3.resource('dynamodb')

# Depois (região explícita — resolve o problema)
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
```

---

### Erro 2 — `SNS.InvalidParameterException`

**Mensagem de erro:**
```json
{
  "error": "SNS.InvalidParameterException",
  "cause": "Invalid parameter: Topic Name (Service: AmazonSNS; Status Code: 400)"
}
```

**Diagnóstico:** O `TopicArn` na state machine ainda continha os placeholders `REGIAO` e `CONTA` sem substituição.

**Solução aplicada:**
1. Acessar **SNS → Topics → lab-notificacao-erro**
2. Copiar o ARN completo diretamente do console (ex: `arn:aws:sns:us-east-1:123456789012:lab-notificacao-erro`)
3. Editar a state machine e colar o ARN correto no `TopicArn`
4. Verificar que a IAM role da state machine possuía a permissão `sns:Publish`

> 💡 **Lição:** ARNs devem sempre ser copiados do console, nunca digitados manualmente. Qualquer caractere incorreto gera erros difíceis de diagnosticar.

---

## 🧠 Arquitetura e Decisões Técnicas

### Step Functions como orquestrador central

O Step Functions atua como "maestro" da arquitetura: cada serviço faz sua parte de forma isolada e desacoplada, enquanto o Step Functions coordena a sequência, trata erros e passa dados entre estados. A alternativa — implementar toda a lógica dentro de uma única Lambda — aumenta o acoplamento, elimina a visibilidade do fluxo e dificulta a manutenção.

### Tipos de estados utilizados

| Estado | Função | Usado em |
|---|---|---|
| `Task` | Executa um trabalho externo | Lambda, SNS, SQS |
| `Choice` | Bifurca o fluxo com base em condição | Verificar `$.valido` |

### Standard vs Express Workflows

| | Standard | Express |
|---|---|---|
| **Duração máxima** | 1 ano | 5 minutos |
| **Histórico** | Completo no console | Apenas via CloudWatch |
| **Cobrança** | Por transição de estado | Por execução + duração |
| **Ideal para** | Processos críticos, auditorias | Streaming, IoT, alto volume |

**Decisão neste lab:** Standard — para permitir monitoramento visual completo de cada execução, essencial durante o aprendizado.

### Integração nativa com serviços AWS (sem Lambda intermediária)

Os estados `EnviarParaFila` e `NotificarErro` usam integração direta via:
- `arn:aws:states:::sqs:sendMessage`
- `arn:aws:states:::sns:publish`

Isso elimina Lambdas intermediárias que existiriam apenas para chamar outros serviços — reduzindo latência, custo e complexidade desnecessária.

### Desacoplamento via SQS

O SQS garante que o resultado do processamento fique disponível para um consumidor futuro sem que o workflow precise saber quem vai consumi-lo ou quando. Produtor (Step Functions) e consumidor evoluem de forma independente.

---

## 💡 Insights e Aprendizados

**1. Visibilidade como vantagem arquitetural**
O Graph view do Step Functions é uma das funcionalidades mais valiosas do serviço. Ver o caminho percorrido em cada execução com indicadores visuais de sucesso e falha torna o debug muito mais rápido do que analisar logs isolados de cada serviço.

**2. ARNs são críticos — sempre copie do console**
Pequenos erros em ARNs causam falhas que podem ser confusas de diagnosticar. O aprendizado prático com o erro do SNS reforçou que copiar diretamente do console é a única prática segura.

**3. IAM é a camada invisível mais importante**
Boa parte dos erros em integrações AWS são de permissão. A role da state machine precisa de acesso explícito a cada serviço que ela chama. A role das Lambdas precisa de acesso a cada serviço que elas acessam. Definir isso corretamente desde o início evita retrabalho.

**4. Região consistente é fundamental**
Criar recursos em regiões diferentes é uma das causas mais comuns de `ResourceNotFoundException`. Definir uma região no início do projeto e usá-la para todos os recursos elimina essa classe de erro completamente.

**5. O estado `Choice` é poderoso e sem código**
Implementar lógica condicional no fluxo apenas com JSON — sem escrever código — é uma das grandes vantagens do Step Functions. Condições sobre booleans, strings, números e expressões JSONPath são suportadas nativamente.

**6. Serverless não dispensa boas práticas de arquitetura**
Mesmo sem servidores para gerenciar, decisões como desacoplamento via mensageria, separação de responsabilidades entre funções e tratamento explícito de erros são tão importantes quanto em sistemas tradicionais.


---

## 📚 Referências

- [Documentação oficial — AWS Step Functions](https://docs.aws.amazon.com/step-functions/)
- [Amazon States Language (ASL)](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html)
- [Integrações SDK nativas do Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/supported-services-awssdk.html)
- [Workshop oficial AWS Step Functions](https://catalog.workshops.aws/stepfunctions/en-US)
- [AWS Serverless Application Model (SAM)](https://docs.aws.amazon.com/serverless-application-model/)

---

<div align="center">

Feito com 🧠 e ☁️ durante o laboratório prático de AWS Step Functions

</div>

# 3 códigos Python para carregar dados com mais eficiência

**PostgreSQL com `COPY` | MongoDB com `insert_many` | Amazon S3 com `upload_file`**

Material técnico de apoio criado para os conteúdos de **Walter Gonzaga**, Engenheiro e Arquiteto de Dados, e para profissionais que desejam desenvolver pipelines mais eficientes e confiáveis.

> **Objetivo:** aprender a implementar três operações comuns na Engenharia de Dados, entender cada linha do código, evitar erros frequentes e saber quando escolher outra abordagem. Este material é didático: antes de usar em produção, avalie políticas de segurança, requisitos, volume, disponibilidade e regras de reprocessamento.

## Sumário

1. [O que você vai aprender](#1-o-que-você-vai-aprender)
2. [Estrutura do projeto e preparação](#2-estrutura-do-projeto-e-preparação)
3. [PostgreSQL: carga em lote com COPY](#3-postgresql-carga-em-lote-com-copy)
4. [MongoDB: inserção de documentos com insert_many](#4-mongodb-inserção-de-documentos-com-insert_many)
5. [Amazon S3: upload de arquivos com transferência multipart](#5-amazon-s3-upload-de-arquivos-com-transferência-multipart)
6. [Comparativo e alternativas](#6-comparativo-e-alternativas)
7. [Recomendações para produção](#7-recomendações-para-produção)
8. [Como testar desempenho sem enganar a si mesmo](#8-como-testar-desempenho-sem-enganar-a-si-mesmo)
9. [Perguntas frequentes](#9-perguntas-frequentes)
10. [Documentação oficial](#10-documentação-oficial)

---

## 1. O que você vai aprender

| Destino | Solução | Por que utilizar? | Principal cuidado |
|---|---|---|---|
| PostgreSQL | `psycopg` + `COPY FROM STDIN` | Carrega registros em fluxo, evitando um `INSERT` por linha | Esquema compatível, transações e chaves duplicadas |
| MongoDB | `pymongo` + `insert_many()` | Agrupa inserções de documentos e evita chamadas individuais | Inserção parcial, duplicação e lotes grandes demais |
| Amazon S3 | `boto3` + `upload_file()` + `TransferConfig` | Pode usar multipart e concorrência para transferir arquivos grandes | Credenciais, permissão, custos e sobrescrita de objetos |

**Importante:** essas técnicas *podem* melhorar desempenho, mas não existe ganho percentual garantido. O resultado depende do tamanho e formato dos dados, rede, limites do serviço, índices, concorrência, latência e infraestrutura. `COPY` e `insert_many` são operações de *carga*; `upload_file` é uma operação de *transferência de objetos*. São problemas diferentes.

### Visão geral

```text
Arquivo CSV             Arquivo JSONL             Arquivo local
     |                        |                         |
     v                        v                         v
Psycopg COPY          PyMongo insert_many      Boto3 upload_file
     |                        |                         |
     v                        v                         v
PostgreSQL                 MongoDB                  Amazon S3
```

## 2. Estrutura do projeto e preparação

```text
guia_cargas_python/
├── README.md
├── requirements.txt
├── .env.example
├── .gitignore
├── dados/
│   ├── vendas.csv
│   └── eventos.jsonl
├── sql/
│   └── 01_criar_tabela.sql
└── scripts/
    ├── 01_postgresql_copy.py
    ├── 02_mongodb_insert_many.py
    └── 03_s3_upload.py
```

### 2.1 Pré-requisitos

- Python **3.10 ou superior** (os exemplos usam anotações de tipos modernas e o operador `:=`).
- PostgreSQL disponível para o exemplo 1, MongoDB para o exemplo 2 e acesso ao S3 para o exemplo 3.
- Acesso de rede e permissões para criar/inserir dados no destino escolhido.
- Um terminal executando os comandos **a partir da pasta `guia_cargas_python`**.

### 2.2 Instalar as bibliotecas

No Linux/macOS:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

No Windows (PowerShell):

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

O `requirements.txt` contém `psycopg[binary]`, `pymongo` e `boto3`, em faixas de versões compatíveis. Para projetos empresariais, trave versões exatas após homologação e mantenha processo de atualização de dependências.

### 2.3 Configurar variáveis de ambiente

Copie `.env.example` como referência. **O código não carrega arquivos `.env` automaticamente**: as variáveis precisam estar exportadas no terminal, fornecidas pelo orquestrador ou obtidas de um gerenciador de segredos.

No PowerShell, por exemplo:

```powershell
$env:PG_DSN = "postgresql://usuario:SENHA@localhost:5432/engenharia"
$env:MONGO_URI = "mongodb://usuario:SENHA@localhost:27017/?authSource=admin"
$env:S3_BUCKET = "meu-bucket"
```

No Linux/macOS, por exemplo:

```bash
export PG_DSN='postgresql://usuario:SENHA@localhost:5432/engenharia'
export MONGO_URI='mongodb://usuario:SENHA@localhost:27017/?authSource=admin'
export S3_BUCKET='meu-bucket'
```

- **Nunca** inclua senhas reais no Git, em prints ou em vídeos. O arquivo `.gitignore` já ignora `.env`.
- Se a senha de conexão contiver `@`, `#`, `:` ou outros caracteres especiais, faça a codificação de URL apropriada ou use parâmetros seguros de conexão.
- Para AWS, prefira credenciais da cadeia padrão do SDK (IAM Role, perfil AWS ou ambiente gerenciado). Evite chaves fixas no código.
- `PG_CSV_FILE`, `MONGO_JSONL_FILE`, `S3_FILE`, `S3_KEY` e `MONGO_BATCH_SIZE` são opcionais e possuem valores de demonstração.

---

## 3. PostgreSQL: carga em lote com COPY

### 3.1 O problema que estamos resolvendo

Uma abordagem ingênua envia milhares de instruções `INSERT` individuais para o banco. Em cargas grandes, essa estratégia pode criar overhead de rede, parsing e execução por registro.

O comando **`COPY FROM STDIN`** recebe o conteúdo em fluxo e foi projetado para importações de alto volume. No nosso exemplo, o Python lê o CSV em blocos de **1 MiB** e transmite cada bloco ao PostgreSQL, sem carregar o arquivo completo em memória.

> Ler 1 MiB por vez **não** significa inserir lotes de 1 MiB como transações separadas. A operação continua sendo **um COPY dentro de uma transação**. O corte de leitura também pode acontecer no meio de uma linha do CSV: o protocolo recebe um fluxo contínuo de bytes e processa as linhas corretamente.

### 3.2 Preparar a tabela

Execute o SQL de `sql/01_criar_tabela.sql` uma vez:

```sql
CREATE TABLE IF NOT EXISTS public.vendas (
    id BIGINT PRIMARY KEY,
    valor NUMERIC(14, 2) NOT NULL CHECK (valor >= 0),
    criado_em DATE NOT NULL
);
```

O arquivo `dados/vendas.csv` já vem no pacote:

```csv
id,valor,criado_em
101,125.50,2026-11-01
102,89.90,2026-11-02
103,199.00,2026-11-03
```

O PostgreSQL espera, nesse exemplo, três campos nessa ordem: `id`, `valor` e `criado_em`. A primeira linha é descartada por `HEADER TRUE`; o nome das colunas no cabeçalho não é validado automaticamente com base na tabela. O separador é vírgula; o decimal é ponto; as datas estão no formato ISO.

### 3.3 Código completo

Arquivo: `scripts/01_postgresql_copy.py`

```python
"""Carrega um CSV para PostgreSQL com COPY, sem carregar tudo na memória."""

import logging
import os
from pathlib import Path
from time import perf_counter

import psycopg

BLOCO_BYTES = 1024 * 1024  # 1 MiB por leitura do arquivo
ARQUIVO = Path(os.getenv("PG_CSV_FILE", "dados/vendas.csv"))


def main() -> None:
    if not ARQUIVO.is_file():
        raise FileNotFoundError(f"Arquivo CSV não encontrado: {ARQUIVO}")

    dsn = os.environ["PG_DSN"]  # Não registre credenciais em logs
    inicio = perf_counter()

    with psycopg.connect(dsn, connect_timeout=10) as conexao:
        with conexao.cursor() as cursor:
            with cursor.copy(
                """COPY public.vendas (id, valor, criado_em)
                   FROM STDIN WITH (FORMAT CSV, HEADER TRUE)"""
            ) as copia:
                with ARQUIVO.open("rb") as entrada:
                    while bloco := entrada.read(BLOCO_BYTES):
                        copia.write(bloco)
    # Ao sair sem exceção, o contexto da conexão confirma a transação.
    logging.info("Carga PostgreSQL concluída em %.2f s", perf_counter() - inicio)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s | %(message)s")
    try:
        main()
    except Exception:
        logging.exception("Falha na carga PostgreSQL")
        raise SystemExit(1)
```

### 3.4 Entendendo o código, linha a linha

As referências abaixo apontam para as linhas do arquivo `.py` fornecido junto com este README. Linhas em branco servem apenas para separar blocos visualmente.

| Linhas | Explicação |
|---|---|
| 1 | Docstring que resume a responsabilidade do script. |
| 3 | Importa `logging` para registrar sucesso e falhas. |
| 4 | Importa `os` para ler configurações do ambiente. |
| 5 | `Path` facilita validação e abertura de arquivos, independentemente do sistema operacional. |
| 6 | `perf_counter` mede o tempo decorrido com relógio adequado a medições de duração. |
| 8 | Importa `psycopg`, driver de comunicação com PostgreSQL. |
| 10 | Define **1.048.576 bytes**, equivalentes a 1 MiB, por leitura. Não define tamanho da transação. |
| 11 | Obtém o caminho de `PG_CSV_FILE`; se ausente, usa `dados/vendas.csv`. |
| 14 | Declara a função principal, deixando o fluxo reutilizável e testável. |
| 15–16 | Interrompe com mensagem clara se o arquivo não existir. |
| 18 | Exige a variável `PG_DSN`, contendo host, porta, banco e autenticação. Evite imprimir seu valor. |
| 19 | Marca o começo da medição de duração. |
| 21 | Abre conexão com timeout de conexão de 10 segundos; o `with` gerencia confirmação ou rollback ao sair. |
| 22 | Abre cursor, objeto usado para enviar comandos SQL. |
| 23–26 | Inicia `COPY` para a tabela e colunas definidas. `STDIN` significa que os dados serão enviados pela aplicação. `FORMAT CSV` indica o formato, e `HEADER TRUE` ignora a primeira linha. |
| 27 | Abre o CSV como arquivo binário (`rb`), preservando os bytes e evitando carregar tudo de uma vez. |
| 28 | Lê blocos até o fim. O operador `:=` atribui a variável `bloco` dentro da condição. |
| 29 | Envia o bloco recebido para o fluxo `COPY`, mantendo a ordem dos bytes. |
| 30 | Comentário sobre transação: sem exceção, o `with` da conexão confirma; com exceção, reverte a operação pendente. |
| 31 | Registra o tempo final do processo. |
| 34 | Garante que o processamento automático só ocorra ao executar o arquivo diretamente. |
| 35 | Configura mensagens de log no terminal. |
| 36–40 | Executa `main()`; caso haja falha, registra o traceback e encerra com código de saída 1 para que o orquestrador identifique o erro. |

### 3.5 Como executar

```bash
python scripts/01_postgresql_copy.py
```

Exemplo de saída:

```text
INFO | Carga PostgreSQL concluída em 0.05 s
```

O tempo acima **é ilustrativo**, não corresponde a benchmark realizado neste material. Após a execução, confira os registros:

```sql
SELECT COUNT(*) AS total FROM public.vendas;
SELECT * FROM public.vendas ORDER BY id;
```

**Atenção ao reprocessamento:** a coluna `id` é chave primária. Executar o script novamente com os mesmos registros resultará em violação de unicidade. O código faz uma carga **append**, não um `UPSERT`.

### 3.6 Erros frequentes e como resolver

| Sintoma ou erro | Causa provável | Como investigar ou corrigir |
|---|---|---|
| `FileNotFoundError` | Arquivo ausente ou execução fora da pasta raiz | Confira `PG_CSV_FILE`, caminho e diretório atual. |
| `KeyError: 'PG_DSN'` | Variável de ambiente não definida | Exporte `PG_DSN` antes de executar. |
| `OperationalError: connection refused` / timeout | Host, porta ou serviço inacessível | Confira DNS, rede, firewall, porta 5432, servidor e timeout. |
| `password authentication failed` | Credencial incorreta ou usuário sem acesso | Valide usuário, senha, configurações do servidor e TLS. |
| `relation "public.vendas" does not exist` | Tabela não criada ou schema diferente | Execute o SQL de criação no banco correto. |
| `invalid input syntax for type numeric/date` | CSV com decimal, data ou coluna fora do formato esperado | Verifique delimitador, ordem, tipos, aspas e conteúdo. |
| `duplicate key value violates unique constraint` | Chave `id` repetida em dados existentes ou na origem | Avalie staging + `ON CONFLICT`, deduplicação ou carga incremental. |
| `permission denied for table vendas` | Usuário sem `INSERT` / permissões necessárias | Solicite privilégio mínimo adequado. |
| Problemas com acentos | Encoding incompatível entre arquivo e base | Padronize UTF-8 ou converta o arquivo explicitamente. |
| Falha no meio do arquivo | Linha malformada, tipo inválido ou limite do banco | Investigue o registro informado pelo erro; a transação normalmente é revertida. |

### 3.7 Alternativas e quando usá-las

- **`cursor.executemany()`**: adequado para lotes de registros gerados em memória, especialmente quando a lógica exige comandos SQL parametrizados. Pode ser menos eficiente do que `COPY` para importações maciças.
- **`copy.write_row()`**: permite transmitir tuplas Python já estruturadas; nessa modalidade **não use `FORMAT CSV`, `DELIMITER` ou `NULL` no comando**, conforme orientação do Psycopg.
- **Staging + `INSERT ... ON CONFLICT`**: carregue primeiro em tabela temporária/de estágio com `COPY`, deduplique e depois faça upsert na tabela final. Recomendado quando o processo pode ser reexecutado.
- **`pandas.DataFrame.to_sql()`**: boa produtividade em cargas pequenas/médias, com atenção ao método de inserção, tipos e custos de conversão.
- **`COPY FROM` no servidor**: diferente de `COPY FROM STDIN`, depende de acesso do servidor ao arquivo e de permissões específicas; não confunda os dois.

#### Exemplo de tratamento de duplicidade com tabela de estágio

```sql
-- Exemplo ilustrativo: exige tabela public.vendas_stage com as mesmas colunas.
INSERT INTO public.vendas (id, valor, criado_em)
SELECT id, valor, criado_em
FROM public.vendas_stage
ON CONFLICT (id) DO UPDATE
SET valor = EXCLUDED.valor,
    criado_em = EXCLUDED.criado_em;
```

Se houver `id` duplicado **dentro da própria tabela de estágio**, deduplique antes do `INSERT ... ON CONFLICT` (por exemplo, com `ROW_NUMBER()`). Para grandes cargas, índices, WAL, restrições e concorrência também influenciam o tempo total.

---

## 4. MongoDB: inserção de documentos com insert_many

### 4.1 O problema que estamos resolvendo

Inserir um documento por operação aumenta o custo das chamadas à base. Com **`insert_many()`**, podemos enviar grupos de documentos em uma única chamada ao driver, em vez de realizar um `insert_one()` para cada item.

O exemplo consome um arquivo **JSON Lines (JSONL)**: cada linha contém um documento JSON válido. A escolha do JSONL é proposital, porque podemos ler arquivos grandes incrementalmente, sem executar `json.load()` sobre o arquivo inteiro.

O parâmetro `ordered=False` informa que as operações não precisam respeitar a ordem original. Em algumas situações, principalmente em coleções distribuídas, isso melhora a eficiência; **não significa garantia de velocidade superior nem sucesso atômico**.

### 4.2 Exemplo de entrada

Arquivo `dados/eventos.jsonl`:

```jsonl
{"_id":"evento-001","tipo":"pedido_criado","pedido_id":101,"valor":125.50}
{"_id":"evento-002","tipo":"pedido_pago","pedido_id":101,"valor":125.50}
{"_id":"evento-003","tipo":"pedido_criado","pedido_id":102,"valor":89.90}
```

Usamos `_id` explícito para que cada documento tenha identidade estável. Isso ajuda a detectar duplicidade, mas **não torna o script automaticamente idempotente**: na segunda execução, ele irá tentar inserir documentos existentes e poderá gerar erro `E11000`.

### 4.3 Código completo

Arquivo: `scripts/02_mongodb_insert_many.py`

```python
"""Insere documentos de um arquivo JSONL no MongoDB usando lotes."""

import json
import logging
import os
from pathlib import Path
from time import perf_counter
from typing import Iterator

from pymongo import MongoClient
from pymongo.errors import BulkWriteError

ARQUIVO = Path(os.getenv("MONGO_JSONL_FILE", "dados/eventos.jsonl"))
TAMANHO_LOTE = int(os.getenv("MONGO_BATCH_SIZE", "1000"))


def ler_lotes(caminho: Path, tamanho: int) -> Iterator[list[dict]]:
    if tamanho < 1:
        raise ValueError("MONGO_BATCH_SIZE precisa ser maior que zero")
    lote: list[dict] = []
    with caminho.open("r", encoding="utf-8") as entrada:
        for numero_linha, linha in enumerate(entrada, start=1):
            if not linha.strip():
                continue
            try:
                documento = json.loads(linha)
            except json.JSONDecodeError as erro:
                raise ValueError(f"JSON inválido na linha {numero_linha}") from erro
            if not isinstance(documento, dict):
                raise ValueError(f"A linha {numero_linha} não contém um objeto JSON")
            lote.append(documento)
            if len(lote) >= tamanho:
                yield lote
                lote = []
        if lote:
            yield lote


def main() -> None:
    if not ARQUIVO.is_file():
        raise FileNotFoundError(f"Arquivo JSONL não encontrado: {ARQUIVO}")
    inicio = perf_counter()
    total = 0

    with MongoClient(os.environ["MONGO_URI"], serverSelectionTimeoutMS=10000) as cliente:
        cliente.admin.command("ping")  # Testa a conectividade antes da carga
        colecao = cliente[
            os.getenv("MONGO_DATABASE", "engenharia")
        ][os.getenv("MONGO_COLLECTION", "eventos")]

        for lote in ler_lotes(ARQUIVO, TAMANHO_LOTE):
            try:
                resultado = colecao.insert_many(lote, ordered=False)
                total += len(resultado.inserted_ids)
            except BulkWriteError as erro:
                detalhes = erro.details or {}
                total += detalhes.get("nInserted", 0)
                logging.error("Falha parcial em lote; inseridos até agora: %d", total)
                raise  # Não trate um lote parcial como sucesso completo

    logging.info("MongoDB: %d documentos inseridos em %.2f s", total, perf_counter() - inicio)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s | %(message)s")
    try:
        main()
    except Exception:
        logging.exception("Falha na carga MongoDB")
        raise SystemExit(1)
```

### 4.4 Entendendo o código, linha a linha

| Linhas | Explicação |
|---|---|
| 1 | Docstring descrevendo a finalidade. |
| 3 | Carrega o módulo `json`, usado para transformar linhas em dicionários Python. |
| 4–5 | Log das operações e leitura de configurações de ambiente. |
| 6 | `Path` para acessar arquivos. |
| 7 | `perf_counter` para medir duração. |
| 8 | `Iterator` indica que a função produz lotes aos poucos, sem retornar uma lista gigante. |
| 10 | Importa `MongoClient`, responsável pela conexão ao MongoDB. |
| 11 | Importa `BulkWriteError` para identificar falhas de operações em lote. |
| 13–14 | Configura caminho do JSONL e quantidade máxima de documentos por lote, padrão **1.000**. |
| 17 | Define gerador `ler_lotes`: recebe caminho e tamanho desejado. |
| 18–19 | Valida que o lote precisa ter pelo menos um documento. |
| 20 | Inicia lista vazia que armazenará temporariamente os documentos do lote atual. |
| 21 | Abre o arquivo em modo texto com UTF-8. |
| 22 | Itera sobre cada linha, preservando o número para mensagens de diagnóstico. |
| 23–24 | Ignora linhas vazias. |
| 25–28 | Tenta decodificar JSON; se falhar, informa em qual linha ocorreu o problema. |
| 29–30 | Garante que o documento seja um objeto JSON (`dict`), não um array, número ou string. |
| 31 | Adiciona o documento válido ao lote atual. |
| 32–34 | Quando o lote atinge o limite, `yield` o entrega à função chamadora e reinicia a lista. |
| 35–36 | Ao chegar ao final, entrega o último lote mesmo que esteja incompleto. |
| 39–41 | Inicia função principal e verifica o arquivo. |
| 42–43 | Inicia cronômetro e contador de inserções concluídas. |
| 45 | Abre o cliente com a URI da variável `MONGO_URI`, limitada por timeout de seleção do servidor. |
| 46 | Faz um `ping` antes de iniciar a carga para detectar problemas básicos de conectividade. |
| 47–49 | Seleciona nome do banco e da coleção, com valores padrão caso não informados no ambiente. |
| 51 | Percorre os lotes produzidos pelo gerador. |
| 52–54 | Insere os documentos com `ordered=False` e soma o total confirmado em resposta de sucesso. |
| 55–59 | Se houver `BulkWriteError`, contabiliza inserções que o servidor informa como bem-sucedidas, registra **falha parcial** e propaga a exceção. |
| 61 | Mostra total de documentos inseridos com sucesso durante a execução e a duração. |
| 64–70 | Configura logs e encerra com status não zero se ocorrer qualquer erro. |

### 4.5 Como executar

```bash
python scripts/02_mongodb_insert_many.py
```

Exemplo de saída com os três registros do arquivo, executados em coleção vazia:

```text
INFO | MongoDB: 3 documentos inseridos em 0.10 s
```

O tempo é **apenas ilustrativo**. Confira no `mongosh`:

```javascript
use engenharia
db.eventos.find({}, { _id: 1, tipo: 1, pedido_id: 1 })
db.eventos.countDocuments({})
```

### 4.6 Erros frequentes e como resolver

| Sintoma ou erro | Causa provável | Correção recomendada |
|---|---|---|
| `FileNotFoundError` | JSONL não encontrado | Revise `MONGO_JSONL_FILE` e o diretório de execução. |
| `JSON inválido na linha N` | Falta de aspas, vírgula ou JSON quebrado | Valide a linha N e o formato JSONL (um objeto por linha). |
| `A linha N não contém um objeto JSON` | Linha com array ou valor simples | Converta para documentos JSON individualizados. |
| `ServerSelectionTimeoutError` | Host inacessível, firewall, DNS ou replica set incorreto | Verifique URI, rede, DNS, TLS e disponibilidade. |
| `Authentication failed` | Usuário/senha ou `authSource` incorretos | Revise autenticação e permissões no banco. |
| `E11000 duplicate key error` | Reprocessamento com `_id` repetido ou índice único violado | Planeje upsert, deduplicação ou identificadores únicos. |
| `BulkWriteError` | Uma ou mais operações de lote falharam | Inspecione `writeErrors`, `nInserted` e reprocessamento seguro; lote pode ter sido inserido parcialmente. |
| `DocumentTooLarge` | Documento acima do limite BSON permitido | Reestruture o conteúdo; arquivos grandes não devem ser embutidos indiscriminadamente. |
| Operação lenta | Lote inadequado, índices, rede, contenção ou infraestrutura | Faça benchmark de `MONGO_BATCH_SIZE` (ex.: 500, 1.000 e 5.000), meça gargalos e avalie índices. |
| Erro ao reutilizar arquivo `JSON` comum | Entrada é array JSON completo, não JSONL | Transforme a origem em JSONL ou adapte o leitor. |

**Muito importante:** `ordered=False` permite que o servidor continue outras inserções mesmo se algumas falharem. Por isso, o código *não considera um lote com erro como sucesso total*. Uma carga desse tipo não garante atomicidade entre todos os documentos.

### 4.7 Alternativas e quando usar

- **`insert_one()`**: bom para inserir documentos isolados e fluxos de baixo volume.
- **`insert_many(ordered=True)`**: quando manter a ordem das tentativas de inserção importa. Uma falha interrompe as operações subsequentes daquele conjunto ordenado.
- **`bulk_write()` com `ReplaceOne` ou `UpdateOne`**: recomendado para **upsert** (criar ou atualizar) e reprocessamentos que não podem gerar duplicatas.
- **`MongoDB Database Tools` (`mongoimport`)**: alternativa de linha de comando para importações; compare com a implementação Python de acordo com o ambiente.
- **Ferramentas de ingestão e CDC**: para fluxos recorrentes ou em tempo real, considere conectores, change streams e plataformas de integração, conforme requisitos de entrega.

#### Exemplo alternativo: UPSERT idempotente por `_id`

```python
from pymongo import MongoClient, ReplaceOne

lote = [
    {"_id": "evento-001", "tipo": "pedido_criado", "pedido_id": 101},
    {"_id": "evento-002", "tipo": "pedido_pago", "pedido_id": 101},
]

with MongoClient("mongodb://localhost:27017/") as cliente:
    colecao = cliente["engenharia"]["eventos"]
    operacoes = [
        ReplaceOne({"_id": doc["_id"]}, doc, upsert=True)
        for doc in lote
    ]
    resultado = colecao.bulk_write(operacoes, ordered=False)
    print("Inseridos:", resultado.upserted_count)
    print("Atualizados:", resultado.modified_count)
```

Este trecho usa uma URI local apenas para ilustrar. Em um projeto real, utilize `os.environ["MONGO_URI"]`, como no script principal. O upsert evita duplicação pelo `_id`, **desde que esse identificador seja determinístico e consistente**. Mesmo assim, o reprocessamento pode sobrescrever valores e requer regras de atualização adequadas.

---

## 5. Amazon S3: upload de arquivos com transferência multipart

### 5.1 O problema que estamos resolvendo

Para arquivos grandes, o upload em uma única transferência pode ser menos eficiente e mais sensível a falhas de rede. O recurso **multipart upload** divide o objeto em partes, que podem ser transmitidas com concorrência e então combinadas no S3.

A biblioteca **Boto3** oferece `upload_file()` como método de alto nível, com escolha automática entre operações comuns e multipart conforme o tamanho e a configuração.

No exemplo:

- `multipart_threshold = 16 MiB`: a partir do limiar configurado, a transferência é candidata ao multipart.
- `multipart_chunksize = 16 MiB`: tamanho-alvo de cada parte multipart.
- `max_concurrency = 8`: limite configurado de requisições de transferência concorrentes na implementação aplicável.
- `use_threads = True`: permite concorrência via threads no gerenciador de transferência compatível.

> **Importante:** o CSV pequeno incluído no pacote é apenas para testar conectividade e permissões; por ser menor que 16 MiB, **não acionará multipart**. Para observar o benefício, utilize um arquivo real maior que o limiar, como um Parquet de teste. A tecnologia de transferência efetiva pode variar conforme ambiente e versão do Boto3.

### 5.2 Código completo

Arquivo: `scripts/03_s3_upload.py`

```python
"""Envia um arquivo ao S3 com multipart e concorrência configuráveis."""

import logging
import os
from pathlib import Path
from time import perf_counter

import boto3
from boto3.s3.transfer import TransferConfig
from botocore.config import Config

MIB = 1024 * 1024
LIMIAR_MULTIPART = 16 * MIB
ARQUIVO = Path(os.getenv("S3_FILE", "dados/vendas.csv"))


def main() -> None:
    if not ARQUIVO.is_file():
        raise FileNotFoundError(f"Arquivo não encontrado: {ARQUIVO}")

    bucket = os.environ["S3_BUCKET"]
    chave = os.getenv("S3_KEY", f"bronze/{ARQUIVO.name}")
    transferencia = TransferConfig(
        multipart_threshold=LIMIAR_MULTIPART,
        multipart_chunksize=16 * MIB,
        max_concurrency=8,
        use_threads=True,
    )
    # Retry de erros transitórios de requisições AWS, quando aplicável.
    cliente = boto3.client(
        "s3", config=Config(retries={"mode": "standard", "total_max_attempts": 5})
    )

    inicio = perf_counter()
    cliente.upload_file(
        Filename=str(ARQUIVO),
        Bucket=bucket,
        Key=chave,
        Config=transferencia,
    )
    logging.info(
        "S3: %s (%d bytes) enviado para s3://%s/%s em %.2f s",
        ARQUIVO, ARQUIVO.stat().st_size, bucket, chave, perf_counter() - inicio,
    )


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s | %(message)s")
    try:
        main()
    except Exception:
        logging.exception("Falha no upload S3")
        raise SystemExit(1)
```

### 5.3 Entendendo o código, linha a linha

| Linhas | Explicação |
|---|---|
| 1 | Docstring descrevendo o script de envio. |
| 3–4 | Logs e variáveis de ambiente. |
| 5 | `Path` para manipular caminho do arquivo. |
| 6 | `perf_counter` para medir tempo da operação. |
| 8 | Boto3 é o SDK Python para AWS. |
| 9 | `TransferConfig` permite configurar o gerenciador de transferências do S3. |
| 10 | `Config` personaliza aspectos do cliente AWS, como comportamento de tentativas. |
| 12–13 | Define MiB e o limiar de multipart em 16 MiB. |
| 14 | Lê `S3_FILE`; por padrão usa o CSV de exemplo do projeto. |
| 17 | Declara a função principal. |
| 18–19 | Interrompe imediatamente se o arquivo não existir. |
| 21 | Lê `S3_BUCKET`, nome do bucket de destino. |
| 22 | Lê `S3_KEY`; se não existir, grava o objeto em `bronze/nome_do_arquivo`. É uma chave lógica, não uma pasta física. |
| 23–28 | Configura limite multipart, tamanho das partes, concorrência e threads. Valores devem ser ajustados por benchmark. |
| 29 | Comentário esclarecendo o objetivo das tentativas automáticas. |
| 30–32 | Cria cliente S3 com política de retry `standard`, até cinco tentativas totais por requisição aplicável. Não garante repetição integral de qualquer falha. |
| 34 | Inicia a contagem do tempo. |
| 35–40 | Envia o arquivo: `Filename` é origem local, `Bucket` é destino, `Key` é o nome do objeto, `Config` aplica as configurações. |
| 41–44 | Registra arquivo, bytes, URI S3 e tempo decorrido. A mensagem só é executada se `upload_file` retornar sem lançar exceção. |
| 47–53 | Configura logs, executa `main()` e sinaliza falha com código de saída 1 se ocorrer erro. |

### 5.4 Como executar

Confirme que seu perfil, role ou ambiente AWS tem acesso ao bucket indicado e configure `S3_BUCKET`.

```bash
python scripts/03_s3_upload.py
```

Exemplo de saída:

```text
INFO | S3: dados/vendas.csv (84 bytes) enviado para s3://meu-bucket/bronze/vendas.csv em 0.40 s
```

Valores de tempo, tamanho e bucket são **ilustrativos**. O log não demonstra, por si só, que o multipart foi ativado. Para testar com outro arquivo:

```bash
export S3_FILE='dados/vendas.parquet'
export S3_KEY='bronze/vendas.parquet'
python scripts/03_s3_upload.py
```

No Windows PowerShell, use `$env:S3_FILE = "dados/vendas.parquet"` e `$env:S3_KEY = "bronze/vendas.parquet"`.

### 5.5 Erros frequentes e como resolver

| Sintoma ou erro | Causa provável | Como corrigir |
|---|---|---|
| `FileNotFoundError` | Caminho local incorreto | Confira `S3_FILE` e a pasta de execução. |
| `KeyError: 'S3_BUCKET'` | Bucket não configurado | Defina `S3_BUCKET`. |
| `NoCredentialsError` | SDK não encontrou credenciais válidas | Configure AWS CLI, perfil, variáveis seguras ou IAM Role. |
| `AccessDenied` / HTTP 403 | Política do bucket, IAM, KMS ou restrição de rede | Confira permissões de upload, criptografia e condições da política. |
| `NoSuchBucket` / erro de endpoint | Nome ou região incorretos | Verifique bucket, conta, região e endpoint. |
| `InvalidAccessKeyId` / `SignatureDoesNotMatch` | Credenciais, assinatura ou configuração incorreta | Revise perfil, tokens e configuração sem expor segredos. |
| `S3UploadFailedError` | Falha de upload, possível erro de rede ou serviço | Inspecione causa original, políticas de retry e tamanho do arquivo. |
| Upload lento | Rede, latência, arquivo pequeno, concorrência ineficiente | Meça diferentes `max_concurrency` e tamanhos de parte, sem assumir que maior é sempre melhor. |
| Objeto anterior desapareceu | Mesmo `S3_KEY` reutilizado | Use versionamento do bucket ou chaves únicas quando necessário. |
| `SlowDown` / throttling | Limitação temporária de solicitações | Ajuste concorrência e política de retries; considere backoff. |

**Atenção:** se o bucket não tiver versionamento habilitado, enviar um novo arquivo para a mesma chave pode substituir o conteúdo existente. Isso não é uma inserção de nova versão de histórico por padrão.

### 5.6 Alternativas e quando usá-las

- **`s3.put_object()`**: simples para conteúdo pequeno ou gerado em memória; não oferece por si só o fluxo gerenciado de multipart do `upload_file`.
- **`s3.upload_fileobj()`**: conveniente quando a origem é um objeto de arquivo aberto, inclusive em pipelines que geram conteúdo dinamicamente.
- **AWS CLI (`aws s3 cp`)**: útil para operações operacionais e scripts de shell; pode aproveitar gerenciamento automático de transferência.
- **AWS DataSync / soluções gerenciadas**: interessante para migração e sincronização grandes ou recorrentes entre sistemas.
- **Parquet em vez de CSV**: isso altera o **formato de armazenamento analítico**, não o método de upload. É uma decisão diferente e pode reduzir armazenamento/tempo de leitura em consultas, conforme padrão de acesso.

### 5.7 Dicas avançadas de operação

1. **Permissões mínimas:** utilize políticas IAM restritas ao bucket e prefixo necessários; confira regras de KMS quando aplicáveis.
2. **Criptografia:** verifique requisitos da organização. O S3 possui criptografia padrão para novos uploads, mas políticas podem exigir uma chave KMS específica.
3. **Organização:** padronize chaves como `bronze/fonte/ano=2026/mes=11/arquivo.parquet` quando fizer sentido para o ecossistema de leitura.
4. **Reprocessamento:** se precisar manter histórico, use nomes versionados ou versionamento do bucket; não dependa só do mesmo `S3_KEY`.
5. **Arquivos muito grandes:** acompanhe limite de partes e requisitos de multipart; um chunk pequeno demais para objetos enormes pode exceder o limite de partes.
6. **Custos:** maior paralelismo e mais partes podem aumentar número de requisições e não necessariamente reduzir tempo total.

---

## 6. Comparativo e alternativas

| Questão | PostgreSQL | MongoDB | Amazon S3 |
|---|---|---|---|
| O que está sendo gravado? | Linhas de uma tabela relacional | Documentos BSON/JSON | Arquivos/objetos |
| Técnica principal | `COPY FROM STDIN` | `insert_many()` | `upload_file()` |
| Leitura da origem | CSV em blocos de bytes | JSONL em lotes de documentos | Arquivo local gerenciado pelo SDK |
| Consumo de memória | Proporcional ao bloco lido, mais buffers internos | Proporcional ao lote e seus documentos | Gerenciado pelo SDK; depende de concorrência/partes |
| Reexecução sem duplicar? | Não no exemplo (`PRIMARY KEY` acusa conflito) | Não no exemplo (`_id` acusa conflito) | Reutilizar chave normalmente substitui o objeto |
| Controle de falhas | Transação PostgreSQL no `with` | Pode haver inserção parcial de lotes | SDK gerencia transferência; verificar estado/objeto conforme requisitos |
| Alternativa comum | Staging + `ON CONFLICT` | `bulk_write()` com upsert | `upload_fileobj()` / CLI |

### Como escolher

**Escolha `COPY`** quando sua carga é tabular, o schema já está definido e você deseja importar grandes volumes para uma tabela PostgreSQL.

**Escolha `insert_many`** quando sua origem é composta por documentos e o objetivo é inserir novos registros na coleção. Se os mesmos eventos podem chegar novamente, prefira planejar chaves e upsert.

**Escolha `upload_file`** quando seu objetivo é transferir objetos para armazenamento S3. Para arquivos pequenos, não espere ganho relevante com multipart; para arquivos grandes, teste tamanho das partes e concorrência.

## 7. Recomendações para produção

### 7.1 Checklist mínimo

- [ ] Validar entrada: existência do arquivo, tamanho, esquema, tipos e encoding.
- [ ] Aplicar credenciais por **gerenciador de segredos**, IAM Role ou variáveis seguras.
- [ ] Definir estratégia para **reprocessamento** e evitar duplicação/sobrescrita indevida.
- [ ] Registrar logs de sucesso e erro sem imprimir senhas ou dados sensíveis.
- [ ] Configurar timeouts e retries **apenas para falhas transitórias**, com limites.
- [ ] Monitorar duração, volume processado, falhas e taxa de sucesso.
- [ ] Prever alertas para falhas repetidas e resultados incompletos.
- [ ] Definir permissões mínimas e requisitos de criptografia.
- [ ] Validar destino e fazer reconciliação de contagens/identificadores quando necessário.
- [ ] Testar com um pequeno conjunto de dados antes de escalar.

### 7.2 Retries não são sinônimo de idempotência

Uma repetição automática de carga após timeout pode executar uma operação que **já foi parcialmente aplicada**. Antes de habilitar retries irrestritos, identifique se a operação é segura para repetição:

- PostgreSQL: prefira staging, chave de negócio e merge/upsert controlado se houver reprocessamento.
- MongoDB: use `_id` determinístico, upsert e tratamento de lotes parcialmente confirmados.
- S3: avalie se pode substituir o objeto da chave existente; use versionamento ou chaves únicas se necessário.

### 7.3 Observabilidade básica

Para cada execução, monitore ao menos:

```text
pipeline_id
data_hora_inicio
data_hora_fim
duracao_segundos
origem
destino
registros_ou_bytes_processados
status
quantidade_de_erros
```

Em aplicações reais, inclua identificador de execução, correlação entre logs e métricas do destino. Evite expor dados pessoais ou credenciais nesses registros.

## 8. Como testar desempenho sem enganar a si mesmo

A afirmação "mais rápido" precisa de medições. Para comparar alternativas, mantenha as condições equivalentes:

1. **Defina o cenário:** número de linhas, tamanho do arquivo, média de bytes por documento, schema e destino.
2. **Meça a execução completa:** tempo total incluindo abertura de conexão, transferência, confirmação e verificações necessárias.
3. **Controle o ambiente:** mesma rede, instância, índices, latência, CPU e concorrência, sempre que possível.
4. **Separe aquecimento e repetição:** execute mais de uma vez e registre mediana, não só o melhor resultado.
5. **Não compare operações de semântica diferente:** `INSERT` com índices e validação não equivale necessariamente a `COPY` em tabela vazia sem índices.
6. **Teste integridade:** depois da carga, confira quantidade, chaves e dados esperados.
7. **Meça custo também:** tempo de execução, CPU, consumo de memória, rede e cobrança por requisições podem importar tanto quanto throughput.

Métricas possíveis:

```text
throughput_registros_por_segundo = registros_confirmados / duracao_segundos
throughput_mib_por_segundo = (bytes_transferidos / 1024**2) / duracao_segundos
```

Essas fórmulas não provam sozinhas que a estratégia é superior: para tomar decisões, rode testes comparáveis e documente as condições.

### Exemplos de comparações úteis

| Operação | Referência | Alternativa |
|---|---|---|
| PostgreSQL | `INSERT` individual | `COPY FROM STDIN` |
| PostgreSQL | `executemany()` | `COPY` |
| MongoDB | `insert_one()` repetido | `insert_many()` com lotes |
| MongoDB | `insert_many(ordered=True)` | `insert_many(ordered=False)` quando a ordem não importa |
| S3 | Upload de objeto pequeno | `upload_file` (sem multipart esperado) |
| S3 | Upload de arquivo grande | `upload_file` com diferentes tamanhos de partes e concorrência |

## 9. Perguntas frequentes

**1. `COPY` sempre é melhor do que `INSERT`?**  
Não. Para cargas grandes e tabulares costuma ser uma escolha forte. Para linhas isoladas, transações de negócio, upserts ou transformações específicas, outras abordagens podem ser mais apropriadas.

**2. Se o arquivo CSV tiver 10 GB, preciso de 10 GB de RAM?**  
Não com o script apresentado. Ele lê blocos de 1 MiB. Ainda existem buffers e consumo de memória do driver e sistema operacional, mas não é necessário carregar todo o arquivo na aplicação.

**3. Posso executar o script do PostgreSQL duas vezes?**  
Pode tentar, mas a segunda execução com o CSV de exemplo causará erro de chave duplicada. Reprocessamento seguro exige uma regra explícita de deduplicação/upsert ou limpeza controlada da área de estágio.

**4. Por que usar JSONL no MongoDB?**  
Porque ler um documento por linha permite formar lotes de modo incremental. Um arquivo JSON tradicional contendo um array gigante geralmente requer outra estratégia de leitura.

**5. `ordered=False` elimina erros de duplicidade?**  
Não. Ele permite continuar outras operações quando uma falha, mas ainda registra o erro e pode produzir um lote parcialmente inserido.

**6. Posso usar mais de 1.000 documentos por lote no MongoDB?**  
Sim. O driver aplica limites do protocolo, mas um tamanho adequado depende do volume e do tamanho de cada documento. Teste com dados representativos.

**7. `max_concurrency=8` significa que oito arquivos serão enviados?**  
Não. É a concorrência de requisições da transferência; para um arquivo multipart, partes do mesmo objeto podem ser transferidas paralelamente.

**8. Meu arquivo tem 2 MiB. Multipart será usado?**  
Não com o limiar de 16 MiB definido no exemplo. Arquivos pequenos normalmente não se beneficiam dessa estratégia.

**9. Posso enviar arquivos Parquet para o S3?**  
Sim. Altere `S3_FILE` e `S3_KEY` para o caminho e nome do arquivo Parquet. O S3 guarda objetos; quem decide o formato adequado para análise é a arquitetura de processamento/consulta.

**10. Esses códigos estão prontos para qualquer produção?**  
Não. São bases executáveis e educacionais. Integração com segredos, métricas, rastreabilidade, permissões, transações, políticas de retenção, custos e reprocessamento dependem dos requisitos da organização.

## 10. Documentação oficial

- **Psycopg 3: `COPY`**: https://www.psycopg.org/psycopg3/docs/basic/copy.html
- **PostgreSQL: comando `COPY`**: https://www.postgresql.org/docs/current/sql-copy.html
- **PyMongo: inserção de documentos**: https://www.mongodb.com/docs/languages/python/pymongo-driver/current/crud/insert/
- **MongoDB: `insertMany`, ordem e lotes**: https://www.mongodb.com/docs/manual/reference/method/db.collection.insertMany/
- **Boto3: personalização de transferências S3**: https://docs.aws.amazon.com/boto3/latest/reference/customizations/s3.html
- **Boto3: `S3.Client.upload_file`**: https://docs.aws.amazon.com/boto3/latest/reference/services/s3/client/upload_file.html
- **AWS: multipart upload**: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html

---

### Sobre este material

**Walter Gonzaga**  
Engenheiro e Arquiteto de Dados | Professor  
**QG da Engenharia de Dados:** https://qg-engenharia-de-dados.com.br/

Este conteúdo tem finalidade educacional. Os scripts não devem ser usados com dados sensíveis sem avaliação e controles apropriados. Nenhum benchmark quantitativo foi executado como parte deste material.

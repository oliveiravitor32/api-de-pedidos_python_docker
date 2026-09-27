# API de Pedidos

API REST para gestão de pedidos, construída em arquitetura de camadas com FastAPI e PostgreSQL, totalmente containerizada.

Trabalho 1 — Desenvolvimento de Sistemas Distribuídos.

---

## Integrantes

| Nome completo | RA | Turma |
|---|---|---|
| BRUNO BONAVIGO DACAL | N309EC4 | CC8P13
| HELEN APARECIDA DA SILVA – N070498 | CC8P13
| LUCAS DE ARAUJO VERGARA | N090758 | CC8P13
| Vitor Oliveira dos Santos | N083DG1 | CC8Q13 |


---

## Início rápido

Pré-requisito: **Docker** com Docker Compose. Nada mais precisa estar instalado (nem Python, nem PostgreSQL).

```bash
git clone <URL_DO_REPOSITORIO>
cd API-Pedidos
docker compose up -d --build
```

Pronto. A API sobe em **http://localhost:8000**.

| Recurso | URL |
|---|---|
| Documentação interativa (Swagger) | http://localhost:8000/docs |
| Verificação de saúde | http://localhost:8000/health |

Para encerrar: `docker compose down` (adicione `-v` para apagar também os dados).

---

## Endpoints

| Método | Rota | O que faz | Respostas |
|---|---|---|---|
| `POST` | `/pedidos` | Cria um pedido. A aplicação calcula o `valor_total` e define o status inicial. | `201` `422` |
| `GET` | `/pedidos` | Lista todos os pedidos. | `200` |
| `GET` | `/pedidos/{id}` | Consulta um pedido específico. | `200` `404` |
| `PATCH` | `/pedidos/{id}/status` | Altera apenas o status do pedido. | `200` `404` `409` `422` |
| `GET` | `/health` | Informa se a aplicação está operacional. | `200` |

### Criar um pedido

```bash
curl -X POST http://localhost:8000/pedidos -H "Content-Type: application/json" -d "{\"cliente\":\"Ana Souza\",\"produto\":\"Teclado Mecanico\",\"quantidade\":3,\"valor_unitario\":\"150.50\"}"
```

```json
{
  "id": 1,
  "cliente": "Ana Souza",
  "produto": "Teclado Mecanico",
  "quantidade": 3,
  "valor_unitario": "150.50",
  "valor_total": "451.50",
  "status": "CRIADO",
  "data_criacao": "2026-09-05T14:59:47.161804Z"
}
```

O cliente informa **apenas** `cliente`, `produto`, `quantidade` e `valor_unitario`. Os campos `id`, `valor_total`, `status` e `data_criacao` são definidos pela aplicação — enviá-los na requisição não tem efeito.

### Consultar e listar

```bash
curl http://localhost:8000/pedidos/1
```

```bash
curl http://localhost:8000/pedidos
```

### Alterar o status

```bash
curl -X PATCH http://localhost:8000/pedidos/1/status -H "Content-Type: application/json" -d "{\"status\":\"CONFIRMADO\"}"
```

### Respostas de erro

| Situação | Status | Corpo |
|---|---|---|
| Pedido inexistente | `404` | `{"detail":"Pedido 9999 nao encontrado."}` |
| Transição de status proibida | `409` | `{"detail":"Nao e possivel alterar o status de CONFIRMADO para CRIADO."}` |
| Dados inválidos na entrada | `422` | Lista dos campos com problema e o motivo |

---

## Arquitetura

A aplicação e o banco executam como componentes independentes, em containers separados, comunicando-se por uma rede interna do Docker.

```mermaid
flowchart LR
    Cliente["Cliente HTTP"]

    subgraph rede["Rede Docker: pedidos-network"]
        subgraph api["Container: pedidos"]
            direction TB
            C["Controllers<br/>interface HTTP"]
            S["Services<br/>regras de negocio"]
            R["Repositories<br/>acesso a dados"]
            C --> S --> R
        end
        DB[("Container: postgres<br/>PostgreSQL 16")]
        R --> DB
    end

    Cliente -->|"HTTP/JSON porta 8000"| C
    DB -.-> V[("Volume postgres_data")]
```

As três camadas são **lógicas** e vivem no mesmo processo. O PostgreSQL é outro container porque é outro componente. A unidade de replicação futura é a aplicação de Pedidos inteira.

### Responsabilidade de cada camada

| Camada | Responsabilidade | O que **não** faz |
|---|---|---|
| **Controller** | Recebe a requisição, valida o formato, devolve a resposta e o status HTTP. | Não contém regra de negócio nem SQL. |
| **Service** | Concentra as regras: calcula o `valor_total`, define o status inicial, valida transições. | Não conhece HTTP — levanta exceções de domínio, não `HTTPException`. |
| **Repository** | Encapsula o acesso ao banco. | Não conhece regras de negócio nem os schemas da API. |
| **Entity** | Mapeia a tabela do banco. | Não é exposta diretamente pela API. |
| **Schema** | Define o contrato de entrada e saída da API. | Não conhece o banco. |

A dependência flui em uma direção só: `Controller → Service → Repository → Entity`.

### Fluxo de uma requisição

```mermaid
sequenceDiagram
    participant Cli as Cliente
    participant Ctl as Controller
    participant Srv as Service
    participant Rep as Repository
    participant DB as PostgreSQL

    Cli->>Ctl: POST /pedidos
    Ctl->>Ctl: valida a estrutura (Pydantic)
    Ctl->>Srv: criar(dados)
    Srv->>Srv: valor_total = quantidade x valor_unitario
    Srv->>Srv: status = CRIADO
    Srv->>Rep: add(pedido)
    Rep->>DB: INSERT
    DB-->>Rep: id gerado
    Srv->>DB: COMMIT (unit of work)
    Srv-->>Ctl: pedido persistido
    Ctl-->>Cli: 201 Created + JSON
```

---

## Modelo de dados

Tabela `pedidos`:

| Campo | Tipo | Origem | Descrição |
|---|---|---|---|
| `id` | `integer` | banco | Identificador, gerado por sequência |
| `cliente` | `varchar(120)` | cliente | Identificação textual do cliente |
| `produto` | `varchar(120)` | cliente | Identificação textual do produto |
| `quantidade` | `integer` | cliente | Quantidade solicitada (> 0) |
| `valor_unitario` | `numeric(12,2)` | cliente | Preço de uma unidade (> 0) |
| `valor_total` | `numeric(12,2)` | **aplicação** | `quantidade × valor_unitario` |
| `status` | `varchar(20)` | **aplicação** | Estado atual do pedido |
| `data_criacao` | `timestamptz` | banco | Instante do registro (`now()`) |

### Estados do pedido

```mermaid
stateDiagram-v2
    [*] --> CRIADO: POST /pedidos
    CRIADO --> CONFIRMADO
    CRIADO --> CANCELADO
    CONFIRMADO --> CANCELADO
    CANCELADO --> [*]
```

Transições fora deste diagrama retornam **409 Conflict**. `CANCELADO` é estado terminal.

---

## Estrutura do projeto

```
app/
├── main.py                 inicialização do FastAPI e tratadores de erro
├── config.py               configuração lida do ambiente
├── controllers/            camada de API (endpoints HTTP)
│   ├── pedido_controller.py
│   └── health_controller.py
├── services/               camada de lógica (regras de negócio)
│   └── pedido_service.py
├── repositories/           camada de dados (acesso ao banco)
│   └── pedido_repository.py
├── entities/               modelos persistentes (SQLAlchemy)
│   └── pedido.py
├── schemas/                contratos de entrada e saída (Pydantic)
│   ├── pedido.py
│   └── health.py
├── database/               conexão e sessão
│   ├── connection.py
│   └── session.py
└── core/                   exceções de domínio e injeção de dependência
    ├── exceptions.py
    └── dependencies.py

Dockerfile                  imagem da aplicação
docker-compose.yml          orquestração: pedidos + postgres
requirements.txt            dependências Python
.env.example                variáveis de ambiente disponíveis
```

---

## Configuração

Nenhum endereço, usuário ou senha está fixo no código. A aplicação recebe tudo do ambiente, e o `docker-compose.yml` já traz valores padrão — **a solução sobe sem nenhum arquivo `.env`**.

| Variável | Padrão | Descrição |
|---|---|---|
| `DATABASE_URL` | `postgresql+psycopg://pedidos:pedidos@postgres:5432/pedidos` | Conexão com o banco |
| `DB_CONNECT_RETRIES` | `10` | Tentativas de conexão no startup |
| `DB_CONNECT_RETRY_DELAY` | `2` | Intervalo entre tentativas (s) |
| `DB_ECHO` | `false` | Loga o SQL gerado |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | `pedidos` | Credenciais do container do banco |

Para sobrescrever, copie `.env.example` para `.env`.

---

## Verificando a persistência

Os pedidos ficam no volume `postgres_data`, fora do container da aplicação. Reiniciar a API não afeta os dados:

```bash
docker compose restart pedidos
```

```bash
curl http://localhost:8000/pedidos/1
```

O pedido continua lá — ele nunca esteve na aplicação, e sim no banco.

Para inspecionar diretamente:

```bash
docker compose exec postgres psql -U pedidos -d pedidos -c "SELECT * FROM pedidos;"
```

---

## Decisões técnicas

**Valores monetários como string no JSON.** `valor_unitario` e `valor_total` são `Decimal`, mapeados para `NUMERIC(12,2)` no PostgreSQL e serializados como string. Isso preserva a precisão decimal exata e evita os erros de arredondamento inerentes ao ponto flutuante.

**Status como `varchar` com validação na aplicação.** Em vez de um tipo `ENUM` nativo do PostgreSQL, o status é `varchar(20)` validado pelo enum Python. Como novos estados surgirão nas próximas entregas, isso evita `ALTER TYPE` no banco a cada mudança.

**Transições de status declarativas.** As regras vivem em um dicionário no Service, não em cadeias de `if`. Adicionar um estado é acrescentar uma linha.

**Espera ativa pelo banco no startup.** O container da aplicação tenta conectar com retry, e o Compose só o inicia após o `healthcheck` do PostgreSQL passar. O primeiro `docker compose up` funciona sem intervenção manual.

**Criação automática do schema.** As tabelas são criadas no startup a partir das entidades mapeadas, dispensando qualquer passo manual de migração.

**Controle transacional no Service.** O Repository executa as operações de persistência, mas quem abre e confirma a transação é o Service — a sessão do SQLAlchemy atua como *unit of work*. Por isso o diagrama de sequência mostra o `COMMIT` partindo do Service: a decisão sobre o que é atômico pertence ao caso de uso, não ao acesso a dados. Isso permite que um único caso de uso agrupe várias operações de forma atômica, algo necessário quando novos componentes forem distribuídos.

**Tratamento de erros centralizado.** O Service levanta exceções de domínio (`PedidoNaoEncontrado`, `TransicaoStatusInvalida`); tratadores globais no `main.py` as traduzem para 404 e 409. É o que mantém a camada de negócio independente de HTTP.

---

## Tecnologias

| Componente | Tecnologia |
|---|---|
| Linguagem | Python 3.12 |
| Framework web | FastAPI |
| Validação | Pydantic v2 |
| ORM | SQLAlchemy 2.0 |
| Driver | psycopg 3 |
| Banco de dados | PostgreSQL 16 |
| Servidor | Uvicorn |
| Containerização | Docker + Docker Compose |

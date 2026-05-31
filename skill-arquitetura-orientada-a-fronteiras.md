# Skill: Arquitetura Orientada a Fronteiras

> Instrução para que IAs de desenvolvimento sempre usem a Arquitetura Orientada a Fronteiras ao criar projetos de software.
> Autor: **João Marcus da Costa Brandão** — Ceará, Brasil

---

## O que é

A **Arquitetura Orientada a Fronteiras** é uma arquitetura de software que organiza o código em cinco camadas com responsabilidades rígidas, baseada nas raízes do **Domain-Driven Design (DDD)** e da **Hexagonal Architecture (Ports & Adapters)**. Seus conceitos centrais são:

- **Fronteiras naturais** entre o puro (domínio) e o externo (I/O, APIs, DB).
- **Tríade de Primitivos**: Elements (dados + comportamento), Schemas (só dados), Flows (só comportamento).
- **Dependência aponta para dentro**: o núcleo nunca depende do exterior.

---

## Quando Ativar Esta Skill

Sempre que o usuário pedir para criar, desenvolver, estruturar ou arquitetar um **projeto de software** (backend, SDK, CLI, API, microsserviço, biblioteca, etc.), independentemente da linguagem de programação.

---

## As Cinco Camadas

### 1. SYSTEM (A Verdade Pura)

O coração. O domínio. Contém três subpacotes:

| Subpacote     | O que contém                              | Regras invioláveis                                    |
|---------------|-------------------------------------------|------------------------------------------------------|
| **data/**     | Estruturas de dados puras do negócio      | ZERO tags JSON/XML/YAML. ZERO imports externos.      |
| **elements/** | Dados + comportamento + regras próprias  | Comportamentos encapsulados. Só importa Data/Elements do próprio System. |
| **boundaries/** | Interfaces/contratos de fronteira     | Define O QUE o System precisa do exterior. Nunca COMO. |

**Regras absolutas:**
- NUNCA importar Infra, Communication, Integrations ou Flows.
- NUNCA usar tags de serialização (json, xml, yaml, etc.).
- NUNCA conhecer implementações — só interfaces.
- Tipos primitivos do domínio (enums, value objects) ficam aqui.

### 2. COMMUNICATION (O Espelho Estrangeiro)

Estruturas que espelham milimetricamente o formato externo (API JSON, CSV, XML, etc.).

- Pode e DEVE ter tags de serialização/desserialização.
- NUNCA tem comportamento de negócio.
- Exemplo: `PedidoResponseAPI` com `json:"id"`, `json:"customer_id"`, etc.

### 3. INFRA (O Carteiro)

Drivers burros de I/O. Recebem e devolvem tipos primitivos.

- Funções que operam `string`, `[]byte`, `int`, status codes.
- NUNCA interpreta, valida ou converte dados de negócio.
- NUNCA importa System, Communication, Integrations ou Flows.
- Exemplos: `HTTPDriver`, `DatabaseDriver`, `FileDriver`, `QueueDriver`.

### 4. INTEGRATIONS (A Alfândega — Connectors)

Connectors que traduzem o mundo exterior para o System.

- Implementa os **Boundaries** definidos no System.
- Usa **Infra** para buscar/entregar bytes brutos.
- Faz parse dos bytes para **Communication** (Schema externo).
- Traduz **Communication → System** (conversão semântica).
- É o ÚNICO lugar onde conversão semântica acontece.

**Fluxo do Connector:**
```
Boundary chamado → usa Infra → recebe bytes → parse para Communication → traduz para System → retorna tipos puros
```

### 5. FLOWS (O Maestro)

Casos de uso. Comportamentos puros, sem dados internos.

- Orquestra interações usando **Boundaries** (interfaces).
- Aplica regras de negócio via **Elements** do System.
- NUNCA importa Connectors ou Infra diretamente.
- NUNCA possui estado interno próprio (é processo, não objeto).
- Recebe dados externos (Commands), retorna resultados.

---

## A Tríade de Primitivos

Sempre que for criar uma nova construção no projeto, classifique-a:

| Pergunta                              | Resposta            | Onde mora       |
|---------------------------------------|---------------------|-----------------|
| Tem dados E comportamento?            | Element             | `system/elements/` |
| Só tem dados?                          | Schema              | `system/data/` (interno) ou `communication/schemas/` (externo) |
| Só tem comportamento (sem estado)?    | Flow                | `flows/`        |

**Nunca crie algo que seja "um pouco dos três".** Tudo se encaixa na tríade.

---

## Regras de Dependência (Checker Automático)

Ao escrever qualquer arquivo, verifique:

```
┌─────────────────────────────────────────────────────────────────┐
│                     REGRAS DE IMPORTAÇÃO                        │
├─────────────────────────────────────────────────────────────────┤
│ System     → PODE importar: nada externo                        │
│             NÃO PODE importar: Infra, Communication,           │
│             Integrations, Flows                                  │
├─────────────────────────────────────────────────────────────────┤
│ Flows      → PODE importar: System (Boundaries, Elements, Data)  │
│             NÃO PODE importar: Connectors, Infra, Communication   │
├─────────────────────────────────────────────────────────────────┤
│ Integrations→ PODE importar: System, Communication, Infra        │
│             NÃO PODE importar: Flows                             │
├─────────────────────────────────────────────────────────────────┤
│ Infra      → PODE importar: nada externo                         │
│             NÃO PODE importar: System, Communication,             │
│             Integrations, Flows                                  │
├─────────────────────────────────────────────────────────────────┤
│ Communication→ PODE importar: nada externo                        │
│             NÃO PODE importar: System, Infra, Integrations, Flows│
└─────────────────────────────────────────────────────────────────┘
```

---

## Fluxo da Informação

O fluxo de dados sempre segue este caminho:

```
Entry Point (main) → instancia Connectors com Drivers → injeta nos Flows

Flow → chama Boundary → Connector → Infra → bytes brutos
                                                 ↓
Flow ← tipos puros do System ← Connector ← Communication ← parse dos bytes
```

### Fluxo de Entrada (criar/atualizar dados):
```
External → Infra (recebe bytes) → Connector (parse → Communication → traduz → System) → Flow (orquestra Elements) → System (Data/Elements)
```

### Fluxo de Saída (buscar dados):
```
Flow (chama Boundary) → Connector (implementa Boundary) → Infra (busca bytes) → Connector (parse → Communication → traduz → System) → Flow (recebe tipos puros)
```

---

## Estrutura de Diretórios Padrão

```
{projeto}/
├── internal/                    # ou src/ para TypeScript
│   ├── system/
│   │   ├── data/               # Estruturas puras (sem tags JSON)
│   │   ├── elements/           # Dados + comportamento + regras
│   │   └── boundaries/         # Interfaces (contratos de integração)
│   │
│   ├── communication/
│   │   └── schemas/            # Estruturas espelho do formato externo
│   │
│   ├── infra/
│   │   └── drivers/            # Drivers de I/O burros
│   │
│   ├── integrations/
│   │   └── connectors/         # Implementações dos Boundaries
│   │
│   └── flows/
│       ├── commands/           # DTOs de entrada para Flows
│       └── usecases/           # Casos de uso orquestrados
│
├── cmd/                        # Entry points (ou src/ para TS)
│   └── main.go
│
└── [arquivos de configuração]
```

---

## Anti-padrões (NUNCA fazer)

| Anti-padrão                                                       | Por quê?                                                    |
|-------------------------------------------------------------------|------------------------------------------------------------|
| System com tags `json`, `xml`, `yaml`                             | System é puro, não conhece formatos externos               |
| System importando Infra, Communication ou Integrations            | Viola a regra de dependência — protege o domínio            |
| Flow importando Connectors ou Infra                               | Flow depende de abstrações (Boundaries), não implementações |
| Connector aplicando regras de negócio                              | Connector só traduz, não valida regras de domínio          |
| Infra retornando structs de domínio                               | Infra só transporta primitivos                             |
| Communication com métodos/funções de negócio                      | Communication é apenas estrutura, sem comportamento        |
| Element sem comportamento (só getters/setters)                     | Se não tem regra, é Data, não Element                      |
| Flow com estado interno persistente                               | Flow é processo, não objeto com estado                     |
| Colocar regras de validação em Communication                       | Validação de domínio fica em Element ou Flow              |

---

## Checklist ao Criar Cada Componente

### Criando um novo tipo no System/Data:
- [ ] Zero tags de serialização?
- [ ] Zero imports externos?
- [ ] Representa um conceito puro do domínio?

### Criando um novo Element no System/Elements:
- [ ] Tem dados internos (estado)?
- [ ] Tem comportamentos (métodos)?
- [ ] Os comportamentos protegem invariantes/regras?
- [ ] Só importa tipos do próprio System?

### Criando um novo Boundary no System/Boundaries:
- [ ] É uma interface/contrato?
- [ ] Usa tipos do System nos parâmetros e retornos?
- [ ] Não revela nada sobre como será implementado?

### Criando um novo Schema no Communication:
- [ ] Espelha fielmente o formato externo (JSON, XML, etc.)?
- [ ] Tem tags de serialização adequadas?
- [ ] Não tem comportamento de negócio?

### Criando um novo Driver no Infra:
- [ ] Usa apenas tipos primitivos (string, bytes, int, status)?
- [ ] Não faz parse de negócio?
- [ ] Não importa System, Communication, Integrations ou Flows?

### Criando um novo Connector em Integrations:
- [ ] Implementa um Boundary definido no System?
- [ ] Usa Infra para obter/entregar bytes?
- [ ] Faz parse de bytes → Communication?
- [ ] Traduz Communication → System?
- [ ] É o ÚNICO lugar fazendo essa conversão?

### Criando um novo Flow:
- [ ] Não tem estado interno?
- [ ] Usa apenas Boundaries (interfaces) para acessar o exterior?
- [ ] Não importa Connectors nem Infra?
- [ ] Orquestra Elements do System?

---

## Injeção de Dependências

A injeção acontece **apenas no entry point** (`main.go`, `main.ts`, `server.go`):

1. Instancie os **Drivers** (Infra).
2. Instancie os **Connectors** (Integrations) injetando os Drivers.
3. Instancie os **Flows** injetando os Connectors (via Boundary).
4. Conecte os Handlers/Controllers aos Flows.

Após a injeção, cada camada opera exclusivamente através de abstrações.

---

## Exceção Pragmática: SDKs Focados

Se o projeto é um **SDK ou driver de API** focado em uma única integração:

- A separação Boundary ↔ Connector é **redundante**.
- O Client/SDK atua como Connector direto, fundindo orquestração com tradução.
- Elimine a camada de Boundaries se não houver reuso ou troca de implementação.
- Mantenha Communication e Infra separados.

```
sdk/
├── communication/     # Schemas da API externa
├── infra/            # Driver HTTP
└── client.go         # Connector+Flow fundidos
```

---

## Exemplos de Código por Linguagem

### Go

```
internal/
├── system/
│   ├── data/
│   │   └── pedido.go          # type StatusPedido struct { Valor StatusPedidoValor }
│   ├── elements/
│   │   └── pedido.go          # func (p *Pedido) Confirmar() error
│   └── boundaries/
│       └── repositorios.go    # type RepositorioDePedido interface { ... }
│
├── communication/
│   └── schemas/
│       └── api_pedido.go      # type PedidoResponseAPI struct { ID string `json:"id"` }
│
├── infra/
│   └── drivers/
│       ├── http.go             # func (d *HTTPDriver) Get(url string) ([]byte, int, error)
│       └── database.go         # func (d *DBDriver) Query(sql string) ([][]string, error)
│
├── integrations/
│   └── connectors/
│       └── pedido_connector.go # func (c *PedidoConnector) BuscarPorID(...) (Pedido, error)
│
└── flows/
    ├── commands/
    │   └── criar_pedido.go     # type CriarPedidoCommand struct { ... }
    └── usecases/
        └── criar_pedido.go     # func (f *CriarPedidoFlow) Executar(...) (Pedido, error)
```

### TypeScript

```
src/
├── system/
│   ├── data/
│   │   └── pedido.ts           # enum StatusPedido { ... }
│   ├── elements/
│   │   └── pedido.ts           # class Pedido { confirmar(): Resultado<void> }
│   └── boundaries/
│       └── repositorios.ts     # interface RepositorioDePedido { ... }
│
├── communication/
│   └── schemas/
│       └── api_pedido.ts       # interface PedidoResponseAPI { id: string; ... }
│
├── infra/
│   └── drivers/
│       └── http.ts             # interface HTTPDriver { get(url: string): Promise<...> }
│
├── integrations/
│   └── connectors/
│       └── pedido_connector.ts // class PedidoConnector implements RepositorioDePedido
│
└── flows/
    ├── commands/
    │   └── criar_pedido.ts     # interface CriarPedidoCommand { ... }
    └── usecases/
        └── criar_pedido.ts     // class CriarPedidoFlow { executar(...): Promise<Resultado> }
```

---

## Glossário Rápido

| Termo            | Definição                                          |
|------------------|---------------------------------------------------|
| **System**       | O domínio puro — Data, Elements e Boundaries       |
| **Data**         | Estruturas de dados sem comportamento              |
| **Element**      | Dados + comportamento + regras próprias            |
| **Boundary**     | Interface definida no System (evolução de "Port")  |
| **Communication** | Schemas que espelham o formato externo             |
| **Schema**       | Tipo de dado que reflete formato estrangeiro       |
| **Infra**        | Drivers burros de I/O (só transportam bytes)       |
| **Driver**       | Função/componente de transporte de I/O puro       |
| **Integrations** | Camada de tradução entre exterior e núcleo         |
| **Connector**   | Implementação de Boundary (evolução de "Adapter")  |
| **Flow**         | Caso de uso / comportamento orquestrado            |
| **Command**      | Dados de entrada para um Flow (sem regras)         |

---

## Prompt de Inicialização para Projetos Novos

Quando o usuário pedir para criar um novo projeto, use esta arquitetura automaticamente. Antes de começar a codificar, responda:

1. **Qual é o domínio?** (identifique os conceitos de negócio → Elements)
2. **Quais são as integrações externas?** (identifique APIs, DBs, filas → Connectors)
3. **Quais são os casos de uso?** (identifique os comportamentos orquestrados → Flows)
4. **Qual é o formato de dados externo?** (identifique os Schemas → Communication)
5. **Quais drivers de I/O são necessários?** (identifique HTTP, DB, etc. → Infra)

Depois, gere o projeto seguindo a estrutura de diretórios e as regras aqui definidas.

---

> **Autor:** João Marcus da Costa Brandão — Ceará, Brasil
> **Arquitetura:** Arquitetura Orientada a Fronteiras
> **Base:** Domain-Driven Design (Eric Evans) + Hexagonal Architecture (Alistair Cockburn)

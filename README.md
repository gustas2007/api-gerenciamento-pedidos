# API de Gerenciamento de Pedidos

API REST desenvolvida em **Node.js** com **Express** para o gerenciamento de pedidos de uma loja, aplicando arquitetura em camadas, validação rigorosa de dados, máquina de estados para transição de status, tratamento centralizado de erros e testes automatizados com **Jest** e **Supertest**.

---

## 1. Arquitetura e Padrão Implementado

O projeto adota o padrão de **Arquitetura em Camadas (Layered Architecture)** com separação clara de responsabilidades ( princípio *Single Responsibility* ):

```text
├── evidencias/                  # Capturas de tela dos testes manuais no Postman
├── src/
│   ├── controllers/
│   │   └── pedidoController.js  # Recebe requisições HTTP, extrai dados e retorna status/JSON
│   ├── services/
│   │   └── pedidoService.js     # Regras de negócio, validações, transições de status e persistência em memória
│   ├── routes/
│   │   └── pedidoRoutes.js      # Mapeamento dos endpoints da rota /pedidos
│   ├── middlewares/
│   │   └── errorHandler.js      # Interceptador global de erros (distingue 400, 404 e 500)
│   ├── errors/
│   │   └── AppError.js          # Classe customizada para erros operacionais e de validação
│   ├── app.js                   # Configuração do Express, middlewares e rotas
│   └── server.js                # Inicialização do servidor HTTP na porta 3000
├── tests/
│   └── pedidos.test.js          # Suíte de testes automatizados de integração e unidade (Jest + Supertest)
└── package.json
```

### Justificativa das pastas adicionais (`errors/` e `middlewares/`)
Além das pastas `routes`, `controllers` e `services`, foram criadas as pastas `errors` e `middlewares` para isolar o tratamento de exceções. Dessa forma, erros de validação (`400`) e de recurso não encontrado (`404`) são lançados pela camada de serviço através da classe `AppError`, enquanto falhas inesperadas de execução são capturadas pelo `errorHandler` e convertidas com segurança em `HTTP 500`, sem expor detalhes internos como erros de validação do usuário.

---

## 2. Instalação e Execução

### Pré-requisitos
* **Node.js** (versão 18 ou superior recomendada)
* **npm**

### Passo a passo

1. Instale as dependências do projeto:
```bash
npm install
```

2. Inicie o servidor da aplicação:
```bash
npm start
```
O servidor iniciará por padrão em `http://localhost:3000`.

3. Para executar a suíte de testes automatizados (Jest):
```bash
npm test
```

---

## 3. Regras de Validação e Transição de Status

### Campos reconhecidos no cadastro (`POST /pedidos`)
* **`cliente`**: Obrigatório, tipo texto (`string`), não podendo ser vazio mesmo após a remoção de espaços nas extremidades (`trim()`).
* **`valor`**: Obrigatório, tipo numérico (`number`) estritamente maior que zero (`> 0`).
* **`tipo`**: Valores aceitos:
  * `entrega`
  * `retirada`
* **`prioridade`**: Valores aceitos:
  * `baixa`
  * `normal`
  * `alta`

### Máquina de Estados (Transições de Status em `PATCH /pedidos/:id/status`)
Todo pedido criado inicia automaticamente com o status **`recebido`**. As transições permitidas seguem o fluxo operacional abaixo:

| Status Atual | Próximos Status Permitidos | Observação |
| :--- | :--- | :--- |
| `recebido` | `em_preparacao`, `cancelado` | Pedido recém-criado pode ir para preparo ou ser cancelado |
| `em_preparacao` | `enviado`, `cancelado` | Pedido em preparo pode ser despachado ou cancelado |
| `enviado` | `entregue` | Após enviado, não pode mais ser cancelado, apenas entregue |
| `entregue` | *(nenhum)* | Estado final imutável |
| `cancelado` | *(nenhum)* | Estado final imutável |

Qualquer tentativa de transição fora das regras acima ou envio de um status inexistente retorna **HTTP 400 (Bad Request)** com mensagem explicativa em JSON.

---

## 4. Documentação de Rotas e Exemplos

### 4.1. Cadastrar Pedido (RF01)
* **Método e Rota:** `POST /pedidos`
* **Corpo da Requisição (JSON):**
```json
{
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal"
}
```
* **Resposta de Sucesso (`HTTP 201 Created`):**
```json
{
  "id": 1,
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal",
  "status": "recebido"
}
```
* **Resposta de Erro de Validação (`HTTP 400 Bad Request`):**
```json
{
  "erro": "O campo "valor" deve ser numérico e maior que zero."
}
```

### 4.2. Listar Pedidos (RF02)
* **Método e Rota:** `GET /pedidos`
* **Resposta de Sucesso (`HTTP 200 OK`):**
```json
[
  {
    "id": 1,
    "cliente": "Carlos Silva",
    "tipo": "entrega",
    "valor": 350,
    "prioridade": "normal",
    "status": "recebido"
  }
]
```

### 4.3. Consultar Pedido por ID (RF03)
* **Método e Rota:** `GET /pedidos/:id`
* **Resposta de Sucesso (`HTTP 200 OK`):**
```json
{
  "id": 1,
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal",
  "status": "recebido"
}
```
* **Resposta para ID Inexistente (`HTTP 404 Not Found`):**
```json
{
  "erro": "Pedido não encontrado."
}
```

### 4.4. Atualizar Status do Pedido (RF04)
* **Método e Rota:** `PATCH /pedidos/:id/status`
* **Corpo da Requisição (JSON):**
```json
{
  "status": "cancelado"
}
```
* **Resposta de Sucesso (`HTTP 200 OK`):**
```json
{
  "id": 1,
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal",
  "status": "cancelado"
}
```

---

## 5. Evidências de Testes Manuais (Postman)

Na pasta `evidencias/` encontram-se as capturas de tela dos cenários obrigatórios executados via Postman:
* `01-criacao.png`: `POST /pedidos` válido retornando `HTTP 201`.
* `02-validacao.png`: `POST /pedidos` com dados inválidos retornando `HTTP 400`.
* `03-listagem.png`: `GET /pedidos` retornando a lista de pedidos com `HTTP 200`.
* `04-consulta-inexistente.png`: `GET /pedidos/999` retornando `HTTP 404`.
* `05-atualizacao.png`: `PATCH /pedidos/1/status` válido retornando `HTTP 200`.

---

## 6. Uso de IA

Durante o desenvolvimento deste projeto, ferramentas de Inteligência Artificial generativa foram utilizadas como apoio para:
1. Revisão da modelagem das regras de transição de status na camada `service`.
2. Estruturação dos casos de teste automatizados com `Jest` e `Supertest`.
3. Revisão ortográfica e formatação da documentação no `README.md`.

### Registro crítico de sugestão (Modificada / Rejeitada)
* **Sugestão inicial da IA:** Inicialmente, a IA sugeriu realizar as validações de campos (`cliente`, `valor`, `tipo` e `prioridade`) diretamente dentro do `pedidoController.js` retornando `res.status(400).json(...)` em múltiplos blocos `if/else`, além de utilizar `Number(valor)` antes de validar o campo `valor`.
* **Decisão técnica (Rejeitada e Modificada):** A sugestão foi **rejeitada e modificada** por dois motivos técnicos:
  1. **Arquitetura em camadas:** Manter regras de validação e de transição de status no `Controller` violaria a separação de responsabilidades, acoplando regras de negócio à camada HTTP. As validações foram movidas integralmente para `pedidoService.js` utilizando uma classe de erro de domínio (`AppError`).
  2. **Validação estrita de tipos:** Converter o campo `valor` com `Number(valor)` permitiria que strings numéricas (como `"350"`) ou valores booleanos (`true` convertido para `1`) passassem indevidamente pela validação. A implementação final foi ajustada para exigir explicitamente `typeof valor === 'number' && !Number.isNaN(valor) && valor > 0`.

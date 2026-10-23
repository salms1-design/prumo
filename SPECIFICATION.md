# Especificação Técnica: Aplicação FullStack de Marketplace de Serviços (Prumo)

---

## 1. Visão Geral e Stack

### 1.1 Objetivo
O **Prumo** é uma aplicação FullStack com Django responsável por conectar **clientes** que precisam resolver serviços do dia a dia (elétrica, jardinagem, limpeza, pequenos reparos, etc.) a **profissionais autônomos** disponíveis para realizá-los. O sistema permite que o cliente publique um pedido de serviço, receba propostas de profissionais, negocie via chat interno, aceite uma proposta, acompanhe o andamento até a conclusão e avalie o profissional ao final.

Um princípio central do produto, definido a partir da análise de mercado (GetNinjas, Cronoshare, GuiaFix, Triider, Parafuzo, TaskRabbit): **o cliente pode navegar, ver categorias, ver pedidos de exemplo e entender a plataforma antes de precisar se cadastrar.** O cadastro completo (incluindo CPF) só é exigido no momento de publicar um pedido de serviço ou enviar uma proposta paga.

### 1.2 Stack Tecnológica
* **Linguagem:** Python 3.11+
* **Framework Web:** Django usando o ORM para gerar em PostgreSQL 16+ e a view sendo feita por Django Template com Bootstrap
* **Persistência / ORM:** PostgreSQL 16+
* **Validação de Dados:** Pydantic v2
* **Testes Unitários:** Usar o módulo `django.test` para criar os testes unitários da aplicação
* **Containerização:** Docker e Docker Compose

### 1.3 Arquitetura e Restrições
* **Padrão Arquitetural:** Usar a arquitetura padrão do Django MVT (Model View Template)
  1. `Domain`: Entidades puras, objetos de valor e exceções de domínio.
  2. `Use Cases`: Regras de negócio da aplicação e orquestração (ex: aceitar proposta e rejeitar as demais automaticamente).
  3. `Adapters / Controllers`: Controllers do Django, Models do Django.
  4. `Infrastructure`: Repositórios dos Models do Django, integração com banco de dados e configurações externas (ex: gateway de pagamento da assinatura).
* **Restrição de Camadas:** É proibido interpor as camadas da MVT do Django.

---

## 2. Entidades e Dados

### 2.0 Formato de Texto - Markdown Support com Campos Expandidos

Todos os campos `TEXT` (descrições de pedidos de serviço e propostas) suportam **Markdown** para formatação de texto rico:
- **Negrito**: `**texto**`
- *Itálico*: `*texto*`
- `Código`: `` `código` ``
- Listas: `- Item 1`, `- Item 2`
- Links: `[texto](url)`

#### Dimensões dos Campos de Descrição

| Campo | Altura Mínima | Linhas | Redimensionável | Markdown |
|-------|:---:|:---:|:---:|:---:|
| Pedido de Serviço (Descrição) | 300px | ~15 | ✅ Vertical | ✅ Sim |
| Proposta (Mensagem) | 150px | ~7 | ✅ Vertical | ✅ Sim |

Os campos TEXT são armazenados como markdown puro no BD (até 65KB por campo).

### 2.0.1 Formato de Datas (Scheduled Date)

A `scheduled_date` (data agendada para execução do serviço) é um campo **OPCIONAL** que usa o tipo `DATE` (sem hora):
* **Formato:** `YYYY-MM-DD` (ex: `"2026-12-31"`)
* **Armazenamento:**
  - **PostgreSQL/SQL:** Tipo `DATE`
  - **Django ORM:** `models.DateField(null=True, blank=True)`
  - **Python:** Tipo `date` do módulo `datetime` (ou `None`)
* **Validação:**
  - Quando fornecida, deve ser **posterior ao dia atual** (não aceita datas passadas nem a data de hoje)
  - Campo vazio no formulário é equivalente a `null` no banco de dados
  - Pode ser preenchida na aceitação da proposta ou ajustada posteriormente

### 2.1 Modelo de Dados (DDL SQL)

```sql
-- Tabela de Usuários
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    cpf VARCHAR(14), -- opcional na criação da conta; exigido apenas ao publicar um pedido ou enviar proposta paga
    city VARCHAR(100),
    state VARCHAR(2),
    role VARCHAR(20) NOT NULL DEFAULT 'CLIENT', -- 'CLIENT', 'PROFESSIONAL', 'ADMIN'
    average_rating NUMERIC(3,2) DEFAULT NULL, -- calculado a partir de reviews recebidas
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Categorias de Serviço
CREATE TABLE service_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(80) NOT NULL, -- ex: 'Elétrica', 'Jardinagem', 'Limpeza', 'Reparos Gerais'
    slug VARCHAR(80) UNIQUE NOT NULL,
    icon VARCHAR(50), -- nome do ícone exibido na UI
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Pedidos de Serviço
CREATE TABLE service_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    category_id UUID NOT NULL REFERENCES service_categories(id) ON DELETE RESTRICT,
    title VARCHAR(120) NOT NULL,
    description TEXT,
    address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(2),
    budget_estimate NUMERIC(10,2), -- opcional, valor de referência informado pelo cliente
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN', -- 'OPEN', 'IN_NEGOTIATION', 'SCHEDULED', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED'
    accepted_proposal_id UUID, -- preenchido quando uma proposta é aceita (ver seção 2.2)
    scheduled_date DATE, -- data combinada para execução (opcional)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Propostas
CREATE TABLE proposals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_request_id UUID NOT NULL REFERENCES service_requests(id) ON DELETE CASCADE,
    professional_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    price NUMERIC(10,2) NOT NULL,
    message TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING', -- 'PENDING', 'ACCEPTED', 'REJECTED', 'WITHDRAWN'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (service_request_id, professional_id) -- um profissional só pode ter uma proposta ativa por pedido
);

-- Tabela de Mensagens (negociação via chat dentro do pedido)
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_request_id UUID NOT NULL REFERENCES service_requests(id) ON DELETE CASCADE,
    sender_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    content TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Avaliações
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_request_id UUID NOT NULL REFERENCES service_requests(id) ON DELETE CASCADE,
    reviewer_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    reviewee_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    rating SMALLINT NOT NULL, -- 1 a 5
    comment TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (service_request_id, reviewer_id) -- um review por pessoa por pedido
);

-- Tabela de Assinaturas (monetização do profissional)
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    professional_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan VARCHAR(20) NOT NULL DEFAULT 'FREE_TRIAL', -- 'FREE_TRIAL', 'MONTHLY'
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE', -- 'ACTIVE', 'EXPIRED', 'CANCELLED'
    price NUMERIC(10,2), -- NULL durante o FREE_TRIAL
    start_date DATE NOT NULL DEFAULT CURRENT_DATE,
    end_date DATE NOT NULL, -- calculado: start_date + 30 dias
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabela de Histórico de Auditoria de Pedidos de Serviço
CREATE TABLE service_request_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service_request_id UUID NOT NULL REFERENCES service_requests(id) ON DELETE CASCADE,
    changed_by_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    field_name VARCHAR(50) NOT NULL, -- 'title', 'description', 'status', 'accepted_proposal_id', 'scheduled_date'
    old_value TEXT,
    new_value TEXT,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 2.2 Relacionamentos
* **User -> ServiceRequest:** 1 Usuário (cliente) pode publicar N Pedidos de Serviço (1:N).
* **ServiceCategory -> ServiceRequest:** 1 Categoria possui N Pedidos de Serviço (1:N).
* **ServiceRequest -> Proposal:** 1 Pedido de Serviço recebe N Propostas de N profissionais diferentes (1:N). Ao excluir um Pedido, suas Propostas são excluídas em cascata (`CASCADE`).
* **User -> Proposal:** 1 Profissional pode enviar N Propostas, mas apenas 1 por Pedido de Serviço (restrição `UNIQUE`).
* **ServiceRequest -> Message:** 1 Pedido de Serviço possui N Mensagens de chat (1:N), trocadas entre o cliente e os profissionais que propuseram naquele pedido.
* **ServiceRequest -> Review:** 1 Pedido de Serviço concluído gera até 2 Avaliações (cliente avalia profissional e profissional avalia cliente).
* **User -> Subscription:** 1 Profissional possui N Assinaturas ao longo do tempo (histórico de planos), mas apenas 1 `ACTIVE` por vez.
* **`accepted_proposal_id`:** referencia a Proposta aceita para aquele Pedido; quando preenchido, todas as demais Propostas `PENDING` daquele pedido são automaticamente movidas para `REJECTED` (ver RN-06).

### 2.3 Status do Pedido de Serviço

* O campo `status` aceita `OPEN`, `IN_NEGOTIATION`, `SCHEDULED`, `IN_PROGRESS`, `COMPLETED` ou `CANCELLED`, iniciando em `OPEN`.
* Transições permitidas:
  * `OPEN` -> `IN_NEGOTIATION` (automática, assim que a primeira proposta é recebida)
  * `IN_NEGOTIATION` -> `SCHEDULED` (quando o cliente aceita uma proposta, ver RN-06)
  * `SCHEDULED` -> `IN_PROGRESS` (quando o profissional inicia o serviço)
  * `IN_PROGRESS` -> `COMPLETED` (quando o serviço é concluído e confirmado pelo cliente)
  * `OPEN` -> `CANCELLED`, `IN_NEGOTIATION` -> `CANCELLED`, `SCHEDULED` -> `CANCELLED` (o cliente pode cancelar antes da conclusão)
* `COMPLETED` e `CANCELLED` são estados finais e não podem retornar para outro status.

### 2.4 Categorias de Serviço (seed inicial)

A tabela `service_categories` deve ser populada via *fixture* ou *migration de dados* com categorias iniciais, entre elas: Elétrica, Hidráulica, Jardinagem / Capina, Limpeza e Faxina, Montagem de Móveis, Pintura, Reparos Gerais, Chaveiro / Fechaduras, Afiação de Ferramentas, Outros.

### 2.5 Autenticação e Funções de Usuário

* O acesso à interface web deve usar a autenticação nativa do Django (`django.contrib.auth`), com sessão, login por e-mail e senha e logout.
* O usuário de autenticação do Django deve ser criado com `username` igual ao e-mail; a senha deve ser armazenada somente pelo mecanismo de hash do Django.
* O perfil de domínio (`users`) deve manter a função do usuário. Os valores permitidos são `CLIENT`, `PROFESSIONAL` e `ADMIN`.
* **Cadastro em duas etapas (diferencial de produto frente à concorrência):**
  1. **Navegação livre:** qualquer visitante pode ver categorias, pedidos de serviço abertos (sem dados de contato) e o funcionamento geral da plataforma sem se cadastrar.
  2. **Cadastro mínimo:** para publicar um pedido, enviar uma proposta ou trocar mensagens, o usuário cria conta apenas com nome, e-mail, senha e função (`CLIENT` ou `PROFESSIONAL`).
  3. **Dados completos (CPF, telefone, endereço):** exigidos apenas no momento de: (a) o cliente aceitar uma proposta (para viabilizar o contato direto), ou (b) o profissional enviar sua primeira proposta paga (fora do período de teste gratuito).
* Para inicializar uma base sem usuários, disponibilizar o comando:
  `python manage.py create_prumo_user --name "Nome" --email usuario@exemplo.com --password "senha-segura" --role PROFESSIONAL`.

### 2.6 Assinatura e Monetização

* Todo usuário `PROFESSIONAL` recém-cadastrado recebe automaticamente uma `Subscription` com `plan = 'FREE_TRIAL'`, `status = 'ACTIVE'`, `price = NULL`, `start_date = hoje` e `end_date = hoje + 30 dias`.
* Ao final do período de teste (`end_date` atingido), um job agendado (`python manage.py expire_trials`) marca a assinatura como `EXPIRED`.
* Um profissional com assinatura `EXPIRED` não pode enviar novas propostas (RN-05) até assinar o plano `MONTHLY` (valor de referência: R$ 20,00/mês, sem cobrança por lead ou por contato).
* O plano `MONTHLY` dá acesso ilimitado ao envio de propostas e ao chat com clientes, sem custo adicional por conversa — em contraste direto com o modelo de "moedas por contato" de concorrentes como GetNinjas e Cronoshare.

---

## 3. Interface Gráfica WEB (Django Templates + Bootstrap 5)

A aplicação disponibiliza uma interface visual completa renderizada via Django Templates e estilizada com Bootstrap 5:

### 3.1 Página Inicial / Vitrine Pública (`GET /`)
* **Rota:** `/` — acessível **sem login**, exibe categorias de serviço e pedidos de serviço abertos na região do visitante (sem exibir dados de contato do cliente).
* **Pré-requisito de inicialização:** Antes de acessar a rota, executar `python manage.py migrate`. A migration inicial `prumo.0001_initial` cria as tabelas `users`, `service_categories`, `service_requests`, `proposals`, `messages`, `reviews` e `subscriptions`.
* **Recursos visuais:**
  * **Grade de Categorias:** ícones clicáveis para cada categoria de serviço.
  * **Lista de Pedidos Abertos:** cartões com título, categoria, cidade/estado, faixa de orçamento (se informada) e badge de status `OPEN` — sem exibir nome completo, telefone ou e-mail do cliente até o profissional se cadastrar.
  * **Chamada para Cadastro:** call-to-action contextual ("Cadastre-se para propor este serviço" / "Publique seu pedido grátis").

### 3.2 Dashboard do Cliente (`GET /dashboard/client`)
* **Recursos visuais:**
  * **Meus Pedidos:** colunas por status (`OPEN`, `IN_NEGOTIATION`, `SCHEDULED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`).
  * **Propostas Recebidas:** lista de propostas por pedido, com preço, mensagem do profissional, nota média do profissional e botões *"Aceitar"* e *"Recusar"*.
  * **Chat:** acesso à conversa vinculada a cada pedido em negociação.
  * **Modal de Criação de Pedido de Serviço** (descrição expandida 300px min-height, redimensionável, suporta Markdown, com seleção de categoria, endereço e orçamento estimado opcional).
  * **Modal de Avaliação:** exibido automaticamente quando um pedido muda para `COMPLETED`, solicitando nota (1-5) e comentário sobre o profissional.

### 3.3 Dashboard do Profissional (`GET /dashboard/professional`)
* **Recursos visuais:**
  * **Barra de Assinatura:** status do plano atual (`FREE_TRIAL` com contagem regressiva de dias restantes, ou `MONTHLY` ativo), com botão *"Assinar"* quando `EXPIRED`.
  * **Pedidos Disponíveis:** lista filtrável por categoria e cidade, com botão *"Enviar Proposta"* (bloqueado com aviso caso a assinatura esteja `EXPIRED`).
  * **Minhas Propostas:** lista de propostas enviadas com status (`PENDING`, `ACCEPTED`, `REJECTED`, `WITHDRAWN`) e botão *"Retirar Proposta"* (disponível apenas em `PENDING`).
  * **Agenda:** pedidos com status `SCHEDULED` e `IN_PROGRESS`, com botão *"Iniciar Serviço"* e *"Concluir Serviço"*.

### 3.4 Ações e Formulários Web
* **Cadastrar Usuário:** `POST /web/users` (Campos: `name`, `email`, `password`, `password_confirmation`, `role`).
* **Entrar:** `POST /login` (Campos: `username` com o e-mail e `password`).
* **Sair:** `GET /logout`.
* **Publicar Pedido de Serviço:** `POST /web/service-requests` (Campos: `category_id`, `title`, `description`, `address`, `city`, `state`, `budget_estimate` [opcional]).
* **Editar Pedido de Serviço:** `POST /web/service-requests/{service_request_id}` (Campos opcionais — atualização parcial: apenas permitido enquanto status é `OPEN` ou `IN_NEGOTIATION`).
* **Cancelar Pedido de Serviço:** `POST /web/service-requests/{service_request_id}/cancel` (Sem campos; bloqueado se status já for `COMPLETED` ou `CANCELLED`).
* **Enviar Proposta:** `POST /web/service-requests/{service_request_id}/proposals` (Campos: `price`, `message`). Bloqueado se o profissional não tiver assinatura `ACTIVE` (RN-05).
* **Retirar Proposta:** `POST /web/proposals/{proposal_id}/withdraw` (Sem campos; disponível apenas se `status = PENDING`).
* **Aceitar Proposta:** `POST /web/proposals/{proposal_id}/accept` (Campo opcional: `scheduled_date`). Dispara RN-06.
* **Recusar Proposta:** `POST /web/proposals/{proposal_id}/reject` (Sem campos).
* **Enviar Mensagem:** `POST /web/service-requests/{service_request_id}/messages` (Campo: `content`).
* **Iniciar Serviço:** `POST /web/service-requests/{service_request_id}/start` (Sem campos; `SCHEDULED` -> `IN_PROGRESS`).
* **Concluir Serviço:** `POST /web/service-requests/{service_request_id}/complete` (Sem campos; `IN_PROGRESS` -> `COMPLETED`).
* **Avaliar:** `POST /web/service-requests/{service_request_id}/reviews` (Campos: `rating`, `comment`).
* **Assinar Plano Mensal:** `POST /web/subscriptions` (Campo: `plan = 'MONTHLY'`). Integração de pagamento tratada na camada de Infrastructure.

---

## 4. Contratos de API REST (v1)

### 4.1 Publicar Pedido de Serviço
* **Endpoint:** `POST /api/v1/service-requests`
* **Headers:** `Content-Type: application/json`
* **Request Body:**
```json
{
  "category_id": "b14e7bd1-6c42-4e27-9e94-19f79a8061a9",
  "title": "Capinar lote de 300m²",
  "description": "Terreno com mato alto, precisa de roçadeira.",
  "address": "Rua das Palmeiras, 120",
  "city": "Maceió",
  "state": "AL",
  "budget_estimate": 150.00
}
```
* **Respostas:**
  * `201 Created`:
```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "client_id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "category_id": "b14e7bd1-6c42-4e27-9e94-19f79a8061a9",
  "title": "Capinar lote de 300m²",
  "description": "Terreno com mato alto, precisa de roçadeira.",
  "status": "OPEN",
  "created_at": "2026-09-24T08:00:00Z"
}
```
  * `400 Bad Request`: Categoria não encontrada.
  * `422 Unprocessable Entity`: Dados de entrada inválidos.

### 4.1a Editar Pedido de Serviço
* **Endpoint:** `PUT /api/v1/service-requests/{service_request_id}` ou `PATCH /api/v1/service-requests/{service_request_id}`
* **Request Body (todos os campos opcionais):**
```json
{
  "title": "Novo título",
  "description": "Nova descrição.",
  "budget_estimate": 200.00
}
```
* **Respostas:**
  * `200 OK`: Retorna o objeto completo atualizado.
  * `404 Not Found`: Pedido não encontrado.
  * `409 Conflict`: Pedido já está `SCHEDULED`, `IN_PROGRESS`, `COMPLETED` ou `CANCELLED` e não pode mais ser editado.
  * `422 Unprocessable Entity`: Dados de entrada inválidos.

### 4.1b Cancelar Pedido de Serviço
* **Endpoint:** `POST /api/v1/service-requests/{service_request_id}/cancel`
* **Respostas:**
  * `200 OK`: Retorna o objeto com `status = "CANCELLED"`.
  * `404 Not Found`: Pedido não encontrado.
  * `409 Conflict`: Pedido já `COMPLETED` ou `CANCELLED`.

---

### 4.2 Enviar Proposta
* **Endpoint:** `POST /api/v1/service-requests/{service_request_id}/proposals`
* **Request Body:**
```json
{
  "price": 120.00,
  "message": "Consigo ir amanhã de manhã, tenho roçadeira própria."
}
```
* **Respostas:**
  * `201 Created`:
```json
{
  "id": "c9bf9e57-1685-4c89-bafb-ff5af830be8a",
  "service_request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "professional_id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "price": 120.00,
  "message": "Consigo ir amanhã de manhã, tenho roçadeira própria.",
  "status": "PENDING",
  "created_at": "2026-09-24T09:00:00Z"
}
```
  * `403 Forbidden`: Assinatura do profissional está `EXPIRED` (código `SUBSCRIPTION_REQUIRED`).
  * `404 Not Found`: Pedido de serviço não encontrado.
  * `409 Conflict`: Profissional já possui uma proposta ativa para este pedido.
  * `422 Unprocessable Entity`: Preço inválido (ex: menor ou igual a zero).

### 4.2a Aceitar Proposta
* **Endpoint:** `POST /api/v1/proposals/{proposal_id}/accept`
* **Request Body (opcional):**
```json
{
  "scheduled_date": "2026-10-05"
}
```
* **Descrição:** Aceita a proposta, move o pedido para `SCHEDULED` e rejeita automaticamente as demais propostas `PENDING` do mesmo pedido (RN-06).
* **Respostas:**
  * `200 OK`: Retorna a proposta com `status = "ACCEPTED"`.
  * `404 Not Found`: Proposta não encontrada.
  * `409 Conflict`: Pedido já possui outra proposta aceita, ou não está em `IN_NEGOTIATION`.

### 4.2b Retirar Proposta
* **Endpoint:** `POST /api/v1/proposals/{proposal_id}/withdraw`
* **Respostas:**
  * `200 OK`: Retorna a proposta com `status = "WITHDRAWN"`.
  * `404 Not Found`: Proposta não encontrada.
  * `409 Conflict`: Proposta já foi `ACCEPTED` ou `REJECTED`.

---

### 4.3 Mensagens
* **Listar / Enviar:** `GET /api/v1/service-requests/{service_request_id}/messages` e `POST /api/v1/service-requests/{service_request_id}/messages`
* **Request Body (envio):**
```json
{
  "content": "Pode ser às 9h da manhã?"
}
```
* **Respostas:**
  * `201 Created`: Retorna a mensagem criada.
  * `403 Forbidden`: Remetente não é o cliente do pedido nem um profissional com proposta ativa nele.
  * `404 Not Found`: Pedido de serviço não encontrado.

---

### 4.4 Ciclo de Vida do Serviço
* **Iniciar:** `PATCH /api/v1/service-requests/{service_request_id}/status` com `{"status": "IN_PROGRESS"}`
* **Concluir:** `PATCH /api/v1/service-requests/{service_request_id}/status` com `{"status": "COMPLETED"}`
* **Respostas:**
  * `200 OK`: Retorna o objeto completo com o novo status.
  * `400 Bad Request`: Transição de status inválida (código `INVALID_STATUS_TRANSITION`).
  * `404 Not Found`: Pedido de serviço não encontrado.

---

### 4.5 Avaliações
* **Endpoint:** `POST /api/v1/service-requests/{service_request_id}/reviews`
* **Request Body:**
```json
{
  "reviewee_id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "rating": 5,
  "comment": "Rápido, educado e resolveu na hora."
}
```
* **Respostas:**
  * `201 Created`: Retorna a avaliação criada e atualiza `average_rating` do avaliado.
  * `400 Bad Request`: Pedido ainda não está `COMPLETED`.
  * `404 Not Found`: Pedido de serviço não encontrado.
  * `409 Conflict`: Avaliador já avaliou este pedido (código `DUPLICATE_REVIEW`).
  * `422 Unprocessable Entity`: `rating` fora do intervalo 1-5.

---

### 4.6 Assinaturas
* **Endpoint:** `POST /api/v1/subscriptions`
* **Request Body:**
```json
{
  "plan": "MONTHLY"
}
```
* **Respostas:**
  * `201 Created`: Retorna a assinatura criada com `status = "ACTIVE"`.
  * `409 Conflict`: Profissional já possui assinatura `ACTIVE`.
* **Consultar Assinatura Atual:** `GET /api/v1/subscriptions/current`
  * `200 OK`: Retorna a assinatura vigente (`FREE_TRIAL` ou `MONTHLY`) com `end_date` e dias restantes.

---

## 5. Regras de Negócio e Casos de Borda

### 5.1 Regras de Negócio (RN)
* **RN-01 (Validação de Título):** O título do pedido de serviço deve conter entre 3 e 120 caracteres e não pode ser composto apenas por espaços em branco.
* **RN-02 (Preço da Proposta):** O `price` de uma proposta deve ser um valor positivo (`> 0`).
* **RN-03 (Valores Permitidos de Enums):**
  * `service_requests.status`: Apenas `OPEN`, `IN_NEGOTIATION`, `SCHEDULED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`.
  * `proposals.status`: Apenas `PENDING`, `ACCEPTED`, `REJECTED`, `WITHDRAWN`.
  * `subscriptions.plan`: Apenas `FREE_TRIAL`, `MONTHLY`.
* **RN-04 (Ciclo de Vida do Pedido de Serviço):**
  * `OPEN` -> `IN_NEGOTIATION` (automática ao receber a primeira proposta)
  * `IN_NEGOTIATION` -> `SCHEDULED` (ao aceitar uma proposta)
  * `SCHEDULED` -> `IN_PROGRESS` -> `COMPLETED`
  * `OPEN`, `IN_NEGOTIATION` e `SCHEDULED` -> `CANCELLED` (cancelamento pelo cliente)
  * `COMPLETED` e `CANCELLED` são estados finais.
* **RN-05 (Assinatura Ativa para Propor):** Um profissional só pode enviar uma nova proposta se possuir uma `Subscription` com `status = 'ACTIVE'` (seja `FREE_TRIAL` ou `MONTHLY`). Caso contrário, a API retorna `403 Forbidden` com código `SUBSCRIPTION_REQUIRED`.
* **RN-06 (Aceite de Proposta):** Ao aceitar uma proposta (`accepted_proposal_id` preenchido), o sistema deve, na mesma transação: (a) mover a proposta aceita para `ACCEPTED`; (b) mover todas as demais propostas `PENDING` do mesmo pedido para `REJECTED`; (c) mover o pedido de serviço para `SCHEDULED`.
* **RN-07 (Acesso ao Chat):** Apenas o cliente dono do pedido e os profissionais que enviaram proposta (em qualquer status) para aquele pedido podem ler e enviar mensagens em sua conversa.
* **RN-08 (Avaliação Restrita a Participantes):** Uma avaliação só pode ser criada se o pedido estiver `COMPLETED`, o avaliador for o cliente do pedido ou o profissional com a proposta `ACCEPTED`, e ainda não existir avaliação daquele avaliador para aquele pedido (`UNIQUE (service_request_id, reviewer_id)`).
* **RN-09 (Cadastro em Camadas):** O cadastro inicial (nome, e-mail, senha, função) não exige CPF, CEP ou telefone. Esses dados só são solicitados no fluxo de aceite de proposta (cliente) ou no envio da primeira proposta paga (profissional).
* **RN-10 (Expiração de Teste Gratuito):** Toda `Subscription` do tipo `FREE_TRIAL` expira automaticamente 30 dias após `start_date`, através do comando agendado `expire_trials`, passando para `status = 'EXPIRED'`.
* **RN-11 (Rastreamento de Auditoria):** Toda alteração relevante em um Pedido de Serviço (status, título, descrição, `accepted_proposal_id`, `scheduled_date`) é registrada na tabela `service_request_history` com usuário responsável, campo alterado, valor anterior/novo e data/hora.

### 5.2 Casos de Borda (CB)
* **CB-01 (Categoria Inexistente):** Publicar um pedido enviando `category_id` inexistente deve retornar `400 Bad Request` com mensagem `"Categoria não encontrada"`.
* **CB-02 (Proposta Duplicada):** Um profissional que já possui proposta `PENDING` ou `ACCEPTED` em um pedido não pode enviar uma segunda proposta para o mesmo pedido (`409 Conflict`).
* **CB-03 (Aceite com Assinatura Expirada Após o Envio):** Se a assinatura do profissional expirar entre o envio da proposta e o aceite pelo cliente, o aceite ainda é válido (a exigência de assinatura ativa se aplica apenas ao *enviar* a proposta, RN-05, não ao aceitá-la).
* **CB-04 (Cliente Tenta Aceitar Duas Propostas):** Tentar aceitar uma segunda proposta para um pedido que já está `SCHEDULED` deve retornar `409 Conflict`.
* **CB-05 (Edição Parcial do Pedido):** Editar um pedido enviando apenas `title` deve atualizar apenas esse campo, mantendo os demais inalterados.
* **CB-06 (Edição de Pedido Já Agendado):** Tentar editar título/descrição/orçamento de um pedido com status `SCHEDULED`, `IN_PROGRESS`, `COMPLETED` ou `CANCELLED` deve retornar `409 Conflict`.
* **CB-07 (Mensagem de Usuário Sem Vínculo):** Um profissional que nunca propôs naquele pedido tentando enviar mensagem deve retornar `403 Forbidden`.
* **CB-08 (Avaliação Antes da Conclusão):** Tentar avaliar um pedido que ainda não está `COMPLETED` deve retornar `400 Bad Request`.
* **CB-09 (Avaliação Duplicada):** Tentar criar uma segunda avaliação do mesmo avaliador para o mesmo pedido deve retornar `409 Conflict` com código `DUPLICATE_REVIEW`.
* **CB-10 (Proposta Sem Assinatura Ativa):** Um profissional com assinatura `EXPIRED` tentando enviar proposta deve retornar `403 Forbidden` com código `SUBSCRIPTION_REQUIRED`, e a interface web deve bloquear o botão *"Enviar Proposta"* exibindo aviso para assinar o plano mensal.
* **CB-11 (Cancelamento de Pedido Já Concluído):** Tentar cancelar um pedido `COMPLETED` deve retornar `409 Conflict`.
* **CB-12 (Retirada de Proposta Já Aceita):** Tentar retirar (`withdraw`) uma proposta já `ACCEPTED` ou `REJECTED` deve retornar `409 Conflict`.
* **CB-13 (Exclusão em Cascata do Pedido):** Excluir um pedido de serviço remove automaticamente (via FK `ON DELETE CASCADE`) todas as propostas e mensagens vinculadas a ele.
* **CB-14 (Segunda Assinatura Ativa):** Tentar criar uma nova `Subscription` enquanto já existe uma com `status = 'ACTIVE'` deve retornar `409 Conflict`.

---

## 6. Critérios de Aceite (Exemplos para Testes Automatizados)

Estes cenários devem orientar a geração de testes automatizados com `pytest` e `httpx`.

### Cenário 1: Sucesso na Publicação de Pedido de Serviço
* **Dado** que existe uma categoria cadastrada com `id = "b14e7bd1-6c42-4e27-9e94-19f79a8061a9"`
* **Quando** for enviada uma requisição `POST` para `/api/v1/service-requests` com o payload:
  ```json
  {
    "category_id": "b14e7bd1-6c42-4e27-9e94-19f79a8061a9",
    "title": "Consertar tomada com fiação exposta",
    "description": "Tomada da sala solta e dando choque.",
    "city": "Maceió",
    "state": "AL"
  }
  ```
* **Então** o código de status HTTP retornado deve ser `201 Created`
* **E** o corpo da resposta deve conter o campo `id` em formato UUID
* **E** o campo `status` deve ser inicializado automaticamente como `"OPEN"`.

### Cenário 2: Bloqueio de Proposta com Assinatura Expirada
* **Dado** um profissional cuja `Subscription` possui `status = "EXPIRED"`
* **Quando** for enviada uma requisição `POST` para `/api/v1/service-requests/{service_request_id}/proposals`
* **Então** o código de status HTTP retornado deve ser `403 Forbidden`
* **E** o corpo da resposta deve conter:
  ```json
  {
    "detail": "Sua assinatura expirou. Assine o plano mensal para continuar enviando propostas.",
    "code": "SUBSCRIPTION_REQUIRED"
  }
  ```

### Cenário 3: Aceite de Proposta Rejeita as Demais Automaticamente
* **Dado** um pedido de serviço `IN_NEGOTIATION` com 3 propostas `PENDING`
* **Quando** for enviada uma requisição `POST` para `/api/v1/proposals/{proposal_id}/accept` referente a uma delas
* **Então** o código de status HTTP retornado deve ser `200 OK`
* **E** a proposta informada deve estar `ACCEPTED`
* **E** as outras 2 propostas do mesmo pedido devem estar `REJECTED`
* **E** o pedido de serviço deve estar `SCHEDULED`

### Cenário 4: Falha por Avaliação Duplicada
* **Dado** um pedido `COMPLETED` que já foi avaliado pelo cliente
* **Quando** o mesmo cliente enviar uma nova requisição `POST` para `/api/v1/service-requests/{service_request_id}/reviews`
* **Então** o código de status HTTP retornado deve ser `409 Conflict`
* **E** o corpo da resposta deve conter `"code": "DUPLICATE_REVIEW"`

---

## 7. Requisitos Rastreáveis (Requirements)

Esta seção mantém a rastreabilidade entre os **Requisitos** do produto e as funcionalidades que os implementam, seguindo o mesmo padrão de rastreabilidade adotado no projeto de referência (código único, tipo, prioridade e status).

<!-- REQUISITOS:START -->
**RF-01 — Navegação pública sem cadastro obrigatório**
*Tipo: Requisito Funcional | Prioridade: HIGH | Status: DRAFT*

Tarefas vinculadas: _nenhuma tarefa vinculada._

**RF-02 — Envio e aceite de propostas com rejeição automática das demais**
*Tipo: Requisito Funcional | Prioridade: HIGH | Status: DRAFT*

Tarefas vinculadas: _nenhuma tarefa vinculada._

**RF-03 — Assinatura mensal sem cobrança por lead/contato**
*Tipo: Regra de Negócio | Prioridade: HIGH | Status: DRAFT*

Tarefas vinculadas: _nenhuma tarefa vinculada._
<!-- REQUISITOS:END -->

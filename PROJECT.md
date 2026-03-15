# PROJECT.md — Sistema São João Batista
> Fonte única de verdade para desenvolvimento do sistema de gestão de lanchonete familiar.
> Versão: 1.1 · Data: 2026-03-14

---

## 1. Visão Geral

**Sistema São João Batista** é um sistema web de gestão operacional desenvolvido para uso interno de uma lanchonete familiar de pequeno porte. O objetivo é substituir o controle manual (cadernos, calculadoras, planilhas) por uma interface digital simples, rápida e confiável.

### Contexto de Uso

| Aspecto | Detalhe |
|---|---|
| Porte | Pequena lanchonete familiar |
| Atendimento | Balcão presencial (local e retirada) + Delivery via WhatsApp/telefone |
| Usuários | 1 funcionário + 2 donos |
| Dispositivos | **Celular** (dispositivo principal); acesso via navegador |
| Perfil técnico | Baixo — interface deve ser extremamente simples |
| Acesso | Via navegador (sem instalação de app) |

### Princípios de UX

- **Poucos cliques**: operações críticas em no máximo 3 toques.
- **Mobile-first**: layout em coluna única em todas as telas, sem exceções.
- **Botões grandes**: altura mínima de 56 px para área de toque confortável em celular.
- **Sem tabelas com scroll horizontal**: listas de dados usam cards empilhados, não tabelas.
- **Teclado nativo**: campos de quantidade e preço ativam o teclado numérico nativo (`inputMode="numeric"`).
- **Navegação inferior fixa**: menu de navegação fixo na base da tela com ícones grandes e labels — sem hamburger menu.
- **Feedback imediato**: confirmação visual após cada ação.
- **Sem ambiguidade**: nomes claros, sem jargão técnico.
- **Resiliência**: a tela de pedidos não pode travar nem perder dados.

---

## 2. Stack Tecnológica

### 2.1 Framework — Next.js 14 (App Router) + TypeScript

**Motivo:** Full-stack em um único repositório. O App Router permite RSC (React Server Components) para carregamento rápido de dados sem precisar de uma API separada. TypeScript garante segurança de tipos e autocomplete, reduzindo bugs em produção.

### 2.2 Banco de Dados e Auth — Supabase (PostgreSQL + Supabase Auth)

**Motivo:** Plano gratuito generoso (500 MB de armazenamento, 50.000 usuários ativos/mês). PostgreSQL é maduro e confiável. Supabase Auth entrega login com email/senha sem código adicional. O cliente JS do Supabase funciona nativamente com Next.js.

### 2.3 UI — shadcn/ui + Tailwind CSS

**Motivo:** shadcn/ui fornece componentes acessíveis e prontos (botões, modais, cards) que podem ser copiados para o projeto e customizados. Tailwind elimina a necessidade de arquivos CSS separados e acelera a prototipação. A combinação é o padrão de facto para projetos Next.js modernos.

### 2.4 Deploy — Vercel (plano hobby / gratuito)

**Motivo:** Integração nativa com Next.js (mesma empresa). Deploy automático a cada push no GitHub. HTTPS grátis, CDN global, sem configuração de servidor.

### 2.5 Resumo da Stack

```
Next.js 14 (App Router)
├── TypeScript
├── Tailwind CSS
├── shadcn/ui
└── Supabase
    ├── PostgreSQL (banco de dados)
    └── Supabase Auth (autenticação)

Deploy: Vercel (hobby)
Repositório: GitHub
```

---

## 3. Navegação

### Tela Inicial

A tela inicial após login é `/pedidos` para **todos** os perfis (funcionário e dono).

### Menu de Navegação Inferior Fixo

O menu de navegação é fixado na base da tela, ocupa toda a largura e exibe **ícone + label** para cada item. Não há hamburger menu.

| Item | Ícone | Rota | Perfis com acesso |
|---|---|---|---|
| Pedidos | `clipboard` | `/pedidos` | todos |
| Estoque | `package` | `/estoque` | todos |
| Cardápio | `book` | `/cardapio` | todos (edição restrita a donos) |
| Painel | `bar-chart` | `/painel` | somente donos |

- O item **Painel** é renderizado e acessível apenas para usuários com `role = 'dono'`. Funcionários não veem esse item no menu.
- Em celular, os 4 itens dividem igualmente a largura da tela com ícone centralizado acima do label.

---

## 4. Módulos do Sistema

### Módulo 0 — Gestão de Cardápio

**Objetivo:** Manter o catálogo de produtos vendidos pela lanchonete. Funcionários podem consultar; apenas donos podem criar, editar ou desativar produtos.

#### Telas

##### 4.0.1 Lista de Produtos (`/cardapio`)

- Listagem de todos os produtos com: nome, preço, categoria e status ativo/inativo.
- Cada card exibe um **toggle** para ativar/desativar o produto diretamente na lista (visível e clicável apenas para donos; somente leitura para funcionários).
- Botão **"+ Novo Produto"** visível apenas para donos.
- Toque no card abre a tela de edição (somente para donos).

##### 4.0.2 Criar/Editar Produto (`/cardapio/novo` e `/cardapio/[id]`)

- Campos: **Nome**, **Preço** (teclado numérico), **Categoria** (texto livre), **Ativo** (toggle).
- Botão **Salvar** e, na edição, botão **Desativar Produto** (que define `active = false`).
- Exclusão física não é permitida — apenas desativação.

#### Regras de Negócio — Módulo 0

- Produtos não são excluídos fisicamente para preservar histórico de pedidos; `active = false` os remove do cardápio operacional.
- Categoria é um campo de texto livre — não há tabela separada de categorias.
- Funcionários têm acesso de leitura ao cardápio; criação e edição são restritas a donos.
- O preço de um produto pode ser alterado a qualquer momento sem afetar pedidos já registrados.

---

### Módulo 1 — Registro de Pedidos ⭐ (Prioridade Máxima)

**Objetivo:** Tela operacional usada pelo funcionário dezenas de vezes por dia para registrar todas as vendas. Um pedido só pode ser fechado quando o cliente for completamente atendido e o pagamento for confirmado.

#### Telas

##### 4.1.1 Tela Principal de Pedidos (`/pedidos`)

- Lista dos pedidos ativos do dia (status: `aberto` ou `em_preparo`).
- Botão grande **"+ Novo Pedido"** no topo.
- Cada pedido exibe: número sequencial do dia, tipo (Delivery / Retirada / Local), total parcial, status.
- Toque no card do pedido abre o detalhe/edição.

> **Fluxo de criação:** ao tocar em "+ Novo Pedido", o app exibe um modal simples de seleção de tipo com três opções: **Delivery**, **Retirada** e **Local**. Após a seleção, o app dispara `POST /api/orders` com o `type` já definido, cria o registro com `status = 'aberto'` no banco e redireciona para `/pedidos/[id]` do pedido recém-criado. O modal garante que `type` nunca chega ao servidor como nulo. Não existe rota de página `/pedidos/novo`.

##### 4.1.2 Criar/Editar Pedido (`/pedidos/[id]`)

- **Tipo do pedido:** seleção visual entre "Delivery", "Retirada" e "Local" (botões grandes com ícone).
- **Campo nome** (opcional, mais relevante para delivery): nome do cliente.
- **Cardápio:** grade de produtos agrupados por categoria. Cada produto exibe nome e preço. Toque adiciona 1 unidade; há botões `+` e `−` para ajustar quantidade.
- **Observações por item:** abaixo de cada item adicionado ao pedido, há um campo de texto opcional para observações do cliente (ex: "sem alface", "bem passado", "sem cebola"). O campo é colapsado por padrão e expandido ao toque.
- **Resumo do pedido:** seção inferior com itens selecionados, quantidades e subtotais.
- **Forma de pagamento:** seleção obrigatória antes de fechar — "Dinheiro", "Pix", "Cartão".
- **Botão "Fechar Pedido":** ativo somente quando ao menos 1 item foi adicionado, a forma de pagamento foi selecionada e o atendimento está completo.
- **Botão "Cancelar Pedido":** disponível enquanto o pedido está aberto.

#### Regras de Negócio — Módulo 1

- Um pedido só pode ser fechado quando o cliente for completamente atendido **e** o pagamento for confirmado (ao menos 1 item + forma de pagamento selecionada).
- O total do pedido é calculado no cliente (soma de `quantity × unit_price`) e validado novamente no servidor antes de persistir.
- O `unit_price` dos itens é capturado no momento do pedido (não muda com alterações futuras no cardápio).
- Pedidos cancelados ficam com status `cancelado` (soft delete — não são excluídos).
- Pedidos fechados não podem ser editados.
- O número sequencial do dia é calculado como: `COUNT(*) + 1` dos pedidos do dia (status ≠ `cancelado`).
- O tipo do pedido pode ser alterado enquanto o status for aberto ou em_preparo. Ao fechar o pedido, o tipo fica imutável junto com os demais campos.

---

### Módulo 2 — Painel do Dia

**Objetivo:** Tela gerencial para os donos acompanharem o desempenho em tempo real.

#### Telas

##### 4.2.1 Painel Gerencial (`/painel`)

- **Total do dia:** valor em destaque, grande.
- **Quantidade de pedidos** fechados no dia.
- **Ticket médio:** Total ÷ Quantidade de pedidos.
- **Breakdown por forma de pagamento:** Dinheiro / Pix / Cartão — valor e percentual de cada.
- **Ranking dos 5 produtos mais vendidos:** nome do produto, quantidade vendida, receita gerada.
- **Comparativo simples:** "vs. ontem" (opcional na V1 — porém a query suporta).
- Filtro de data (padrão: hoje). Seleção de outra data para ver dias anteriores.
- Layout em coluna única com fontes grandes (mobile-first).

#### Regras de Negócio — Módulo 2

- Considera apenas pedidos com status `fechado`.
- O painel é somente leitura — nenhuma ação é realizada a partir dele.
- Os dados são carregados a cada acesso (sem polling automático na V1).
- Acesso restrito a usuários com `role = 'dono'`.

---

### Módulo 3 — Controle de Estoque

**Objetivo:** Registro e monitoramento manual dos insumos para evitar falta de produtos.

> **Modelo mental de estoque:** somente itens fechados/não abertos são contabilizados. Um pacote aberto em uso não entra na contagem.

#### Telas

##### 4.3.1 Lista de Insumos (`/estoque`)

- Listagem de todos os insumos com: nome, quantidade atual, unidade (kg, un, L, etc.), quantidade mínima, indicador de perecível.
- Itens **abaixo do mínimo** (não perecíveis) são destacados visualmente (fundo vermelho claro / ícone de alerta).
- Itens perecíveis não exibem alerta de mínimo.
- Botão **"+ Novo Insumo"**.
- Cada card tem acesso rápido para **Baixa Rápida**, **Baixa por Deterioração** (apenas perecíveis) e **Editar**.

##### 4.3.2 Cadastro/Edição de Insumo (`/estoque/[id]`)

- Campos: Nome, Quantidade Atual, Unidade, Quantidade Mínima, Perecível (toggle).
- Salvar ou Excluir insumo.

##### 4.3.3 Registrar Ajuste de Estoque

- Modal ou seção inline com campo numérico para a quantidade.
- Seleção do motivo do ajuste: **Consumo**, **Deterioração** (apenas perecíveis), **Reposição**.
- Confirmação com preview do novo saldo.

#### Regras de Negócio — Módulo 3

- Quantidade nunca pode ficar negativa (validação no servidor).
- O alerta de mínimo é exibido quando `quantity <= min_quantity` **somente para itens não perecíveis** (`is_perishable = false`).
- Para itens perecíveis, `min_quantity` é ignorado e nenhum alerta é exibido.
- Não há integração automática com pedidos nesta versão (baixa sempre manual).
- A unidade de medida é um campo de texto livre (sem conversão entre unidades).
- Todo ajuste de estoque registra o motivo: `consumo`, `deterioracao` ou `reposicao`.
- Itens perecíveis possuem uma ação específica de "Baixa por Deterioração" além da baixa normal de consumo.

---

### Módulo 4 — Gestão de Usuários

**Objetivo:** Permitir que donos gerenciem os acessos ao sistema — criem novos usuários, atribuam perfis e desativem contas quando necessário. Acesso restrito exclusivamente a donos.

#### Telas

##### 4.4.1 Lista de Usuários (`/usuarios`)

- Listagem de todos os usuários com: nome, e-mail e role (`Funcionário` / `Dono`).
- Cada card indica visualmente se o acesso está ativo ou desativado.
- Botão **"+ Novo Usuário"** que abre um modal de criação na mesma tela.
- Ação **"Desativar Acesso"** disponível em cada card para revogar o login do usuário (sem excluir o registro).

##### 4.4.2 Modal de Criação de Usuário

- Campos: **Nome**, **E-mail**, **Perfil** (seleção entre `Funcionário` e `Dono`).
- O servidor cria o usuário no Supabase Auth (via service role key) e insere o registro em `profiles`.
- Senha inicial gerada automaticamente e enviada por e-mail (fluxo padrão do Supabase Auth).

#### Regras de Negócio — Módulo 4

- Apenas usuários com `role = 'dono'` podem acessar `/usuarios`.
- Desativar um usuário executa **duas ações obrigatórias em sequência**:
  1. Define `active = false` na tabela `profiles` (via `PATCH /api/users/[id]`).
  2. Chama `supabase.auth.admin.updateUserById(id, { ban_duration: '876000h' })` via service role key para banir o usuário no Supabase Auth, impedindo qualquer novo login.
- Reativar um usuário reverte ambas as ações: `active = true` em `profiles` e `ban_duration: 'none'` (remoção do ban) no Supabase Auth.
- Não é possível desativar o próprio acesso (prevenção de lock-out).
- Não é possível ter zero donos — o sistema deve rejeitar a desativação do último dono com `active = true`.

---

## 5. Modelo de Dados

### 5.1 Tabela `profiles`

```sql
CREATE TABLE profiles (
  id         UUID PRIMARY KEY REFERENCES auth.users(id),
  name       VARCHAR(100) NOT NULL,
  email      VARCHAR(255) NOT NULL,
  role       VARCHAR(20) NOT NULL DEFAULT 'funcionario'
             CHECK (role IN ('funcionario', 'dono')),
  active     BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Mesmo ID do usuário em `auth.users` |
| `name` | VARCHAR(100) | Nome de exibição do usuário |
| `email` | VARCHAR(255) | E-mail do usuário (denormalizado de `auth.users` para consulta direta) |
| `role` | VARCHAR(20) | `funcionario` ou `dono` |
| `active` | BOOLEAN | Se `false`, acesso revogado — login bloqueado no Supabase Auth |
| `created_at` | TIMESTAMPTZ | Data de cadastro |

---

### 5.2 Tabela `products`

```sql
CREATE TABLE products (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name       VARCHAR(100) NOT NULL,
  price      NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
  category   VARCHAR(50) NOT NULL DEFAULT 'Geral',
  active     BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `name` | VARCHAR(100) | Nome do produto exibido no cardápio |
| `price` | NUMERIC(10,2) | Preço de venda atual |
| `category` | VARCHAR(50) | Agrupamento no cardápio (ex: "Lanches", "Bebidas") — texto livre |
| `active` | BOOLEAN | Se `false`, não aparece no cardápio mas histórico é preservado |
| `created_at` | TIMESTAMPTZ | Data de cadastro |

---

### 5.3 Tabela `orders`

```sql
CREATE TABLE orders (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type           VARCHAR(10) NOT NULL CHECK (type IN ('delivery', 'retirada', 'local')),
  status         VARCHAR(15) NOT NULL DEFAULT 'aberto'
                   CHECK (status IN ('aberto', 'em_preparo', 'fechado', 'cancelado')),
  payment_method VARCHAR(10) CHECK (payment_method IN ('dinheiro', 'pix', 'cartao')),
  customer_name  VARCHAR(100),
  total          NUMERIC(10, 2) NOT NULL DEFAULT 0 CHECK (total >= 0),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  closed_at      TIMESTAMPTZ
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `type` | VARCHAR(10) | `delivery`, `retirada` ou `local` |
| `status` | VARCHAR(15) | `aberto`, `em_preparo`, `fechado`, `cancelado` |
| `payment_method` | VARCHAR(10) | Obrigatório ao fechar; `dinheiro`, `pix` ou `cartao` |
| `customer_name` | VARCHAR(100) | Nome do cliente (opcional, mais usado no delivery) |
| `total` | NUMERIC(10,2) | Soma de `order_items.quantity × unit_price` |
| `created_at` | TIMESTAMPTZ | Abertura do pedido |
| `closed_at` | TIMESTAMPTZ | Momento do fechamento (NULL enquanto aberto) |

**Tipos de atendimento:**
- `delivery` — pedido remoto entregue no endereço do cliente.
- `retirada` — cliente pede no balcão e leva embora.
- `local` — cliente pede no balcão e consome no espaço físico.

---

### 5.4 Tabela `order_items`

```sql
CREATE TABLE order_items (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id     UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id   UUID NOT NULL REFERENCES products(id),
  quantity     INTEGER NOT NULL CHECK (quantity > 0),
  unit_price   NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
  observations TEXT
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `order_id` | UUID | Referência ao pedido pai |
| `product_id` | UUID | Referência ao produto (preserva histórico) |
| `quantity` | INTEGER | Quantidade pedida |
| `unit_price` | NUMERIC(10,2) | Preço no momento do pedido (snapshot) |
| `observations` | TEXT | Observações do cliente para o item (nullable; ex: "sem alface", "bem passado") |

---

### 5.5 Tabela `stock_items`

```sql
CREATE TABLE stock_items (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name           VARCHAR(100) NOT NULL UNIQUE,
  quantity       NUMERIC(10, 3) NOT NULL DEFAULT 0 CHECK (quantity >= 0),
  unit           VARCHAR(20) NOT NULL DEFAULT 'un',
  min_quantity   NUMERIC(10, 3) NOT NULL DEFAULT 0 CHECK (min_quantity >= 0),
  is_perishable  BOOLEAN NOT NULL DEFAULT FALSE,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `name` | VARCHAR(100) | Nome do insumo (ex: "Pão de hambúrguer") |
| `quantity` | NUMERIC(10,3) | Estoque atual (somente itens fechados/não abertos) |
| `unit` | VARCHAR(20) | Unidade de medida (ex: `kg`, `un`, `L`, `cx`) |
| `min_quantity` | NUMERIC(10,3) | Quantidade mínima para alerta (ignorado se `is_perishable = true`) |
| `is_perishable` | BOOLEAN | Se `true`, item é perecível — alerta de mínimo desativado |
| `created_at` | TIMESTAMPTZ | Data de cadastro |
| `updated_at` | TIMESTAMPTZ | Última atualização (via trigger) |

---

### 5.6 Tabela `stock_adjustments`

```sql
CREATE TABLE stock_adjustments (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stock_item_id UUID NOT NULL REFERENCES stock_items(id) ON DELETE CASCADE,
  delta         NUMERIC(10, 3) NOT NULL,
  reason        VARCHAR(20) NOT NULL CHECK (reason IN ('consumo', 'deterioracao', 'reposicao')),
  adjusted_by   UUID NOT NULL REFERENCES profiles(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `stock_item_id` | UUID | Referência ao insumo |
| `delta` | NUMERIC(10,3) | Variação de quantidade (positivo = entrada, negativo = saída) |
| `reason` | VARCHAR(20) | Motivo: `consumo`, `deterioracao` ou `reposicao` |
| `adjusted_by` | UUID | Referência ao usuário que realizou o ajuste (`profiles.id`) |
| `created_at` | TIMESTAMPTZ | Data do ajuste |

---

### 5.7 Relacionamentos

```
profiles ──── auth.users (1:1, mesmo id)

products ───< order_items >─── orders
                  │
              (unit_price e observations capturados no momento do pedido)

stock_items ───< stock_adjustments >─── profiles
              (adjusted_by rastreia quem fez o ajuste)
              (independente — sem FK para orders nesta versão)
```

### 5.8 Índices Recomendados

```sql
-- Consultas de pedidos por data (painel do dia)
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
CREATE INDEX idx_orders_status       ON orders (status);

-- Itens por pedido (montagem do resumo)
CREATE INDEX idx_order_items_order_id ON order_items (order_id);

-- Produtos ativos no cardápio
CREATE INDEX idx_products_active ON products (active);

-- Ajustes por insumo
CREATE INDEX idx_stock_adjustments_item ON stock_adjustments (stock_item_id);
```

---

## 6. Estrutura de Pastas do Projeto

```
SistemasjB/
├── app/                          # Next.js App Router
│   ├── layout.tsx                # Layout raiz (providers, nav)
│   ├── page.tsx                  # Redireciona para /pedidos
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx          # Página de login
│   ├── (app)/                    # Grupo de rotas autenticadas
│   │   ├── layout.tsx            # Bottom nav + auth guard + role check
│   │   ├── pedidos/
│   │   │   ├── page.tsx          # Lista de pedidos do dia; botão cria pedido e redireciona
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Editar / visualizar pedido
│   │   ├── painel/
│   │   │   └── page.tsx          # Painel do dia (apenas donos)
│   │   ├── estoque/
│   │   │   ├── page.tsx          # Lista de insumos
│   │   │   ├── novo/
│   │   │   │   └── page.tsx      # Cadastrar insumo
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Editar insumo / registrar ajuste
│   │   ├── cardapio/
│   │   │   ├── page.tsx          # Lista de produtos
│   │   │   ├── novo/
│   │   │   │   └── page.tsx      # Cadastrar produto (apenas donos)
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Editar produto (apenas donos)
│   │   └── usuarios/
│   │       └── page.tsx          # Gestão de usuários (apenas donos; modal de criação inline)
│   └── api/
│       ├── orders/
│       │   ├── route.ts          # GET (listar) / POST (criar — redireciona para /pedidos/[id])
│       │   └── [id]/
│       │       ├── route.ts      # GET / PATCH / DELETE
│       │       ├── close/
│       │       │   └── route.ts  # POST — fechar pedido
│       │       └── items/
│       │           ├── route.ts  # POST — adicionar item ao pedido
│       │           └── [itemId]/
│       │               └── route.ts  # PATCH / DELETE — alterar/remover item
│       ├── products/
│       │   ├── route.ts          # GET / POST
│       │   └── [id]/
│       │       └── route.ts      # GET / PATCH / DELETE
│       ├── stock/
│       │   ├── route.ts          # GET / POST
│       │   └── [id]/
│       │       ├── route.ts      # GET / PATCH / DELETE
│       │       └── adjust/
│       │           └── route.ts  # POST — ajuste com motivo e adjusted_by
│       ├── users/
│       │   ├── route.ts          # GET (listar) / POST (criar via service role)
│       │   └── [id]/
│       │       └── route.ts      # PATCH (role / desativar)
│       └── dashboard/
│           └── route.ts          # GET — dados do painel do dia
├── components/
│   ├── ui/                       # Componentes shadcn/ui (gerados)
│   ├── orders/
│   │   ├── OrderCard.tsx
│   │   ├── OrderForm.tsx
│   │   ├── ProductGrid.tsx
│   │   ├── OrderItemRow.tsx      # Item com campo de observações
│   │   └── PaymentSelector.tsx
│   ├── dashboard/
│   │   ├── SummaryCard.tsx
│   │   ├── PaymentBreakdown.tsx
│   │   └── TopProductsList.tsx
│   ├── stock/
│   │   ├── StockItemCard.tsx
│   │   └── AdjustStockModal.tsx
│   ├── users/
│   │   ├── UserCard.tsx
│   │   └── CreateUserModal.tsx
│   └── layout/
│       └── BottomNav.tsx         # Menu de navegação inferior fixo
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # Cliente Supabase para o browser
│   │   ├── server.ts             # Cliente Supabase para Server Components
│   │   └── admin.ts              # Cliente privilegiado com service role (server-side only)
│   ├── utils.ts                  # Funções utilitárias (formatação, cálculo)
│   └── validations.ts            # Schemas Zod para validação de entrada
├── middleware.ts                  # Auth guard + verificação de role por rota
├── types/
│   └── index.ts                  # Tipos TypeScript (Order, Product, Profile, etc.)
├── hooks/                        # Custom React hooks
│   ├── useOrders.ts
│   ├── useProducts.ts
│   └── useProfile.ts
├── public/                       # Assets estáticos
├── .env.local                    # Variáveis de ambiente (não commitado)
├── .env.example                  # Template de variáveis de ambiente
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── components.json               # Config shadcn/ui
└── PROJECT.md                    # Este arquivo
```

---

## 7. Rotas de Página e de API

### 7.1 Rotas de Página (`/app`)

| Rota | Componente | Acesso |
|---|---|---|
| `/` | `app/page.tsx` | Redirect para `/pedidos` |
| `/login` | `app/(auth)/login/page.tsx` | Público |
| `/pedidos` | `app/(app)/pedidos/page.tsx` | todos |
| `/pedidos/[id]` | `app/(app)/pedidos/[id]/page.tsx` | todos |
| `/painel` | `app/(app)/painel/page.tsx` | somente donos |
| `/estoque` | `app/(app)/estoque/page.tsx` | todos |
| `/estoque/novo` | `app/(app)/estoque/novo/page.tsx` | todos |
| `/estoque/[id]` | `app/(app)/estoque/[id]/page.tsx` | todos |
| `/cardapio` | `app/(app)/cardapio/page.tsx` | todos (leitura); donos (edição) |
| `/cardapio/novo` | `app/(app)/cardapio/novo/page.tsx` | somente donos |
| `/cardapio/[id]` | `app/(app)/cardapio/[id]/page.tsx` | somente donos |
| `/usuarios` | `app/(app)/usuarios/page.tsx` | somente donos |

> **Nota:** `/pedidos/novo` não é uma rota de página. Tocar em "+ Novo Pedido" chama `POST /api/orders` e redireciona diretamente para `/pedidos/[id]`.

### 7.2 Rotas de API (`/api`)

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/api/orders` | Lista pedidos; aceita query `?date=YYYY-MM-DD&status=aberto` |
| `POST` | `/api/orders` | Cria novo pedido (`type` obrigatório; `customer_name` opcional) |
| `GET` | `/api/orders/[id]` | Retorna pedido com seus itens |
| `PATCH` | `/api/orders/[id]` | Atualiza tipo, nome do cliente ou status (aberto→em_preparo) |
| `DELETE` | `/api/orders/[id]` | Cancela pedido (status → `cancelado`) |
| `POST` | `/api/orders/[id]/close` | Fecha pedido: valida atendimento completo + pagamento, calcula total, grava `closed_at` |
| `POST` | `/api/orders/[id]/items` | Adiciona item ao pedido (`product_id`, `quantity`, `observations?`; captura `unit_price` do produto) |
| `PATCH` | `/api/orders/[id]/items/[itemId]` | Atualiza `quantity` ou `observations` de um item do pedido |
| `DELETE` | `/api/orders/[id]/items/[itemId]` | Remove item do pedido |
| `GET` | `/api/products` | Lista produtos; aceita query `?active=true` |
| `POST` | `/api/products` | Cria produto (somente donos) |
| `GET` | `/api/products/[id]` | Retorna produto |
| `PATCH` | `/api/products/[id]` | Atualiza nome, preço, categoria ou `active` (somente donos) |
| `DELETE` | `/api/products/[id]` | Desativa produto (`active = false`, soft delete; somente donos) |
| `GET` | `/api/stock` | Lista insumos; aceita query `?low=true` para só os abaixo do mínimo |
| `POST` | `/api/stock` | Cria insumo |
| `GET` | `/api/stock/[id]` | Retorna insumo |
| `PATCH` | `/api/stock/[id]` | Atualiza nome, unidade, quantidade mínima ou `is_perishable` |
| `DELETE` | `/api/stock/[id]` | Remove insumo |
| `POST` | `/api/stock/[id]/adjust` | Ajuste de estoque: `{ delta: number, reason: 'consumo' \| 'deterioracao' \| 'reposicao' }` |
| `GET` | `/api/dashboard` | Agrega dados do dia: total, contagem, breakdown pagamento, top produtos |
| `GET` | `/api/users` | Lista usuários (somente donos) |
| `POST` | `/api/users` | Cria usuário via Supabase Auth service role + insere em `profiles` (somente donos) |
| `PATCH` | `/api/users/[id]` | Atualiza `role` ou desativa acesso (somente donos) |

---

## 8. Fluxo de Uso Real — Do Pedido ao Fechamento

```
1. FUNCIONÁRIO — Tela /pedidos
   └─► Toca em "+ Novo Pedido"
       └─► Modal de seleção de tipo aparece na tela
           └─► Seleciona: [Delivery] [Retirada] [Local]
               └─► App chama POST /api/orders com type já definido
                   ├─► Servidor cria registro com status "aberto" no banco
                   └─► Redireciona para /pedidos/[id] do pedido recém-criado

2. FUNCIONÁRIO — Tela /pedidos/[id] (pedido recém-criado, type já definido)
   └─► Se Delivery: digita nome do cliente (opcional)
       └─► Navega pelo cardápio por categoria
           └─► Toca nos produtos para adicionar
               ├─► O resumo inferior atualiza em tempo real
               ├─► Ajusta quantidades com [+] e [−]
               └─► Adiciona observações por item (opcional)

3. FUNCIONÁRIO — Seleciona pagamento
   └─► Toca em [Dinheiro] [Pix] ou [Cartão]
       └─► Botão "Fechar Pedido" fica ativo

4. FUNCIONÁRIO — Fecha o pedido
   └─► Toca em [Fechar Pedido]
       └─► App chama POST /api/orders/[id]/close
           ├─► Servidor valida: itens > 0, payment_method presente, atendimento completo
           ├─► Calcula total (soma dos itens)
           ├─► Atualiza status → "fechado", grava closed_at
           └─► Retorna sucesso

5. SISTEMA — Feedback
   └─► Toast de confirmação "Pedido #X fechado com sucesso!"
       └─► Redireciona para /pedidos (lista atualizada)

6. DONO — A qualquer momento, acessa /painel
   └─► Vê totais do dia, breakdown de pagamentos e top produtos
       (dados calculados ao vivo por /api/dashboard)
```

---

## 9. Regras de Negócio

### 9.1 Pedidos

| # | Regra |
|---|---|
| RN-01 | Um pedido só pode ser fechado quando o cliente for completamente atendido **e** o pagamento for confirmado: ao menos 1 item adicionado e forma de pagamento selecionada. |
| RN-02 | O `unit_price` de cada item é capturado no momento da criação do item (não rastreia mudanças de preço futuras). |
| RN-03 | O `total` do pedido é calculado e validado no servidor; o valor calculado no cliente é apenas informativo. |
| RN-04 | Pedidos fechados (`fechado`) são imutáveis — nenhum campo pode ser alterado. |
| RN-05 | Pedidos cancelados (`cancelado`) são preservados no banco (soft delete) e excluídos do painel do dia. |
| RN-06 | O número sequencial do dia (#1, #2…) é exibido ao usuário mas não armazenado; é calculado dinamicamente como `ROW_NUMBER()` ordenado por `created_at` para pedidos não cancelados do dia. |
| RN-07 | O tipo do pedido pode ser alterado enquanto o status for aberto ou em_preparo. Ao fechar o pedido, o tipo fica imutável junto com os demais campos. |

### 9.2 Produtos

| # | Regra |
|---|---|
| RN-08 | Produtos não são excluídos fisicamente para preservar histórico de pedidos; `active = false` os remove do cardápio. |
| RN-09 | O preço de um produto pode ser alterado a qualquer momento sem afetar pedidos já registrados. |

### 9.3 Estoque

| # | Regra |
|---|---|
| RN-10 | A quantidade de um insumo não pode ficar abaixo de `0` (validação no servidor). |
| RN-11 | Um insumo não perecível é considerado em alerta quando `quantity <= min_quantity`. |
| RN-12 | Não há baixa automática de estoque ao fechar um pedido nesta versão. |
| RN-13 | Ajustes de estoque são registrados com `delta`, `reason` e `adjusted_by` obrigatórios. |
| RN-14 | Itens perecíveis (`is_perishable = true`) não exibem alerta de estoque mínimo; `min_quantity` é ignorado para eles. |
| RN-15 | Itens perecíveis possuem a ação "Baixa por Deterioração" (`reason = 'deterioracao'`) além da baixa normal de consumo. |
| RN-16 | Todo ajuste de estoque registra o motivo: `consumo` (uso normal), `deterioracao` (perda por perecimento) ou `reposicao` (entrada de mercadoria). |

### 9.4 Autenticação e Autorização

| # | Regra |
|---|---|
| RN-17 | Todas as rotas do grupo `(app)` requerem sessão autenticada. |
| RN-18 | Existem dois perfis de usuário: `funcionario` e `dono`. O perfil é armazenado na tabela `profiles` e verificado pelo middleware Next.js antes de renderizar rotas protegidas. |
| RN-19 | A sessão é gerenciada pelo Supabase Auth com cookies seguros (middleware Next.js). |
| RN-20 | Perfil `funcionario`: acesso a `/pedidos`, `/cardapio` (somente leitura) e `/estoque`. Acesso negado a `/painel`, `/usuarios` e edição do cardápio. |
| RN-21 | Perfil `dono`: acesso completo a todas as rotas, incluindo `/painel`, `/cardapio` com edição e `/usuarios`. Tentativa de acesso não autorizado por funcionário redireciona para `/pedidos` com toast de erro. |

---

## 10. Estrutura de Pastas do Projeto

> Já documentado na seção 6.

---

## 11. Variáveis de Ambiente

```bash
# .env.example

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGci...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGci...  # Apenas para operações server-side privilegiadas

# App
NEXT_PUBLIC_APP_NAME="Sistema São João Batista"
```

> ⚠️ **Nunca commitar `.env.local`**. Adicione ao `.gitignore`.

---

## 12. Queries SQL Úteis

### Total e contagem do dia

```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'fechado')         AS total_pedidos,
  COALESCE(SUM(total) FILTER (WHERE status = 'fechado'), 0) AS total_dia,
  COALESCE(SUM(total) FILTER (WHERE status = 'fechado' AND payment_method = 'dinheiro'), 0) AS total_dinheiro,
  COALESCE(SUM(total) FILTER (WHERE status = 'fechado' AND payment_method = 'pix'), 0)     AS total_pix,
  COALESCE(SUM(total) FILTER (WHERE status = 'fechado' AND payment_method = 'cartao'), 0)  AS total_cartao
FROM orders
WHERE created_at::date = CURRENT_DATE;
```

### Top produtos do dia

```sql
SELECT
  p.name,
  SUM(oi.quantity)               AS total_quantidade,
  SUM(oi.quantity * oi.unit_price) AS total_receita
FROM order_items oi
JOIN products p ON p.id = oi.product_id
JOIN orders o   ON o.id = oi.order_id
WHERE o.status = 'fechado'
  AND o.created_at::date = CURRENT_DATE
GROUP BY p.id, p.name
ORDER BY total_quantidade DESC
LIMIT 5;
```

### Insumos não perecíveis abaixo do mínimo

```sql
SELECT * FROM stock_items
WHERE is_perishable = FALSE
  AND quantity <= min_quantity
ORDER BY name;
```

### Histórico de ajustes de um insumo

```sql
SELECT sa.created_at, sa.delta, sa.reason, p.name AS adjusted_by
FROM stock_adjustments sa
JOIN profiles p ON p.id = sa.adjusted_by
WHERE sa.stock_item_id = '<uuid>'
ORDER BY sa.created_at DESC;
```

---

## 13. Ordem de Implementação

A seguinte sequência respeita as dependências entre as partes e garante que o sistema seja utilizável o mais cedo possível.

```
Fase 0 — Infraestrutura (pré-requisito de tudo)
  1. [ ] Criar projeto Next.js + TypeScript
  2. [ ] Configurar Tailwind CSS + shadcn/ui
  3. [ ] Criar projeto no Supabase + obter credenciais
  4. [ ] Configurar variáveis de ambiente (.env.local)
  5. [ ] Instalar e configurar cliente Supabase (client.ts + server.ts)
  6. [ ] Criar tabela profiles no Supabase
  7. [ ] Configurar middleware de autenticação com verificação de role por rota
  8. [ ] Implementar tela de login (/login)
  9. [ ] Implementar BottomNav com visibilidade condicional por role

Fase 1 — Dados base (pré-requisito do Módulo 0 e 1)
  10. [ ] Criar tabelas: products, orders, order_items (com campo observations), stock_items (com is_perishable), stock_adjustments (com reason e adjusted_by)
  11. [ ] Definir tipos TypeScript (types/index.ts) incluindo Profile, Order com novos tipos, OrderItem com observations
  12. [ ] Implementar API de produtos (GET /api/products)

Fase 2 — Módulo 0: Gestão de Cardápio
  13. [ ] Implementar tela /cardapio (lista com toggle ativo/inativo)
  14. [ ] Implementar tela /cardapio/novo e /cardapio/[id] (restrito a donos)
  15. [ ] Popular cardápio com produtos reais

Fase 3 — Módulo 1: Registro de Pedidos ⭐
  16. [ ] Implementar POST /api/orders (criar pedido com type: delivery | retirada | local; retorna id para redirect)
  17. [ ] Implementar GET /api/orders (listar pedidos do dia)
  18. [ ] Implementar tela /pedidos (lista; botão "+ Novo Pedido" cria registro e redireciona para /pedidos/[id])
  19. [ ] Implementar componente ProductGrid
  20. [ ] Implementar OrderItemRow com campo de observações opcional
  21. [ ] Implementar tela /pedidos/[id] (edição completa: tipo, produtos, observações, pagamento)
  22. [ ] Implementar POST /api/orders/[id]/close (fechar pedido com validação completa)
  23. [ ] Implementar PATCH /api/orders/[id] e DELETE (cancelar)
  24. [ ] Testes manuais do fluxo completo de pedido

Fase 4 — Módulo 2: Painel do Dia
  25. [ ] Implementar GET /api/dashboard (aggregations)
  26. [ ] Implementar componentes do painel (SummaryCard, PaymentBreakdown, TopProductsList)
  27. [ ] Implementar tela /painel (com guard de role = 'dono')

Fase 5 — Módulo 3: Estoque
  28. [ ] Implementar API de estoque (CRUD + /api/stock/[id]/adjust com reason)
  29. [ ] Implementar tela /estoque (lista com alerta para não perecíveis, badge para perecíveis)
  30. [ ] Implementar tela /estoque/novo e /estoque/[id] (com toggle is_perishable)
  31. [ ] Implementar AdjustStockModal com seleção de motivo (consumo / deterioracao / reposicao)

Fase 6 — Módulo 4: Gestão de Usuários
  32. [ ] Implementar GET /api/users e POST /api/users (via service role key)
  33. [ ] Implementar PATCH /api/users/[id] (role e desativação)
  34. [ ] Implementar tela /usuarios (lista + CreateUserModal)
  35. [ ] Validar regras: não desativar a si mesmo, não remover último dono

Fase 7 — Polimento e Deploy
  36. [ ] Revisão de UX mobile: altura de botões (min 56px), cards empilhados, teclado numérico
  37. [ ] Verificar ausência de tabelas com scroll horizontal — substituir por cards onde necessário
  38. [ ] Configurar projeto na Vercel + conectar ao GitHub
  39. [ ] Configurar variáveis de ambiente na Vercel
  40. [ ] Deploy e testes em produção
  41. [ ] Onboarding dos usuários (funcionário + donos)
```

---

## 14. Decisões Técnicas Fora do Escopo (V1)

As seguintes funcionalidades foram **conscientemente excluídas** desta versão para manter o escopo focado e o prazo viável.

| Funcionalidade | Motivo da Exclusão |
|---|---|
| **Integração fiscal (NF-e / NFC-e)** | Complexidade regulatória alta. Requer certificado digital, integração com SEFAZ e homologação estadual. Escopo futuro separado. |
| **Modo offline / PWA** | Service Workers e sincronização de dados offline aumentam muito a complexidade. Internet estável está disponível no local de uso. |
| **Integração com WhatsApp** | APIs do WhatsApp Business têm custo e limitações. O atendimento via WhatsApp continuará manual; o sistema registra o pedido após o recebimento. |
| **Histórico por cliente / CRM** | A lanchonete não mantém cadastro de clientes. Seria necessário esforço de coleta de dados que não se encaixa no fluxo atual. |
| **Baixa automática de estoque** | Requer mapeamento de receitas (quais insumos cada produto consome e em que quantidade), o que é trabalhoso de manter. Mantido para versão futura. |
| **Impressão de comanda / cupom** | Requer integração com impressora térmica (driver, SDK) — complexidade de hardware fora do escopo atual. |
| **Aplicativo mobile nativo** | O acesso via navegador no celular é suficiente. App nativo (iOS/Android) exigiria publicação em lojas e manutenção adicional. |
| **Relatórios históricos avançados** | O painel do dia cobre a necessidade imediata. Relatórios de semana/mês/período são escopo de V2. |
| **Multi-tenant / multi-loja** | Sistema é projetado para uma única lanchonete. Isolamento por tenant exigiria redesenho do modelo de dados. |

---

*Documento gerado em 2026-03-14. Última atualização: 2026-03-14.*

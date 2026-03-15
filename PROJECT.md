# PROJECT.md — LanchoneteSaaS
> Fonte única de verdade para desenvolvimento do sistema de gestão de lanchonete familiar.
> Versão: 1.0 · Data: 2026-03-14

---

## 1. Visão Geral

**LanchoneteSaaS** é um sistema web de gestão operacional desenvolvido para uso interno de uma lanchonete familiar de pequeno porte. O objetivo é substituir o controle manual (cadernos, calculadoras, planilhas) por uma interface digital simples, rápida e confiável.

### Contexto de Uso

| Aspecto | Detalhe |
|---|---|
| Porte | Pequena lanchonete familiar |
| Atendimento | Balcão presencial + Delivery via WhatsApp/telefone |
| Usuários | 1 funcionário + 2 donos |
| Dispositivos | Desktop ou tablet no balcão; celular para gestão |
| Perfil técnico | Baixo — interface deve ser extremamente simples |
| Acesso | Via navegador (sem instalação de app) |

### Princípios de UX

- **Poucos cliques**: operações críticas em no máximo 3 toques.
- **Botões grandes**: área de toque mínima de 48×48 px.
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

**Motivo:** shadcn/ui fornece componentes acessíveis e prontos (botões, modais, tabelas) que podem ser copiados para o projeto e customizados. Tailwind elimina a necessidade de arquivos CSS separados e acelera a prototipação. A combinação é o padrão de facto para projetos Next.js modernos.

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

## 3. Módulos do Sistema

### Módulo 1 — Registro de Pedidos ⭐ (Prioridade Máxima)

**Objetivo:** Tela operacional usada pelo funcionário dezenas de vezes por dia para registrar todas as vendas.

#### Telas

##### 3.1.1 Tela Principal de Pedidos (`/pedidos`)

- Lista dos pedidos ativos do dia (status: `aberto` ou `em_preparo`).
- Botão grande **"+ Novo Pedido"** no topo.
- Cada pedido exibe: número sequencial do dia, tipo (Balcão / Delivery), total parcial, status.
- Toque no pedido abre o detalhe/edição.

##### 3.1.2 Criar/Editar Pedido (`/pedidos/[id]`)

- **Tipo do pedido:** seleção visual entre "Balcão" e "Delivery" (botões grandes com ícone).
- **Campo nome** (opcional, mais relevante para delivery): nome do cliente.
- **Cardápio:** grade de produtos agrupados por categoria. Cada produto exibe nome e preço. Toque adiciona 1 unidade; há botões `+` e `−` para ajustar quantidade.
- **Resumo do pedido:** lista lateral (ou inferior em mobile) com itens selecionados, quantidades e subtotais.
- **Forma de pagamento:** seleção obrigatória antes de fechar — "Dinheiro", "Pix", "Cartão".
- **Botão "Fechar Pedido":** ativo somente quando ao menos 1 item foi adicionado e a forma de pagamento foi selecionada.
- **Botão "Cancelar Pedido":** disponível enquanto o pedido está aberto.

#### Regras de Negócio — Módulo 1

- Um pedido só pode ser fechado se tiver pelo menos 1 item.
- Um pedido só pode ser fechado se a forma de pagamento estiver selecionada.
- O total do pedido é calculado no cliente (soma de `quantity × unit_price`) e validado novamente no servidor antes de persistir.
- O `unit_price` dos itens é capturado no momento do pedido (não muda com alterações futuras no cardápio).
- Pedidos cancelados ficam com status `cancelado` (soft delete — não são excluídos).
- Pedidos fechados não podem ser editados.
- O número sequencial do dia é calculado como: `COUNT(*) + 1` dos pedidos do dia (status ≠ `cancelado`).

---

### Módulo 2 — Painel do Dia

**Objetivo:** Tela gerencial para os donos acompanharem o desempenho em tempo real.

#### Telas

##### 3.2.1 Painel Gerencial (`/painel`)

- **Total do dia:** valor em destaque, grande.
- **Quantidade de pedidos** fechados no dia.
- **Ticket médio:** Total ÷ Quantidade de pedidos.
- **Breakdown por forma de pagamento:** Dinheiro / Pix / Cartão — valor e percentual de cada.
- **Ranking dos 5 produtos mais vendidos:** nome do produto, quantidade vendida, receita gerada.
- **Comparativo simples:** "vs. ontem" (opcional na V1 — porém a query suporta).
- Filtro de data (padrão: hoje). Seleção de outra data para ver dias anteriores.
- Layout legível em celular (coluna única, fontes grandes).

#### Regras de Negócio — Módulo 2

- Considera apenas pedidos com status `fechado`.
- O painel é somente leitura — nenhuma ação é realizada a partir dele.
- Os dados são carregados a cada acesso (sem polling automático na V1).

---

### Módulo 3 — Controle de Estoque

**Objetivo:** Registro e monitoramento manual dos insumos para evitar falta de produtos.

#### Telas

##### 3.3.1 Lista de Insumos (`/estoque`)

- Listagem de todos os insumos com: nome, quantidade atual, unidade (kg, un, L, etc.), quantidade mínima.
- Itens **abaixo do mínimo** são destacados visualmente (fundo vermelho claro / ícone de alerta).
- Botão **"+ Novo Insumo"**.
- Cada linha tem acesso rápido para **Baixa Rápida** e **Editar**.

##### 3.3.2 Cadastro/Edição de Insumo (`/estoque/[id]`)

- Campos: Nome, Quantidade Atual, Unidade, Quantidade Mínima.
- Salvar ou Excluir insumo.

##### 3.3.3 Registrar Baixa Manual

- Modal ou seção inline: campo numérico para inserir a quantidade a debitar.
- Confirmação com preview do novo saldo.
- Opção de adicionar (reposição de estoque) ou subtrair (baixa/consumo).

#### Regras de Negócio — Módulo 3

- Quantidade nunca pode ficar negativa (validação no servidor).
- O alerta de mínimo é exibido quando `quantity <= min_quantity`.
- Não há integração automática com pedidos nesta versão (baixa sempre manual).
- A unidade de medida é um campo de texto livre (sem conversão entre unidades).

---

## 4. Modelo de Dados

### 4.1 Tabela `products`

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
| `category` | VARCHAR(50) | Agrupamento no cardápio (ex: "Lanches", "Bebidas") |
| `active` | BOOLEAN | Se `false`, não aparece no cardápio mas histórico é preservado |
| `created_at` | TIMESTAMPTZ | Data de cadastro |

---

### 4.2 Tabela `orders`

```sql
CREATE TABLE orders (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type           VARCHAR(10) NOT NULL CHECK (type IN ('balcao', 'delivery')),
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
| `type` | VARCHAR(10) | `balcao` ou `delivery` |
| `status` | VARCHAR(15) | `aberto`, `em_preparo`, `fechado`, `cancelado` |
| `payment_method` | VARCHAR(10) | Obrigatório ao fechar; `dinheiro`, `pix` ou `cartao` |
| `customer_name` | VARCHAR(100) | Nome do cliente (opcional, mais usado no delivery) |
| `total` | NUMERIC(10,2) | Soma de `order_items.quantity × unit_price` |
| `created_at` | TIMESTAMPTZ | Abertura do pedido |
| `closed_at` | TIMESTAMPTZ | Momento do fechamento (NULL enquanto aberto) |

---

### 4.3 Tabela `order_items`

```sql
CREATE TABLE order_items (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id    UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id  UUID NOT NULL REFERENCES products(id),
  quantity    INTEGER NOT NULL CHECK (quantity > 0),
  unit_price  NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0)
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `order_id` | UUID | Referência ao pedido pai |
| `product_id` | UUID | Referência ao produto (preserva histórico) |
| `quantity` | INTEGER | Quantidade pedida |
| `unit_price` | NUMERIC(10,2) | Preço no momento do pedido (snapshot) |

---

### 4.4 Tabela `stock_items`

```sql
CREATE TABLE stock_items (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name         VARCHAR(100) NOT NULL UNIQUE,
  quantity     NUMERIC(10, 3) NOT NULL DEFAULT 0 CHECK (quantity >= 0),
  unit         VARCHAR(20) NOT NULL DEFAULT 'un',
  min_quantity NUMERIC(10, 3) NOT NULL DEFAULT 0 CHECK (min_quantity >= 0),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | UUID | Identificador único |
| `name` | VARCHAR(100) | Nome do insumo (ex: "Pão de hambúrguer") |
| `quantity` | NUMERIC(10,3) | Estoque atual |
| `unit` | VARCHAR(20) | Unidade de medida (ex: `kg`, `un`, `L`, `cx`) |
| `min_quantity` | NUMERIC(10,3) | Quantidade mínima para alerta |
| `created_at` | TIMESTAMPTZ | Data de cadastro |
| `updated_at` | TIMESTAMPTZ | Última atualização (via trigger ou app) |

---

### 4.5 Relacionamentos

```
products ───< order_items >─── orders
                  │
              (unit_price capturado no momento do pedido)

stock_items  (independente — sem FK para orders nesta versão)
```

### 4.6 Índices Recomendados

```sql
-- Consultas de pedidos por data (painel do dia)
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
CREATE INDEX idx_orders_status       ON orders (status);

-- Itens por pedido (montagem do resumo)
CREATE INDEX idx_order_items_order_id ON order_items (order_id);

-- Produtos ativos no cardápio
CREATE INDEX idx_products_active ON products (active);
```

---

## 5. Estrutura de Pastas do Projeto

```
LanchoneteSaaS/
├── app/                          # Next.js App Router
│   ├── layout.tsx                # Layout raiz (providers, nav)
│   ├── page.tsx                  # Redireciona para /pedidos
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx          # Página de login
│   ├── (app)/                    # Grupo de rotas autenticadas
│   │   ├── layout.tsx            # Sidebar/navbar + auth guard
│   │   ├── pedidos/
│   │   │   ├── page.tsx          # Lista de pedidos do dia
│   │   │   ├── novo/
│   │   │   │   └── page.tsx      # Criar novo pedido
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Editar / visualizar pedido
│   │   ├── painel/
│   │   │   └── page.tsx          # Painel do dia (gerencial)
│   │   ├── estoque/
│   │   │   ├── page.tsx          # Lista de insumos
│   │   │   ├── novo/
│   │   │   │   └── page.tsx      # Cadastrar insumo
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Editar insumo / registrar baixa
│   │   └── cardapio/
│   │       ├── page.tsx          # Lista de produtos (admin)
│   │       ├── novo/
│   │       │   └── page.tsx      # Cadastrar produto
│   │       └── [id]/
│   │           └── page.tsx      # Editar produto
│   └── api/
│       ├── orders/
│       │   ├── route.ts          # GET (listar) / POST (criar)
│       │   └── [id]/
│       │       ├── route.ts      # GET / PATCH / DELETE
│       │       └── close/
│       │           └── route.ts  # POST — fechar pedido
│       ├── products/
│       │   ├── route.ts          # GET / POST
│       │   └── [id]/
│       │       └── route.ts      # GET / PATCH / DELETE
│       ├── stock/
│       │   ├── route.ts          # GET / POST
│       │   └── [id]/
│       │       ├── route.ts      # GET / PATCH / DELETE
│       │       └── adjust/
│       │           └── route.ts  # POST — baixa ou reposição
│       └── dashboard/
│           └── route.ts          # GET — dados do painel do dia
├── components/
│   ├── ui/                       # Componentes shadcn/ui (gerados)
│   ├── orders/
│   │   ├── OrderCard.tsx
│   │   ├── OrderForm.tsx
│   │   ├── ProductGrid.tsx
│   │   └── PaymentSelector.tsx
│   ├── dashboard/
│   │   ├── SummaryCard.tsx
│   │   ├── PaymentBreakdown.tsx
│   │   └── TopProductsList.tsx
│   ├── stock/
│   │   ├── StockItemRow.tsx
│   │   └── AdjustStockModal.tsx
│   └── layout/
│       ├── Navbar.tsx
│       └── Sidebar.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # Cliente Supabase para o browser
│   │   └── server.ts             # Cliente Supabase para Server Components
│   ├── utils.ts                  # Funções utilitárias (formatação, cálculo)
│   └── validations.ts            # Schemas Zod para validação de entrada
├── types/
│   └── index.ts                  # Tipos TypeScript (Order, Product, etc.)
├── hooks/                        # Custom React hooks
│   ├── useOrders.ts
│   └── useProducts.ts
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

## 6. Rotas de Página e de API

### 6.1 Rotas de Página (`/app`)

| Rota | Componente | Descrição |
|---|---|---|
| `/` | `app/page.tsx` | Redirect para `/pedidos` |
| `/login` | `app/(auth)/login/page.tsx` | Formulário de login (email + senha) |
| `/pedidos` | `app/(app)/pedidos/page.tsx` | Lista de pedidos abertos e fechados do dia |
| `/pedidos/novo` | `app/(app)/pedidos/novo/page.tsx` | Formulário de criação de pedido |
| `/pedidos/[id]` | `app/(app)/pedidos/[id]/page.tsx` | Edição/visualização de pedido |
| `/painel` | `app/(app)/painel/page.tsx` | Painel gerencial do dia |
| `/estoque` | `app/(app)/estoque/page.tsx` | Lista de insumos com alertas |
| `/estoque/novo` | `app/(app)/estoque/novo/page.tsx` | Cadastro de novo insumo |
| `/estoque/[id]` | `app/(app)/estoque/[id]/page.tsx` | Edição/baixa de insumo |
| `/cardapio` | `app/(app)/cardapio/page.tsx` | Gestão do cardápio (produtos) |
| `/cardapio/novo` | `app/(app)/cardapio/novo/page.tsx` | Cadastro de novo produto |
| `/cardapio/[id]` | `app/(app)/cardapio/[id]/page.tsx` | Edição de produto |

### 6.2 Rotas de API (`/api`)

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/api/orders` | Lista pedidos; aceita query `?date=YYYY-MM-DD&status=aberto` |
| `POST` | `/api/orders` | Cria novo pedido (`type`, `customer_name` opcionais) |
| `GET` | `/api/orders/[id]` | Retorna pedido com seus itens |
| `PATCH` | `/api/orders/[id]` | Atualiza tipo, nome do cliente ou status (aberto→em_preparo) |
| `DELETE` | `/api/orders/[id]` | Cancela pedido (status → `cancelado`) |
| `POST` | `/api/orders/[id]/close` | Fecha pedido: valida itens + pagamento, calcula total, grava `closed_at` |
| `GET` | `/api/products` | Lista produtos; aceita query `?active=true` |
| `POST` | `/api/products` | Cria produto |
| `GET` | `/api/products/[id]` | Retorna produto |
| `PATCH` | `/api/products/[id]` | Atualiza nome, preço, categoria ou `active` |
| `DELETE` | `/api/products/[id]` | Desativa produto (`active = false`, soft delete) |
| `GET` | `/api/stock` | Lista insumos; aceita query `?low=true` para só os abaixo do mínimo |
| `POST` | `/api/stock` | Cria insumo |
| `GET` | `/api/stock/[id]` | Retorna insumo |
| `PATCH` | `/api/stock/[id]` | Atualiza nome, unidade ou quantidade mínima |
| `DELETE` | `/api/stock/[id]` | Remove insumo |
| `POST` | `/api/stock/[id]/adjust` | Ajuste de estoque: `{ delta: number }` (positivo=entrada, negativo=baixa) |
| `GET` | `/api/dashboard` | Agrega dados do dia: total, contagem, breakdown pagamento, top produtos |

---

## 7. Fluxo de Uso Real — Do Pedido ao Fechamento

```
1. FUNCIONÁRIO — Tela /pedidos
   └─► Clica em "+ Novo Pedido"
       └─► Vai para /pedidos/novo

2. FUNCIONÁRIO — Tela /pedidos/novo
   └─► Seleciona tipo: [Balcão] ou [Delivery]
       ├─► Se Delivery: digita nome do cliente (opcional)
       └─► Navega pelo cardápio por categoria
           └─► Toca nos produtos para adicionar
               ├─► O resumo lateral atualiza em tempo real
               └─► Ajusta quantidades com [+] e [−]

3. FUNCIONÁRIO — Seleciona pagamento
   └─► Toca em [Dinheiro] [Pix] ou [Cartão]
       └─► Botão "Fechar Pedido" fica ativo

4. FUNCIONÁRIO — Fecha o pedido
   └─► Toca em [Fechar Pedido]
       └─► App chama POST /api/orders/[id]/close
           ├─► Servidor valida: itens > 0, payment_method presente
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

## 8. Regras de Negócio

### 8.1 Pedidos

| # | Regra |
|---|---|
| RN-01 | Um pedido só pode ser fechado com pelo menos 1 item adicionado. |
| RN-02 | Um pedido só pode ser fechado com forma de pagamento selecionada. |
| RN-03 | O `unit_price` de cada item é capturado no momento da criação do item (não rastreia mudanças de preço futuras). |
| RN-04 | O `total` do pedido é calculado e validado no servidor; o valor calculado no cliente é apenas informativo. |
| RN-05 | Pedidos fechados (`fechado`) são imutáveis — nenhum campo pode ser alterado. |
| RN-06 | Pedidos cancelados (`cancelado`) são preservados no banco (soft delete) e excluídos do painel do dia. |
| RN-07 | O número sequencial do dia (#1, #2…) é exibido ao usuário mas não armazenado; é calculado dinamicamente como `ROW_NUMBER()` ordenado por `created_at` para pedidos não cancelados do dia. |

### 8.2 Produtos

| # | Regra |
|---|---|
| RN-08 | Produtos não são excluídos fisicamente para preservar histórico de pedidos; `active = false` os remove do cardápio. |
| RN-09 | O preço de um produto pode ser alterado a qualquer momento sem afetar pedidos já registrados. |

### 8.3 Estoque

| # | Regra |
|---|---|
| RN-10 | A quantidade de um insumo não pode ficar abaixo de `0` (validação no servidor). |
| RN-11 | Um insumo é considerado em alerta quando `quantity <= min_quantity`. |
| RN-12 | Não há baixa automática de estoque ao fechar um pedido nesta versão. |
| RN-13 | Ajustes de estoque são registrados com `delta` positivo (entrada) ou negativo (saída). |

### 8.4 Autenticação e Autorização

| # | Regra |
|---|---|
| RN-14 | Todas as rotas do grupo `(app)` requerem sessão autenticada. |
| RN-15 | Não há diferenciação de perfil entre "funcionário" e "dono" na V1 — todos os usuários autenticados têm acesso a todas as telas. |
| RN-16 | A sessão é gerenciada pelo Supabase Auth com cookies seguros (middleware Next.js). |

---

## 9. Ordem de Implementação

A seguinte sequência respeita as dependências entre as partes e garante que o sistema seja utilizável o mais cedo possível.

```
Fase 0 — Infraestrutura (pré-requisito de tudo)
  1. [ ] Criar projeto Next.js + TypeScript
  2. [ ] Configurar Tailwind CSS + shadcn/ui
  3. [ ] Criar projeto no Supabase + obter credenciais
  4. [ ] Configurar variáveis de ambiente (.env.local)
  5. [ ] Instalar e configurar cliente Supabase (client.ts + server.ts)
  6. [ ] Configurar middleware de autenticação (auth guard para rotas /app)
  7. [ ] Implementar tela de login (/login)

Fase 1 — Dados base (pré-requisito do Módulo 1)
  8. [ ] Criar tabelas no Supabase: products, orders, order_items
  9. [ ] Definir tipos TypeScript (types/index.ts)
  10. [ ] Implementar API de produtos (GET /api/products)
  11. [ ] Implementar tela de cardápio (/cardapio, /cardapio/novo, /cardapio/[id])
  12. [ ] Popular cardápio com produtos reais

Fase 2 — Módulo 1: Registro de Pedidos ⭐
  13. [ ] Implementar POST /api/orders (criar pedido)
  14. [ ] Implementar GET /api/orders (listar pedidos do dia)
  15. [ ] Implementar tela /pedidos (lista)
  16. [ ] Implementar componente ProductGrid
  17. [ ] Implementar tela /pedidos/novo (criação com seleção de produtos)
  18. [ ] Implementar POST /api/orders/[id]/close (fechar pedido)
  19. [ ] Implementar PATCH /api/orders/[id] e DELETE (cancelar)
  20. [ ] Implementar tela /pedidos/[id] (edição)
  21. [ ] Testes manuais do fluxo completo de pedido

Fase 3 — Módulo 2: Painel do Dia
  22. [ ] Implementar GET /api/dashboard (aggregations)
  23. [ ] Implementar componentes do painel (SummaryCard, PaymentBreakdown, TopProductsList)
  24. [ ] Implementar tela /painel

Fase 4 — Módulo 3: Estoque
  25. [ ] Criar tabela stock_items no Supabase
  26. [ ] Implementar API de estoque (CRUD + /api/stock/[id]/adjust)
  27. [ ] Implementar tela /estoque (lista com alertas)
  28. [ ] Implementar tela /estoque/novo e /estoque/[id]
  29. [ ] Implementar AdjustStockModal

Fase 5 — Polimento e Deploy
  30. [ ] Revisão de UX: tamanho de botões, feedback visual, toasts
  31. [ ] Responsividade mobile (painel do dia)
  32. [ ] Configurar projeto na Vercel + conectar ao GitHub
  33. [ ] Configurar variáveis de ambiente na Vercel
  34. [ ] Deploy e testes em produção
  35. [ ] Onboarding dos usuários (funcionário + donos)
```

---

## 10. Decisões Técnicas Fora do Escopo (V1)

As seguintes funcionalidades foram **conscientemente excluídas** desta versão para manter o escopo focado e o prazo viável.

| Funcionalidade | Motivo da Exclusão |
|---|---|
| **Integração fiscal (NF-e / NFC-e)** | Complexidade regulatória alta. Requer certificado digital, integração com SEFAZ e homologação estadual. Escopo futuro separado. |
| **Modo offline / PWA** | Service Workers e sincronização de dados offline aumentam muito a complexidade. Internet estável está disponível no local de uso. |
| **Integração com WhatsApp** | APIs do WhatsApp Business têm custo e limitações. O atendimento via WhatsApp continuará manual; o sistema registra o pedido após o recebimento. |
| **Histórico por cliente / CRM** | A lanchonete não mantém cadastro de clientes. Seria necessário esforço de coleta de dados que não se encaixa no fluxo atual. |
| **Baixa automática de estoque** | Requer mapeamento de receitas (quais insumos cada produto consome e em que quantidade), o que é trabalhoso de manter. Mantido para versão futura. |
| **Múltiplos perfis de acesso (RBAC)** | Com 3 usuários conhecidos e confiança mútua, controle granular de permissões é desnecessário na V1. |
| **Impressão de comanda / cupom** | Requer integração com impressora térmica (driver, SDK) — complexidade de hardware fora do escopo atual. |
| **Aplicativo mobile nativo** | O acesso via navegador no tablet/celular é suficiente. App nativo (iOS/Android) exigiria publicação em lojas e manutenção adicional. |
| **Relatórios históricos avançados** | O painel do dia cobre a necessidade imediata. Relatórios de semana/mês/período são escopo de V2. |
| **Multi-tenant / multi-loja** | Sistema é projetado para uma única lanchonete. Isolamento por tenant exigiria redesenho do modelo de dados. |

---

## 11. Variáveis de Ambiente

```bash
# .env.example

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGci...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGci...  # Apenas para operações server-side privilegiadas

# App
NEXT_PUBLIC_APP_NAME="LanchoneteSaaS"
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

### Insumos abaixo do mínimo

```sql
SELECT * FROM stock_items
WHERE quantity <= min_quantity
ORDER BY name;
```

---

*Documento gerado em 2026-03-14. Última atualização: 2026-03-14.*

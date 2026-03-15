# TASKS.md — Sistema São João Batista
> Tarefas executáveis derivadas do `PROJECT.md`.
> Cada tarefa é autossuficiente: um agente pode executá-la sem depender de outra rodando em paralelo.
> Versão: 1.0 · Data: 2026-03-14

---

## Como usar este arquivo

- `[ ]` — tarefa pendente
- `[/]` — tarefa em andamento
- `[x]` — tarefa concluída
- **Dependências** listadas em cada tarefa devem estar `[x]` antes de iniciar a tarefa atual.
- Tarefas dentro de uma mesma fase podem ser executadas em paralelo quando não há dependência entre elas.

---

## Fase 0 — Infraestrutura

> Pré-requisito de tudo. Nenhuma outra fase pode começar antes de F0-01 e F0-02 estarem concluídas.

---

### F0-01 — Inicializar o projeto Next.js com toda a configuração base

**Dependências:** nenhuma

**O que fazer:**

- Criar o projeto com `npx create-next-app@latest ./ --typescript --tailwind --eslint --app --src-dir=false --import-alias="@/*"` no diretório `LanchoneteSaaS/`
- Instalar e configurar shadcn/ui: `npx shadcn@latest init` (escolher tema Neutral, CSS variables: yes)
- Adicionar os componentes shadcn/ui usados no projeto:
  - `npx shadcn@latest add button card badge input label select sheet toast dialog`
- Instalar dependências adicionais:
  - `npm install @supabase/supabase-js @supabase/ssr zod`
- Criar arquivo `.env.example` na raiz com as variáveis:
  ```
  NEXT_PUBLIC_SUPABASE_URL=
  NEXT_PUBLIC_SUPABASE_ANON_KEY=
  SUPABASE_SERVICE_ROLE_KEY=
  NEXT_PUBLIC_APP_NAME="Sistema São João Batista"
  ```
- Criar `.env.local` a partir do `.env.example` (não commitar)
- Confirmar que `.env.local` está no `.gitignore`
- Criar estrutura de pastas conforme `PROJECT.md` seção 6:
  - `components/ui/` (gerado pelo shadcn)
  - `components/orders/`
  - `components/dashboard/`
  - `components/stock/`
  - `components/users/`
  - `components/layout/`
  - `lib/supabase/`
  - `types/`
  - `hooks/`
- Criar `app/page.tsx` com redirect para `/pedidos`

**Critério de conclusão:**
- `npm run dev` sobe sem erros
- `npm run build` compila sem erros
- A rota `/` redireciona para `/pedidos` (ainda que a página não exista, o redirect está correto)
- Estrutura de pastas existe conforme especificado

---

### F0-02 — Criar e configurar o projeto no Supabase

**Dependências:** F0-01

**O que fazer:**

- Criar novo projeto no [Supabase Dashboard](https://supabase.com/dashboard) com nome "sistema-sao-joao-batista"
- Copiar as credenciais para `.env.local`:
  - `NEXT_PUBLIC_SUPABASE_URL`
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
  - `SUPABASE_SERVICE_ROLE_KEY`
- Criar `lib/supabase/client.ts` — cliente para uso no browser (componentes Client):
  ```ts
  import { createBrowserClient } from '@supabase/ssr'
  export function createClient() {
    return createBrowserClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
    )
  }
  ```
- Criar `lib/supabase/server.ts` — cliente para uso em Server Components e Route Handlers:
  ```ts
  import { createServerClient } from '@supabase/ssr'
  import { cookies } from 'next/headers'
  export async function createClient() {
    const cookieStore = await cookies()
    return createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      { cookies: { getAll() { return cookieStore.getAll() }, setAll(cs) { cs.forEach(({ name, value, options }) => cookieStore.set(name, value, options)) } } }
    )
  }
  ```
- Criar `lib/supabase/admin.ts` — cliente privilegiado com service role (apenas server-side):
  ```ts
  import { createClient } from '@supabase/supabase-js'
  export const supabaseAdmin = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )
  ```

**Critério de conclusão:**
- Os três arquivos de cliente Supabase existem e compilam sem erro
- As variáveis de ambiente estão preenchidas no `.env.local`
- `npm run build` continua passando

---

### F0-03 — Criar todas as tabelas no banco de dados

**Dependências:** F0-02

**O que fazer:**

Executar os seguintes SQLs no **SQL Editor do Supabase Dashboard**, nesta ordem:

1. **Tabela `profiles`**
```sql
CREATE TABLE profiles (
  id         UUID PRIMARY KEY REFERENCES auth.users(id),
  name       VARCHAR(100) NOT NULL,
  role       VARCHAR(20) NOT NULL DEFAULT 'funcionario'
             CHECK (role IN ('funcionario', 'dono')),
  active     BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

2. **Tabela `products`**
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

3. **Tabela `orders`**
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

4. **Tabela `order_items`**
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

5. **Tabela `stock_items`**
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

6. **Tabela `stock_adjustments`**
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

7. **Índices**
```sql
CREATE INDEX idx_orders_created_at      ON orders (created_at DESC);
CREATE INDEX idx_orders_status          ON orders (status);
CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_products_active        ON products (active);
CREATE INDEX idx_stock_adjustments_item ON stock_adjustments (stock_item_id);
```

8. **Trigger `updated_at` para `stock_items`**
```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_stock_items_updated_at
BEFORE UPDATE ON stock_items
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

9. **Trigger para criar `profiles` automaticamente ao criar usuário no Supabase Auth**
```sql
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, name, role)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data->>'name', NEW.email),
    COALESCE(NEW.raw_user_meta_data->>'role', 'funcionario')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users
FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

10. **Row Level Security (RLS)** — habilitar RLS em todas as tabelas públicas e criar policies permissivas para usuários autenticados (a autorização por `role` será feita na camada Next.js/middleware):
```sql
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE stock_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE stock_adjustments ENABLE ROW LEVEL SECURITY;

-- Policy: usuário autenticado lê e escreve tudo
CREATE POLICY "authenticated_full_access" ON profiles      FOR ALL TO authenticated USING (true) WITH CHECK (true);
CREATE POLICY "authenticated_full_access" ON products      FOR ALL TO authenticated USING (true) WITH CHECK (true);
CREATE POLICY "authenticated_full_access" ON orders        FOR ALL TO authenticated USING (true) WITH CHECK (true);
CREATE POLICY "authenticated_full_access" ON order_items   FOR ALL TO authenticated USING (true) WITH CHECK (true);
CREATE POLICY "authenticated_full_access" ON stock_items   FOR ALL TO authenticated USING (true) WITH CHECK (true);
CREATE POLICY "authenticated_full_access" ON stock_adjustments FOR ALL TO authenticated USING (true) WITH CHECK (true);
```

**Critério de conclusão:**
- Todas as 6 tabelas existem no Supabase com estrutura correta (verificar no Table Editor)
- Índices e triggers criados (verificar em Database → Triggers e Indexes)
- RLS habilitado em todas as tabelas
- Inserção de teste manual em `products` funciona via SQL Editor

---

### F0-04 — Configurar middleware de autenticação com verificação de role

**Dependências:** F0-01, F0-02, F0-03

**O que fazer:**

- Criar `middleware.ts` na raiz do projeto com a seguinte lógica:
  1. Para qualquer rota dentro de `/(app)`, verificar se há sessão Supabase Auth ativa; se não, redirecionar para `/login`
  2. Para rotas restritas a `dono` (`/painel`, `/usuarios`, `/cardapio/novo`, `/cardapio/[id]`), buscar `profiles.role` do usuário autenticado; se `role = 'funcionario'`, redirecionar para `/pedidos` (o toast de erro será disparado via query param `?error=acesso_negado`)
  3. Rotas públicas: `/login` — não verificar sessão
- Criar `lib/auth.ts` com helper `getSession()` e `getUserRole()` para uso nos Server Components e Route Handlers
- Implementar `app/(app)/layout.tsx` que:
  - Valida sessão (fallback do middleware)
  - Busca o `profile` do usuário e o disponibiliza via Context ou props

**Critério de conclusão:**
- Acessar `/pedidos` sem sessão redireciona para `/login`
- Acessar `/painel` como funcionário redireciona para `/pedidos`
- Acessar `/painel` como dono funciona normalmente
- `npm run build` passa sem erros de tipo

---

### F0-05 — Implementar tela de login

**Dependências:** F0-01, F0-02, F0-04

**O que fazer:**

- Criar `app/(auth)/login/page.tsx` com:
  - Formulário com campos: **E-mail** e **Senha**
  - Botão **"Entrar"** com altura mínima de 56px
  - Exibição de erro em caso de credenciais inválidas (mensagem clara, sem jargão técnico)
  - Submit chama `supabase.auth.signInWithPassword({ email, password })`
  - Em caso de sucesso, redireciona para `/pedidos`
  - Estado de loading durante o submit (botão desabilitado + spinner)
- Criar `app/(auth)/layout.tsx` com layout centralizado (tela de login ocupando toda a altura)
- Criar `app/api/auth/callback/route.ts` para lidar com o callback do Supabase Auth (necessário para o fluxo de session)
- Exibir o nome do sistema "Sistema São João Batista" na tela de login

**Critério de conclusão:**
- Login com credenciais válidas redireciona para `/pedidos`
- Login com credenciais inválidas exibe mensagem de erro legível
- Botão tem altura mínima de 56px
- Layout é responsivo em celular (coluna única, centralizado)

---

### F0-06 — Implementar componente BottomNav com visibilidade condicional por role

**Dependências:** F0-04, F0-05

**O que fazer:**

- Criar `components/layout/BottomNav.tsx`:
  - Menu fixo na base da tela (`fixed bottom-0 left-0 right-0`)
  - 4 itens (ou 3 para funcionário): ícone centralizado acima do label
  - Itens: **Pedidos** (`clipboard`), **Estoque** (`package`), **Cardápio** (`book`), **Painel** (`bar-chart`)
  - Item **Painel** renderizado apenas quando `role = 'dono'`
  - Item ativo destacado visualmente (cor diferente)
  - Toda a largura da tela dividida igualmente entre os itens
  - Altura suficiente para toque confortável (mínimo 64px de altura)
  - Usar ícones do pacote `lucide-react` (já incluído com shadcn/ui)
- Registrar o componente no `app/(app)/layout.tsx` passando o `role` do usuário
- Garantir que o conteúdo das páginas não fique escondido pelo menu (`pb-20` ou equivalente no wrapper principal)

**Critério de conclusão:**
- Menu aparece na base da tela em todas as páginas autenticadas
- Funcionário vê apenas 3 itens (sem Painel)
- Dono vê todos os 4 itens
- Item da rota atual aparece destacado
- Conteúdo das páginas não é coberto pelo menu

---

## Fase 1 — Dados Base

> Pré-requisito para os módulos 0 e 1. Pode ser iniciada assim que F0-01 a F0-03 estiverem concluídas.

---

### F1-01 — Definir os tipos TypeScript globais

**Dependências:** F0-03

**O que fazer:**

- Criar `types/index.ts` com os seguintes tipos, alinhados com o modelo de dados do `PROJECT.md`:

```ts
export type Role = 'funcionario' | 'dono'

export type Profile = {
  id: string
  name: string
  role: Role
  active: boolean
  created_at: string
}

export type Product = {
  id: string
  name: string
  price: number
  category: string
  active: boolean
  created_at: string
}

export type OrderType = 'delivery' | 'retirada' | 'local'
export type OrderStatus = 'aberto' | 'em_preparo' | 'fechado' | 'cancelado'
export type PaymentMethod = 'dinheiro' | 'pix' | 'cartao'

export type OrderItem = {
  id: string
  order_id: string
  product_id: string
  quantity: number
  unit_price: number
  observations: string | null
  product?: Product // join opcional
}

export type Order = {
  id: string
  type: OrderType
  status: OrderStatus
  payment_method: PaymentMethod | null
  customer_name: string | null
  total: number
  created_at: string
  closed_at: string | null
  order_items?: OrderItem[]
}

export type StockItem = {
  id: string
  name: string
  quantity: number
  unit: string
  min_quantity: number
  is_perishable: boolean
  created_at: string
  updated_at: string
}

export type AdjustReason = 'consumo' | 'deterioracao' | 'reposicao'

export type StockAdjustment = {
  id: string
  stock_item_id: string
  delta: number
  reason: AdjustReason
  adjusted_by: string
  created_at: string
  profile?: Pick<Profile, 'name'> // join opcional
}
```

**Critério de conclusão:**
- `types/index.ts` exporta todos os tipos listados
- Nenhum arquivo do projeto usa `any` implícito para esses tipos
- `npm run build` passa sem erros de tipo

---

### F1-02 — Implementar a API de produtos (`GET /api/products`)

**Dependências:** F0-02, F0-03, F1-01

**O que fazer:**

- Criar `app/api/products/route.ts`:
  - `GET`: lista todos os produtos; aceita query param `?active=true` para filtrar apenas ativos
  - Retorna array de `Product` ordenado por `category ASC, name ASC`
  - Requer sessão autenticada (verificar com `createClient()` do servidor)
- Criar `app/api/products/[id]/route.ts`:
  - `GET`: retorna um produto pelo `id`
  - `PATCH`: atualiza `name`, `price`, `category` ou `active` — requer `role = 'dono'`
  - `DELETE`: define `active = false` (soft delete) — requer `role = 'dono'`
- Criar `lib/validations.ts` com schema Zod para produto:
  ```ts
  export const productSchema = z.object({
    name: z.string().min(1).max(100),
    price: z.number().min(0),
    category: z.string().min(1).max(50),
    active: z.boolean().optional(),
  })
  ```

**Critério de conclusão:**
- `GET /api/products` retorna `[]` (banco vazio) com status 200
- `GET /api/products?active=true` retorna apenas produtos com `active = true`
- Acesso sem sessão retorna 401
- `PATCH` e `DELETE` por funcionário retornam 403
- `npm run build` passa sem erros

---

## Fase 2 — Módulo 0: Gestão de Cardápio

> Pode ser iniciada após F0-04 a F0-06 e F1-01, F1-02 estarem concluídas.

---

### F2-01 — Implementar tela `/cardapio` (lista de produtos com toggle)

**Dependências:** F0-06, F1-01, F1-02

**O que fazer:**

- Criar `app/(app)/cardapio/page.tsx`:
  - Server Component que busca todos os produtos via `GET /api/products`
  - Exibe lista de cards empilhados, um por produto
  - Cada card mostra: **nome**, **preço** (formatado em R$), **categoria**, badge de status (Ativo / Inativo)
  - **Toggle ativo/inativo** em cada card:
    - Visível e clicável apenas para `role = 'dono'`
    - Funcionário vê o estado mas não pode alterar (toggle desabilitado)
    - Ao alternar, chama `PATCH /api/products/[id]` com `{ active: !current }`
    - Exibe toast de confirmação após sucesso
  - Botão **"+ Novo Produto"** no topo, visível apenas para donos; leva para `/cardapio/novo`
  - Toque no card (para donos) navega para `/cardapio/[id]`
  - Layout: coluna única, sem tabelas

**Critério de conclusão:**
- Lista exibe todos os produtos cadastrados
- Toggle funciona para donos e está desabilitado para funcionários
- Botão "+ Novo Produto" aparece apenas para donos
- Layout funciona corretamente em tela de celular (375px de largura)

---

### F2-02 — Implementar telas `/cardapio/novo` e `/cardapio/[id]`

**Dependências:** F2-01

**O que fazer:**

- Criar `app/(app)/cardapio/novo/page.tsx`:
  - Guard: redirecionar para `/cardapio` se `role = 'funcionario'`
  - Formulário com campos:
    - **Nome** (input texto, obrigatório)
    - **Preço** (input numérico, `inputMode="decimal"`, obrigatório, mínimo R$ 0,00)
    - **Categoria** (input texto livre, obrigatório)
    - **Ativo** (toggle/switch, padrão: ativo)
  - Botão **"Salvar"** (altura mínima 56px): chama `POST /api/products` e redireciona para `/cardapio` em caso de sucesso
  - Validação de formulário com Zod antes do submit
  - Exibe erros de validação inline
- Criar `app/(app)/cardapio/[id]/page.tsx`:
  - Guard: redirecionar para `/cardapio` se `role = 'funcionario'`
  - Formulário pré-preenchido com dados do produto (`GET /api/products/[id]`)
  - Mesmos campos que `/cardapio/novo`
  - Botão **"Salvar alterações"**: chama `PATCH /api/products/[id]`
  - Botão **"Desativar Produto"**: chama `DELETE /api/products/[id]` (define `active = false`) e redireciona para `/cardapio`
  - Exclusão física não é permitida — apenas desativação
- Criar `app/api/products/route.ts` método `POST` (complementar à F1-02):
  - Requer `role = 'dono'`
  - Valida payload com `productSchema`
  - Insere produto e retorna o registro criado

**Critério de conclusão:**
- Criação de produto funciona: produto aparece na lista após salvar
- Edição de produto funciona: alterações persistem
- Desativar produto define `active = false` e produto some do cardápio operacional
- Funcionário é redirecionado ao tentar acessar as telas de edição

---

## Fase 3 — Módulo 1: Registro de Pedidos

> Fase mais crítica. Pode ser iniciada após F2-01 estar concluída (cardápio populado).

---

### F3-01 — Implementar `POST /api/orders` (criar pedido)

**Dependências:** F0-03, F1-01

**O que fazer:**

- Criar `app/api/orders/route.ts` com:
  - `GET`: lista pedidos do dia; aceita `?date=YYYY-MM-DD` (padrão: hoje) e `?status=aberto`
    - Ordena por `created_at DESC`
    - Inclui contagem de itens por pedido
    - Retorna array de `Order`
  - `POST`: cria novo pedido
    - Payload: `{ type: OrderType, customer_name?: string }`
    - Valida que `type` é `'delivery' | 'retirada' | 'local'`
    - Cria registro com `status = 'aberto'`, `total = 0`
    - Retorna o pedido criado (incluindo `id`)
    - Requer sessão autenticada
- Criar schema Zod em `lib/validations.ts`:
  ```ts
  export const createOrderSchema = z.object({
    type: z.enum(['delivery', 'retirada', 'local']),
    customer_name: z.string().max(100).optional(),
  })
  ```

**Critério de conclusão:**
- `POST /api/orders` com `{ type: 'local' }` retorna o pedido criado com status 201
- `POST /api/orders` sem `type` retorna 400
- `GET /api/orders` retorna pedidos do dia atual
- `GET /api/orders?status=aberto` filtra corretamente

---

### F3-02 — Implementar `PATCH /api/orders/[id]` e `DELETE /api/orders/[id]`

**Dependências:** F3-01

**O que fazer:**

- Criar `app/api/orders/[id]/route.ts`:
  - `GET`: retorna pedido completo com `order_items` e dados do produto em cada item (join)
  - `PATCH`: atualiza pedido
    - Campos permitidos: `type`, `customer_name`, `status` (apenas transições válidas)
    - Transições de status permitidas: `aberto → em_preparo`, `em_preparo → aberto`
    - Pedidos `fechado` ou `cancelado` são imutáveis — retornar 409
    - Validar que `type` não é alterado em pedidos fechados (RN-07)
  - `DELETE`: cancela pedido — define `status = 'cancelado'`
    - Apenas pedidos com status `aberto` ou `em_preparo` podem ser cancelados
    - Pedidos já fechados retornam 409

**Critério de conclusão:**
- `PATCH` atualiza tipo e nome do cliente
- `DELETE` cancela pedido (status `cancelado`)
- Tentar editar pedido fechado retorna 409
- Tentativa de transição de status inválida retorna 400

---

### F3-03 — Implementar tela `/pedidos` com modal de seleção de tipo

**Dependências:** F0-06, F3-01

**O que fazer:**

- Criar `app/(app)/pedidos/page.tsx`:
  - Busca pedidos do dia via `GET /api/orders` (todos os status exceto `cancelado`)
  - Lista de cards empilhados, cada card mostra:
    - Número sequencial do dia (calculado via `ROW_NUMBER()` ou no cliente pela posição + 1 em order por `created_at`)
    - Tipo com ícone visual: Delivery 🛵, Retirada 🛍️, Local 🪑
    - Nome do cliente (se preenchido)
    - Status com badge colorido
    - Total parcial (R$)
  - Botão **"+ Novo Pedido"** fixo no topo (hauteur mínima 56px)
  - Ao tocar em "+ Novo Pedido": abre **modal de seleção de tipo** com 3 botões grandes:
    - **Delivery**, **Retirada**, **Local**
    - Cada botão tem ícone + label, altura mínima 64px
    - Ao selecionar: chama `POST /api/orders` com o `type`, aguarda resposta e redireciona para `/pedidos/[id]` do pedido criado
    - Modal tem botão "Cancelar" que fecha sem criar pedido
  - Toque em um card de pedido navega para `/pedidos/[id]`
  - Toast de erro se `?error=acesso_negado` estiver na URL

**Critério de conclusão:**
- Lista exibe pedidos do dia
- Modal de seleção aparece ao tocar em "+ Novo Pedido"
- Seleção de tipo cria pedido e redireciona para `/pedidos/[id]`
- Toque em card navega para edição do pedido

---

### F3-04 — Implementar componente `ProductGrid`

**Dependências:** F1-02

**O que fazer:**

- Criar `components/orders/ProductGrid.tsx`:
  - Props: `products: Product[]`, `onAdd: (product: Product) => void`
  - Agrupa produtos por `category`
  - Cada categoria é uma seção com título
  - Cada produto é um card com:
    - Nome do produto (destaque)
    - Preço em R$
    - Botão de adicionar: toque dispara `onAdd(product)`
    - Feedback visual ao adicionar (animação rápida ou flash)
  - Cards grandes o suficiente para toque confortável em celular
  - Layout em grid de 2 colunas em celular
  - Produtos com `active = false` não aparecem

**Critério de conclusão:**
- Componente renderiza sem erro quando passada uma lista de produtos
- Produtos agrupados por categoria
- `onAdd` é chamado corretamente ao tocar no card do produto

---

### F3-05 — Implementar componente `OrderItemRow` com campo de observações

**Dependências:** F1-01

**O que fazer:**

- Criar `components/orders/OrderItemRow.tsx`:
  - Props: `item: OrderItem & { product: Product }`, `onChangeQuantity: (id, delta) => void`, `onChangeObservations: (id, text) => void`, `onRemove: (id) => void`
  - Exibe: nome do produto, preço unitário, subtotal (`quantity × unit_price`)
  - Botões `−` e `+` para ajustar quantidade (altura mínima 44px cada)
  - Campo de texto **Observações** colapsado por padrão
    - Expandido ao tocar em "Adicionar observação"
    - Placeholder: "ex: sem alface, bem passado..."
    - `inputMode` padrão (texto)
    - Ao perder foco, salva automaticamente
  - Botão para remover o item do pedido (ícone de lixeira)

**Critério de conclusão:**
- Componente renderiza com botões `+` e `−`
- Campo de observações é colapsado por padrão e expande ao toque
- `onChangeQuantity` e `onChangeObservations` são chamados corretamente

---

### F3-06 — Implementar tela `/pedidos/[id]` (edição completa do pedido)

**Dependências:** F3-02, F3-03, F3-04, F3-05

**O que fazer:**

- Criar `app/(app)/pedidos/[id]/page.tsx`:
  - Carrega pedido via `GET /api/orders/[id]` (com itens e produtos)
  - Carrega cardápio via `GET /api/products?active=true`
  - **Seção Tipo:** seleção entre Delivery / Retirada / Local (botões grandes); editável enquanto status é `aberto` ou `em_preparo`; chama `PATCH /api/orders/[id]` ao alterar
  - **Campo nome do cliente:** visível e editável; mais relevante para Delivery
  - **Seção Cardápio:** renderiza `<ProductGrid>` com todos os produtos ativos
  - **Seção "Meu Pedido":** lista de itens usando `<OrderItemRow>`
    - Adicionar produto: insere em `order_items` via `POST /api/orders/[id]/items` (criar essa sub-rota)
    - Ajustar quantidade: atualiza `order_items` via `PATCH /api/orders/[id]/items/[itemId]`
    - Remover item: deleta `order_items` via `DELETE /api/orders/[id]/items/[itemId]`
    - Observações: atualiza campo `observations` no item
  - **Total parcial** atualizado em tempo real (calculado no cliente)
  - **Seção Pagamento:** seleção entre Dinheiro / Pix / Cartão (botões grandes, altura 56px)
  - **Botão "Fechar Pedido":** ativo apenas quando `items.length > 0` e `payment_method !== null`; chama `POST /api/orders/[id]/close`
  - **Botão "Cancelar Pedido":** disponível para pedidos `aberto` ou `em_preparo`; chama `DELETE /api/orders/[id]`; pede confirmação antes
  - Pedidos com status `fechado` ou `cancelado`: tela de visualização somente leitura
- Criar rotas de items aninhadas:
  - `app/api/orders/[id]/items/route.ts`: `POST` — adiciona item ao pedido
  - `app/api/orders/[id]/items/[itemId]/route.ts`: `PATCH` e `DELETE`

**Critério de conclusão:**
- Adicionar produto ao pedido funciona e lista atualiza
- Ajustar quantidade e observações funcionam e persistem
- Forma de pagamento selecionável
- Botão "Fechar Pedido" fica ativo apenas quando há itens E pagamento selecionado
- Cancelar pedido pede confirmação e muda status para `cancelado`
- Pedido fechado exibe tela somente leitura

---

### F3-07 — Implementar `POST /api/orders/[id]/close` (fechar pedido)

**Dependências:** F3-01, F0-03

**O que fazer:**

- Criar `app/api/orders/[id]/close/route.ts`:
  - `POST`: fecha o pedido
  - Validações obrigatórias no servidor (RN-01):
    1. Pedido deve ter status `aberto` ou `em_preparo` (caso contrário: 409)
    2. Deve ter ao menos 1 item em `order_items` (caso contrário: 400)
    3. `payment_method` deve estar definido no payload ou no registro (caso contrário: 400)
  - Payload opcional: `{ payment_method: PaymentMethod }` (pode já estar no registro)
  - Calcula `total = SUM(quantity × unit_price)` de todos os itens do pedido
  - Atualiza o pedido:
    ```sql
    UPDATE orders SET status = 'fechado', total = <calculado>, closed_at = NOW(), payment_method = <valor>
    WHERE id = <id>
    ```
  - Retorna o pedido atualizado

**Critério de conclusão:**
- `POST /api/orders/[id]/close` sem itens retorna 400
- `POST /api/orders/[id]/close` sem payment_method retorna 400
- Pedido válido é fechado: status `fechado`, `total` calculado, `closed_at` preenchido
- Tentar fechar pedido já fechado retorna 409

---

## Fase 4 — Módulo 2: Painel do Dia

> Pode ser iniciada após F3-07 estar concluída (sistema de pedidos completo).

---

### F4-01 — Implementar `GET /api/dashboard`

**Dependências:** F0-03, F1-01

**O que fazer:**

- Criar `app/api/dashboard/route.ts`:
  - `GET`: aceita query `?date=YYYY-MM-DD` (padrão: hoje)
  - Requer `role = 'dono'` (retorna 403 para funcionários)
  - Executa as seguintes queries no Supabase:

  **Totais gerais:**
  ```sql
  SELECT
    COUNT(*) FILTER (WHERE status = 'fechado')                                    AS total_pedidos,
    COALESCE(SUM(total) FILTER (WHERE status = 'fechado'), 0)                     AS total_dia,
    COALESCE(SUM(total) FILTER (WHERE status = 'fechado' AND payment_method = 'dinheiro'), 0) AS total_dinheiro,
    COALESCE(SUM(total) FILTER (WHERE status = 'fechado' AND payment_method = 'pix'), 0)      AS total_pix,
    COALESCE(SUM(total) FILTER (WHERE status = 'fechado' AND payment_method = 'cartao'), 0)   AS total_cartao
  FROM orders WHERE created_at::date = $1;
  ```

  **Top 5 produtos:**
  ```sql
  SELECT p.name,
         SUM(oi.quantity)               AS total_quantidade,
         SUM(oi.quantity * oi.unit_price) AS total_receita
  FROM order_items oi
  JOIN products p ON p.id = oi.product_id
  JOIN orders o   ON o.id = oi.order_id
  WHERE o.status = 'fechado' AND o.created_at::date = $1
  GROUP BY p.id, p.name
  ORDER BY total_quantidade DESC
  LIMIT 5;
  ```

  - Retorna JSON com: `{ total_pedidos, total_dia, ticket_medio, total_dinheiro, total_pix, total_cartao, top_produtos[] }`
  - `ticket_medio = total_pedidos > 0 ? total_dia / total_pedidos : 0`

**Critério de conclusão:**
- `GET /api/dashboard` retorna estrutura correta com dia sem pedidos (zeros)
- `GET /api/dashboard` retorna totais corretos com pedidos fechados no banco
- Funcionário recebe 403

---

### F4-02 — Implementar componentes do painel

**Dependências:** F1-01

**O que fazer:**

- Criar `components/dashboard/SummaryCard.tsx`:
  - Props: `title: string`, `value: string`, `subtitle?: string`
  - Card grande com título, valor em destaque, subtítulo opcional
  - Usado para: Total do Dia, Qtd. Pedidos, Ticket Médio

- Criar `components/dashboard/PaymentBreakdown.tsx`:
  - Props: `dinheiro: number`, `pix: number`, `cartao: number`, `total: number`
  - Exibe os 3 métodos com valor em R$ e percentual do total
  - Layout em coluna única (3 linhas separadas)

- Criar `components/dashboard/TopProductsList.tsx`:
  - Props: `produtos: { name: string, total_quantidade: number, total_receita: number }[]`
  - Lista numerada (1º ao 5º) com nome, quantidade e receita
  - Fontes legíveis em celular

**Critério de conclusão:**
- Componentes renderizam sem erro com dados de teste (mock)
- Layout funciona em tela de celular (375px)

---

### F4-03 — Implementar tela `/painel`

**Dependências:** F4-01, F4-02

**O que fazer:**

- Criar `app/(app)/painel/page.tsx`:
  - Guard de role: redireciona para `/pedidos` com `?error=acesso_negado` se `role = 'funcionario'`
  - Busca dados via `GET /api/dashboard?date=<data_selecionada>`
  - Seletor de data (padrão: hoje) no topo — ao alterar, refaz a busca
  - Exibe em sequência (coluna única):
    1. `<SummaryCard>` — Total do Dia
    2. `<SummaryCard>` — Qtd. de Pedidos + Ticket Médio
    3. `<PaymentBreakdown>` — Breakdown por Pagamento
    4. `<TopProductsList>` — Top 5 Produtos
  - Estado de loading durante a busca (skeleton ou spinner)

**Critério de conclusão:**
- Painel exibe dados corretos para hoje
- Trocar a data atualiza os dados
- Funcionário é redirecionado
- Layout correto em celular

---

## Fase 5 — Módulo 3: Estoque

> Pode ser iniciada após F0-03 e F1-01 estarem concluídas (banco pronto e tipos definidos).

---

### F5-01 — Implementar API de estoque (CRUD + ajuste)

**Dependências:** F0-03, F1-01

**O que fazer:**

- Criar `app/api/stock/route.ts`:
  - `GET`: lista todos os insumos; aceita `?low=true` para retornar apenas não perecíveis com `quantity <= min_quantity`
  - `POST`: cria novo insumo; requer sessão autenticada

- Criar `app/api/stock/[id]/route.ts`:
  - `GET`: retorna insumo por id
  - `PATCH`: atualiza `name`, `unit`, `min_quantity` ou `is_perishable`; valida que `quantity` não seja alterado por aqui (apenas via `/adjust`)
  - `DELETE`: remove insumo (exclusão física permitida se não houver ajustes — verificar antes)

- Criar `app/api/stock/[id]/adjust/route.ts`:
  - `POST`: registra ajuste de estoque
  - Payload: `{ delta: number, reason: AdjustReason }`
  - Validações:
    - `delta` não pode ser zero
    - `reason` deve ser `consumo`, `deterioracao` ou `reposicao`
    - Resultado (`quantity + delta`) não pode ser negativo — retornar 400
    - `reason = 'deterioracao'` apenas para insumos com `is_perishable = true`
  - Executa em transação (ou sequência atômica):
    1. Insere registro em `stock_adjustments` com `adjusted_by = user.id`
    2. Atualiza `stock_items.quantity += delta`
  - Retorna o insumo atualizado

- Criar schema Zod em `lib/validations.ts`:
  ```ts
  export const stockAdjustSchema = z.object({
    delta: z.number().refine(n => n !== 0, 'Delta não pode ser zero'),
    reason: z.enum(['consumo', 'deterioracao', 'reposicao']),
  })
  ```

**Critério de conclusão:**
- CRUD de insumos funciona via API
- Ajuste com `delta` negativo que resultaria em quantidade negativa retorna 400
- Ajuste com `reason = 'deterioracao'` em item não perecível retorna 400
- `stock_adjustments` registra `adjusted_by` corretamente

---

### F5-02 — Implementar tela `/estoque` (lista de insumos)

**Dependências:** F5-01, F0-06

**O que fazer:**

- Criar `app/(app)/estoque/page.tsx`:
  - Busca todos os insumos via `GET /api/stock`
  - Exibe lista de cards empilhados, um por insumo
  - Cada card mostra:
    - Nome do insumo
    - Quantidade atual + unidade (ex: "3 kg")
    - Badge **"Perecível"** para `is_perishable = true`
    - **Alerta visual** (borda vermelha + ícone ⚠️) quando `!is_perishable && quantity <= min_quantity`
    - Ações rápidas acessíveis direto no card:
      - Botão **"Baixa"**: abre `<AdjustStockModal>` com `reason = 'consumo'` pré-selecionado
      - Botão **"Deterioração"**: visível apenas para insumos perecíveis, abre o modal com `reason = 'deterioracao'`
      - Botão **"Editar"**: navega para `/estoque/[id]`
  - Botão **"+ Novo Insumo"** no topo; navega para `/estoque/novo`

**Critério de conclusão:**
- Lista exibe insumos com alertas para os não perecíveis abaixo do mínimo
- Perecíveis não exibem alerta
- Botão "Deterioração" aparece apenas para perecíveis
- Layout correto em celular

---

### F5-03 — Implementar telas `/estoque/novo` e `/estoque/[id]`

**Dependências:** F5-01

**O que fazer:**

- Criar `app/(app)/estoque/novo/page.tsx`:
  - Formulário com campos:
    - **Nome** (texto, obrigatório)
    - **Quantidade Atual** (`inputMode="decimal"`, obrigatório, mín. 0)
    - **Unidade** (texto livre, ex: kg, un, L, cx)
    - **Quantidade Mínima** (`inputMode="decimal"`, mín. 0; irrelevante se perecível, mas ainda editável)
    - **Perecível** (toggle/switch)
  - Botão **"Salvar"** (altura mínima 56px): chama `POST /api/stock`
  - Redireciona para `/estoque` após salvar com toast de sucesso

- Criar `app/(app)/estoque/[id]/page.tsx`:
  - Pré-preenchido com dados do insumo (`GET /api/stock/[id]`)
  - Mesmos campos que `/estoque/novo`
  - Botão **"Salvar"**: chama `PATCH /api/stock/[id]`
  - Botão **"Excluir Insumo"**: chama `DELETE /api/stock/[id]`, pede confirmação
  - Seção "Histórico de Ajustes" (opcional na V1): lista os últimos 10 ajustes do insumo com data, delta, motivo e quem fez

**Critério de conclusão:**
- Cadastro de novo insumo funciona: aparece na lista
- Edição persiste corretamente
- Toggle perecível funciona

---

### F5-04 — Implementar componente `AdjustStockModal`

**Dependências:** F5-01, F1-01

**O que fazer:**

- Criar `components/stock/AdjustStockModal.tsx`:
  - Props: `item: StockItem`, `defaultReason?: AdjustReason`, `onClose: () => void`, `onSuccess: (updated: StockItem) => void`
  - Modal (usando `Dialog` do shadcn/ui)
  - Campos:
    - **Quantidade** (`inputMode="decimal"`, obrigatório, positivo)
    - **Tipo de ajuste:** Entrada (reposição) ou Saída (consumo/deterioração) — toggle visual
    - **Motivo:** seleção entre Consumo / Deterioração / Reposição
      - Deterioração visível apenas se `item.is_perishable = true`
      - Lógica: Entrada → `reason = 'reposicao'`, `delta = +valor`; Saída → `reason = motivo_selecionado`, `delta = -valor`
  - Preview do novo saldo: "Saldo atual: X → Novo saldo: Y" (calculado no cliente)
  - Botão **"Confirmar"** (56px): chama `POST /api/stock/[id]/adjust`
  - Exibe erro se saldo ficaria negativo

**Critério de conclusão:**
- Modal abre e fecha corretamente
- Preview de saldo atualiza em tempo real
- Confirmar chama a API e fecha o modal em caso de sucesso
- Saldo negativo bloqueado no cliente e no servidor

---

## Fase 6 — Módulo 4: Gestão de Usuários

> Pode ser iniciada após F0-03 e F0-04 estarem concluídas.

---

### F6-01 — Implementar `GET /api/users` e `POST /api/users`

**Dependências:** F0-03, F0-04, F1-01

**O que fazer:**

- Criar `app/api/users/route.ts`:
  - `GET`: lista todos os usuários da tabela `profiles`; requer `role = 'dono'`; retorna array de `Profile`
  - `POST`: cria novo usuário; requer `role = 'dono'`
    - Payload: `{ name: string, email: string, role: Role }`
    - Usa `supabaseAdmin.auth.admin.createUser({ email, password: undefined, email_confirm: true, user_metadata: { name, role } })` para criar usuário no Auth
    - O trigger `on_auth_user_created` insere automaticamente em `profiles`
    - Retorna o `Profile` criado

- Criar schema Zod:
  ```ts
  export const createUserSchema = z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    role: z.enum(['funcionario', 'dono']),
  })
  ```

**Critério de conclusão:**
- `GET /api/users` retorna lista de usuários para donos e 403 para funcionários
- `POST /api/users` cria usuário no Supabase Auth e o registro aparece em `profiles`
- Usuário criado recebe e-mail de confirmação/definição de senha (fluxo padrão Supabase)

---

### F6-02 — Implementar `PATCH /api/users/[id]` (atualizar role e desativar)

**Dependências:** F6-01

**O que fazer:**

- Criar `app/api/users/[id]/route.ts`:
  - `PATCH`: atualiza `role` ou desativa o usuário; requer `role = 'dono'`
  - Payload: `{ role?: Role, active?: boolean }`
  - **Validações de segurança obrigatórias:**
    1. Não é possível desativar o próprio usuário (comparar `id` com o `user.id` da sessão) — retornar 400
    2. Não é possível remover o último dono ativo: verificar `COUNT(*) WHERE role = 'dono' AND active = true` antes de desativar qualquer dono — retornar 400 se resultado seria zero
  - **Ao desativar** (`active = false`):
    1. Atualizar `profiles.active = false`
    2. Chamar `supabaseAdmin.auth.admin.updateUserById(id, { ban_duration: '876000h' })` (banimento efetivo permanente)
  - **Ao reativar** (`active = true`):
    1. Atualizar `profiles.active = true`
    2. Chamar `supabaseAdmin.auth.admin.updateUserById(id, { ban_duration: 'none' })` (remover banimento)

**Critério de conclusão:**
- Desativar usuário bloqueia o login desse usuário
- Reativar restaura o acesso
- Não é possível desativar a si mesmo (retorna 400)
- Não é possível desativar o último dono (retorna 400)

---

### F6-03 — Implementar tela `/usuarios` com `CreateUserModal`

**Dependências:** F6-01, F6-02, F0-06

**O que fazer:**

- Criar `app/(app)/usuarios/page.tsx`:
  - Guard: redireciona para `/pedidos` com `?error=acesso_negado` se `role = 'funcionario'`
  - Busca usuários via `GET /api/users`
  - Exibe lista de cards empilhados, cada card com:
    - Nome e e-mail do usuário
    - Badge de perfil: **Dono** ou **Funcionário**
    - Badge de status: **Ativo** / **Inativo**
    - Botão **"Desativar"** (para usuários ativos, não para si mesmo)
    - Botão **"Reativar"** (para usuários inativos)
  - Botão **"+ Novo Usuário"** no topo — abre `<CreateUserModal>` inline

- Criar `components/users/CreateUserModal.tsx`:
  - Campos: **Nome**, **E-mail**, **Perfil** (seleção: Funcionário / Dono)
  - Botão **"Criar Usuário"**: chama `POST /api/users` e fecha modal em caso de sucesso
  - Exibe erro inline em caso de falha (ex: e-mail já cadastrado)
  - Toast de sucesso: "Usuário criado. Um e-mail de acesso foi enviado."

**Critério de conclusão:**
- Lista de usuários exibe todos os perfis
- Criar usuário via modal funciona e aparece na lista
- Desativar e reativar funcionam com feedback visual imediato
- Usuário logado não vê botão "Desativar" no próprio card

---

## Fase 7 — Polimento e Deploy

> Executar após todas as fases anteriores estarem concluídas e o sistema funcional.

---

### F7-01 — Revisão de UX mobile

**Dependências:** todas as fases anteriores

**O que fazer:**

- Auditar todas as telas em um celular real ou DevTools (viewport 375px):
  - [ ] Verificar que todos os botões de ação têm altura mínima de 56px
  - [ ] Verificar que não há tabelas com scroll horizontal — substituir por cards onde encontrado
  - [ ] Verificar que campos de quantidade usam `inputMode="decimal"` ou `inputMode="numeric"`
  - [ ] Verificar que o BottomNav não cobre conteúdo (padding-bottom adequado)
  - [ ] Verificar que textos são legíveis (mínimo 16px para corpo, 20px+ para valores em destaque)
  - [ ] Verificar que toasts aparecem acima do BottomNav
  - [ ] Testar fluxo completo de pedido em celular: abrir → selecionar tipo → adicionar itens → observações → pagamento → fechar
  - [ ] Verificar que estados de loading são exibidos em todas as ações de rede

**Critério de conclusão:**
- Nenhum elemento crítico com altura abaixo de 56px
- Nenhuma tabela com scroll horizontal
- Teclado numérico ativado em campos de quantidade e preço
- Fluxo completo de pedido executável em 3 toques principais

---

### F7-02 — Configurar hospedagem na Vercel e conectar ao GitHub

**Dependências:** F7-01

**O que fazer:**

- Criar repositório no GitHub (se ainda não existir): `sistema-sao-joao-batista`
- Fazer o primeiro push do código
- Acessar [vercel.com](https://vercel.com) e criar novo projeto apontando para o repositório
- Configurar o framework como **Next.js** (detecção automática)
- Configurar **todas** as variáveis de ambiente na interface da Vercel (Settings → Environment Variables):
  - `NEXT_PUBLIC_SUPABASE_URL`
  - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
  - `SUPABASE_SERVICE_ROLE_KEY`
  - `NEXT_PUBLIC_APP_NAME`
- Garantir que o deploy automático está configurado (push na branch `main` → deploy automático)

**Critério de conclusão:**
- Primeiro deploy bem-sucedido na Vercel (sem erros de build)
- URL de produção acessível
- Variáveis de ambiente configuradas (verificar no painel da Vercel)

---

### F7-03 — Testes em produção

**Dependências:** F7-02

**O que fazer:**

- Acessar a URL de produção e executar o seguinte roteiro de testes:
  - [ ] Login com e-mail e senha de um dono
  - [ ] Criar produto no cardápio
  - [ ] Criar pedido local, adicionar 2 itens com observações, fechar com pagamento em Pix
  - [ ] Criar pedido delivery com nome do cliente, adicionar itens, fechar com dinheiro
  - [ ] Acessar /painel e verificar os totais
  - [ ] Criar insumo, registrar baixa por consumo, registrar baixa por deterioração
  - [ ] Criar usuário funcionário, logar com ele e verificar acesso restrito ao /painel
  - [ ] Desativar e reativar o usuário funcionário
- Documentar qualquer bug encontrado para correção

**Critério de conclusão:**
- Todos os itens do roteiro executados sem erro
- Sistema funcionando de forma confiável na URL de produção

---

### F7-04 — Onboarding dos usuários

**Dependências:** F7-03

**O que fazer:**

- Criar os usuários dos 3 operadores reais no sistema (via /usuarios):
  - 2 donos com `role = 'dono'`
  - 1 funcionário com `role = 'funcionario'`
- Garantir que todos receberam o e-mail de definição de senha e conseguiram acessar
- Fazer uma sessão rápida de uso assistido com o funcionário:
  - Abrir pedido, adicionar itens, selecionar pagamento, fechar
  - Consultar cardápio
  - Registrar baixa de estoque
- Popular o cardápio com todos os produtos reais da lanchonete
- Popular o estoque com todos os insumos reais e suas quantidades iniciais

**Critério de conclusão:**
- Os 3 usuários reais conseguem fazer login
- Cardápio populado com produtos reais
- Estoque populado com insumos reais e quantidades iniciais registradas
- Funcionário consegue executar o fluxo de pedido sem assistência

---

*Documento gerado em 2026-03-14. Derivado do PROJECT.md v1.1.*

# Raízes do Nordeste — API Back-end

Projeto Multidisciplinar (Uninter) — **trilha Back-end**  
**RU:** 4825877

API REST da rede de lanchonetes **Raízes do Nordeste**, com foco no **Fluxo A**: criar pedido → pagamento mock → atualização de status. Multicanalidade via campo obrigatório `canalPedido`.

## Stack

- Node.js 20+
- NestJS
- PostgreSQL 16 (banco relacional — ver opções de instalação abaixo)
- Prisma 5 (ORM + migrations + seed)
- JWT + roles
- Swagger/OpenAPI em `/api`

## O que precisa de Docker?

| Componente | Precisa de Docker? | Como roda |
|---|---|---|
| **API (NestJS)** | **Não** | `npm install` + `npm run start:dev` |
| **Banco PostgreSQL** | **Não** (mas precisa existir) | Docker **ou** PostgreSQL instalado na máquina |
| **Testes Postman / Newman** | **Não** | Coleção contra `http://localhost:3000` |

**Resumo:** Docker **não é obrigatório**. Ele só facilita subir o PostgreSQL sem instalar nada além do Node. Se o corretor já tiver PostgreSQL (local, servidor da faculdade, etc.), basta apontar o `DATABASE_URL` no `.env` e seguir com `migrate` + `seed`.

## Pré-requisitos

**Obrigatórios (sempre):**

- Node.js 20 ou superior
- npm
- Git
- **PostgreSQL 14+** acessível (via Docker **ou** instalação nativa)

**Opcionais:**

- Docker Desktop / Docker Engine + Compose — recomendado para quem não tem PostgreSQL instalado

## Subir o ambiente (professor / corretor)

### Opção A — com Docker (caminho mais simples)

Use esta opção se você **não** tem PostgreSQL instalado na máquina. O Docker sobe **somente o banco**; a API continua rodando com `npm`.

```bash
git clone https://github.com/rmvillela/raizes-nordeste.git
cd raizes-nordeste
cp .env.example .env
npm install
npm run docker:up          # sobe PostgreSQL na porta 5432
npx prisma migrate deploy
npm run db:seed
npm run start:dev
```

Para derrubar o banco: `npm run docker:down`

### Opção B — sem Docker (PostgreSQL já instalado)

Use esta opção se você **já tem PostgreSQL** (Windows, macOS ou Linux) e **não quer usar Docker**.

1. Crie o banco e o usuário (ajuste se preferir outro nome/senha):

```sql
CREATE USER raizes WITH PASSWORD 'raizes';
CREATE DATABASE raizes_nordeste OWNER raizes;
```

2. Configure o `.env` com a sua conexão:

```env
DATABASE_URL=postgresql://raizes:raizes@localhost:5432/raizes_nordeste?schema=public
```

Se usar o usuário padrão `postgres` do seu SO, altere usuário e senha na URL.

3. Suba a API (sem nenhum comando Docker):

```bash
git clone https://github.com/rmvillela/raizes-nordeste.git
cd raizes-nordeste
cp .env.example .env    # depois edite DATABASE_URL se necessário
npm install
npx prisma migrate deploy
npm run db:seed
npm run start:dev
```

### Verificar se está funcionando

| URL / comando | Resultado esperado |
|---|---|
| http://localhost:3000/api | Swagger abre (HTTP 200) |
| `GET /unidades` sem token | HTTP 401 |
| `POST /auth/login` com `cliente@raizes.local` / `Senha@123` | HTTP 201 + `accessToken` |

- API: http://localhost:3000
- Swagger: http://localhost:3000/api
- Coleção Postman: `postman/Raizes-Nordeste-Fluxo-A.postman_collection.json`
- Ambiente Postman: `postman/Raizes-Nordeste-local.postman_environment.json`
- Evidências de testes: `docs/evidencias/`

### Ordem sugerida na coleção Postman

1. Importar a coleção e o environment `Raizes Nordeste — local`.
2. Garantir que a API está no ar (`npm run start:dev`) e o seed foi aplicado.
3. Rodar as pastas nesta ordem: Auth → Catálogo → Pedidos → Pagamentos → Status → Erros.
4. O script da coleção grava `accessToken`, `unidadeId`, `produtoId` e `pedidoId` nas variáveis do environment.

### Variáveis de ambiente

Copie `.env.example` → `.env`.

**Com Docker (opção A):** os valores abaixo já batem com o `docker-compose.yml`.

**Sem Docker (opção B):** altere `DATABASE_URL` para o host, usuário e senha do seu PostgreSQL.

| Variável | Exemplo |
|---|---|
| `DATABASE_URL` | `postgresql://raizes:raizes@localhost:5432/raizes_nordeste?schema=public` |
| `JWT_SECRET` | string qualquer para desenvolvimento |
| `JWT_EXPIRES_IN` | `1d` |
| `PORT` | `3000` |

**Não versionar** o arquivo `.env`.

## Usuários do seed

Senha de todos: `Senha@123`

| E-mail | Role |
|---|---|
| admin@raizes.local | ADMIN |
| gerente@raizes.local | GERENTE |
| atendente@raizes.local | ATENDENTE |
| cozinha@raizes.local | COZINHA |
| cliente@raizes.local | CLIENTE |

O seed também cria 1 unidade (Recife Centro) e 2 produtos.

## Fluxo A (manual via Swagger ou HTTP)

1. `POST /auth/login` com `cliente@raizes.local` → copiar `accessToken`
2. `GET /unidades` e `GET /produtos?unidadeId=...` (Authorize Bearer)
3. `POST /pedidos` com `canalPedido`, `unidadeId` e `itens`
4. `POST /pedidos/{id}/pagamentos` com `{ "aprovado": true }`
5. Login como `cozinha@raizes.local` → `PATCH /pedidos/{id}/status` com `EM_PREPARO`, depois `PRONTO`
6. Login como `atendente@raizes.local` → `PATCH` para `ENTREGUE`

Filtro multicanal: `GET /pedidos?canalPedido=TOTEM`

## Estrutura (camadas)

```
src/
  domain/           # regras de domínio (ex.: transições de status)
  application/      # casos de uso / serviços de aplicação
  infrastructure/   # Prisma, JWT strategy, auditoria
  api/              # controllers + DTOs (Swagger)
  common/           # filters, guards, decorators
  modules/          # módulos Nest
```


## Scripts úteis

| Script | Ação |
|---|---|
| `npm run docker:up` | Sobe PostgreSQL no Docker (opção A) |
| `npm run docker:down` | Para o container do PostgreSQL |
| `npm run db:deploy` | Aplica migrations |
| `npm run db:seed` | Dados demo |
| `npm run start:dev` | API em watch |
| `npm run build` | Compila |
| `npm run lint` | ESLint |

## Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `Can't reach database server` | PostgreSQL não está rodando | Opção A: `npm run docker:up`. Opção B: iniciar o serviço PostgreSQL do SO |
| `port 5432 already in use` | Outro Postgres na máquina | Use o Postgres existente (opção B) ou pare o outro serviço |
| `401` em todas as rotas | Token ausente ou seed não rodou | `npm run db:seed` e faça login em `/auth/login` |
| Docker não instalado | — | Use a **opção B** com PostgreSQL nativo — não é impeditivo |

## Escopo entregue

Auth JWT com papéis, catálogo de unidades e produtos, pedidos com `canalPedido`, pagamento mock (aprovado, recusado e falha), atualização de status, erro JSON padronizado, registro de auditoria nas ações do fluxo, Swagger, coleção Postman e diagramas em `docs/diagramas/`.



## Licença

Uso acadêmico — Uninter 2026.

# Playbook Técnico Trilhar

Princípios, arquitetura, padrões e processo pra construir software robusto.

**Versão:** 1.0 (2026-05-13)
**Origem:** destilação de lições aprendidas durante a construção do Kenji
(plataforma de análise financeira IA), de janeiro a maio de 2026.
**Audiência:** desenvolvedores Trilhar (humanos + Claude) que iniciam ou
trabalham em projetos da organização.
**Status:** documento vivo — atualizado conforme novos projetos revelam
padrões reusáveis ou anti-patterns.

---

## Como ler este documento

Este playbook não substitui julgamento técnico — é base de partida.
Princípios são transferíveis entre projetos; detalhes de implementação
podem variar por stack ou domínio. A regra geral é: se vai divergir do
playbook em algo significativo, documenta o porquê em ADR do projeto.
Divergência sem rationale escrito é dívida técnica futura.

Cada seção segue uma estrutura consistente: princípio nuclear (o porquê),
como aplicar (com exemplo concreto do Kenji quando faz sentido),
anti-pattern correspondente (o que NÃO fazer) e quando flexibilizar
(situações onde o princípio falha). Quando uma seção não tem alguma das
partes, é porque não há ambiguidade real — o princípio simplesmente vale.

Os exemplos do Kenji são marcados com tags `[ex: Kenji]`. Eles existem
porque são casos reais, validados em produção; não invente exemplos
sintéticos pra preencher.

## Índice

1. [Filosofia geral](#1-filosofia-geral)
2. [Arquitetura recomendada](#2-arquitetura-recomendada)
3. [Stack tech recomendada](#3-stack-tech-recomendada)
4. [Code quality](#4-code-quality)
5. [Testing strategy](#5-testing-strategy)
6. [Database & Migrations](#6-database--migrations)
7. [Security baseline](#7-security-baseline)
8. [Auth & multi-user](#8-auth--multi-user)
9. [Frontend patterns](#9-frontend-patterns)
10. [Integrações externas](#10-integrações-externas)
11. [Process patterns](#11-process-patterns)
12. [Anti-patterns](#12-anti-patterns)
13. [Como começar um projeto do zero](#13-como-começar-um-projeto-do-zero)

---

## 1. Filosofia geral

### 1.1 Two-Gate triplo-ator

**Princípio:** trabalho técnico complexo passa por 3 atores com
responsabilidades distintas: **planner**, **operador** e **executor**.

- **Planner (Claude em chat):** vê o macro. Audita estado, propõe
  arquitetura, identifica decisões pendentes, antecipa riscos. Não
  escreve código diretamente — escreve planos que podem ser executados.
- **Operador (humano, Cauê):** valida intenção. Confirma que o plano
  resolve o problema real, não outro. Aprova decisões pendentes.
  Decide trade-offs que envolvem valor de negócio (custo, prazo,
  prioridade).
- **Executor (Claude Code):** implementa sem ambiguidade. Aplica DIFFs
  exatos, roda typecheck, roda tests, reporta resultado factual. Não
  toma decisões arquiteturais — pergunta quando ambiguidade aparece.

**Por que funciona:** cada ator atua em camada diferente do problema.
Planner vê 5-10 passos à frente sem se prender em sintaxe. Operador
ancora o trabalho na realidade do negócio. Executor garante precisão
mecânica que humano cansa de manter.

**Como aplicar:**
- Operador escreve prompt curto explicando objetivo
- Planner produz plano (FASE A audit + FASE B+C plano)
- Operador revisa plano + aprova decisões
- Planner especifica DIFFs pra executor
- Executor aplica + reporta
- Operador valida resultado

**[ex: Kenji]** Validado em 16+ sub-colas durante Fase 4 e Refinamento
V1.0. Zero falha procedural com pattern aplicado de forma disciplinada.
A maioria das falhas observadas no Kenji veio de tentativas de pular
um dos 3 atores ("já sei o que fazer, não preciso plano").

**Anti-pattern:** misturar papéis. "Claude, escreve isso direto" sem
plano vira retrabalho. "Vou só ajustar à mão" sem updating de tests
gera drift entre código e cobertura.

### 1.2 Micro-turnos

**Princípio:** divide trabalho grande em sub-colas isoladas. Cada sub-cola
tem entrada clara, saída clara, validação clara. Sem mega-turnos
processando 5 coisas ao mesmo tempo.

**Estrutura típica:**
- **FASE A audit:** lê estado atual, identifica gaps, sem propor mudança
- **FASE B+C plano:** apresenta DIFFs propostos + decisões a fechar +
  riscos identificados
- **FASE D execução em sub-colas D1, D2, D3...:** uma mudança ou um
  conjunto coerente de mudanças por sub-cola

**Por que funciona:** sub-cola pequena é validável. Output de Claude (em
chat ou Code) tem limite prático de qualidade que cai com tamanho. STDOUT
muito longo trunca. Atenção do operador também cansa.

**[ex: Kenji]** A Fase 3 do Kenji teve 7 falhas de STDOUT em mensagens
muito longas. A decisão `D-Fase4-9` formalizou micro-turnos como padrão
pra Fase 4 — eliminou recorrência. Custo: mais turnos, mais latência.
Benefício: zero retrabalho, validação granular.

**Quando flexibilizar:** mudanças triviais (typo em comment, ajuste de
log) não precisam FASE A. Aplica direto. O peso do processo deve ser
proporcional ao risco da mudança.

### 1.3 Read-state-before-change

**Princípio:** SEMPRE auditar estado atual antes de propor mudança. Sem
isso, propostas batem em surpresas — código que já existe, schema que
diverge, dependência circular não-óbvia.

**Como aplicar:**
- Antes de propor migration: ver migrations existentes
- Antes de propor função nova: grep por nome similar
- Antes de propor refactor: entender consumers do código a refatorar
- Antes de propor schema novo: ver se já existe equivalente

**[ex: Kenji]** A Sub-onda 4.1 do Kenji começou com FASE A audit do
schema. Descoberta crítica: as 4 camadas de memória (events,
recommendations, profile, patterns) JÁ EXISTIAM como migrations criadas
em 2026-04-23, antes da Fase 3 entrar. Sem o audit, a proposta original
criaria tabelas duplicadas (`client_decisions` paralela a
`recommendations` que já existia). O reescopo pós-audit eliminou ~40%
do trabalho previsto. Documentado em ADR 0011.

**Anti-pattern:** propor mudança baseado em memória ou em "achismo"
(`acho que ainda não existe X`). Custo de FASE A audit é 5-15 minutos.
Custo de reverter mudança duplicada ou conflitante é horas.

### 1.4 Honestidade epistêmica

**Princípio:** não simular dados / cliente / cenário fake quando o real
não está disponível. Documentar diferimento explícito é melhor que
fingir validação.

**Como aplicar:**
- Calibração IA exige cliente real? Diferir até cadastro acontecer
- Test E2E exige API externa estável? Marcar como pendente, não mockar
  o externo "como se" funcionasse
- Validação requer Studio manual? Listar checklist pra operador, não
  inferir resultado

**[ex: Kenji]** Calibrações 4.3 e 4.4 do Kenji foram diferidas porque
o banco estava vazio (sem clientes seedados). Em vez de criar cliente
fake e fingir validar, documentamos no ADR 0014 a diferimento + custo
estimado pra primeira execução real. A calibração ocorreu naturalmente
quando o seed foi implementado e Resolucar foi cadastrado.

**Anti-pattern:** "calibrei mockado, deu OK" quando o mock não tem
nenhuma das características que o real vai ter. Pior que não calibrar.

### 1.5 Decisões formalizadas

**Princípio:** toda decisão arquitetural ou de design não-óbvia ganha
ID + decisão final + rationale curto. Vai pra commit message + ADR
consolidador.

**Convenção:**
- `D-IA-{N}` pra decisões dentro de uma sub-onda focada em IA
- `D-{Fase}-{N}` pra decisões dentro de uma fase específica
- `D-{Tema}-{N}` pra sub-ondas focadas (ex: D-Refino-1, D-Seed-3)

**Por que funciona:** seis meses depois, quando alguém pergunta "por que
isso é assim", a resposta está no ADR ou no commit. Reduz custo de
re-discutir decisões já tomadas.

**[ex: Kenji]** Fase 4 V1.0 formalizou 53 decisões (D-Fase4-1 a 9 +
D-IA-75 a 80 + 4.2-D1 a 12 + 4.3-D1 a 15 + 4.4-D1 a 11). A Sub-onda
Refinamento V1.0 formalizou 8 D-Refino. A Sub-onda Seed formalizou 13
D-Seed. Todas referenciadas em ADR 0014.

**Quando flexibilizar:** decisões triviais (escolha de nome de variável,
detalhe de formatação) não precisam ID. Reserva o esforço pra decisões
que afetam interface ou comportamento.

### 1.6 ADRs consolidadores no fim de cada fase

**Princípio:** ao fechar uma fase, escreve ADR consolidador documentando
contexto, decisões finais, métricas, lições aprendidas e TODOs herdados
pra fase próxima.

**Estrutura típica:**
- Cabeçalho: data, sub-ondas cobertas, commits, custo IA, ADRs prévios
- Contexto: o que motivou a fase
- Arquitetura final: diagrama + descrição
- Cronologia: tabela de sub-ondas + datas + LOC + tests
- Decisões D-{...}: lista todas com 1-2 linhas cada
- TODOs V1.1+: o que foi adiado conscientemente
- Validações pendentes pré-produção: o que falta validar
- Lições aprendidas: o que funcionou + falhas operacionais observadas
- Métricas finais: tabela
- Próximos passos: critérios pra próxima fase

**[ex: Kenji]** ADRs 0010-0014 documentam Fase 3, Sub-onda 3.2, Fase 4
e refinamentos. Servem de referência quando entra novo dev / agente OU
quando próxima fase precisa contexto histórico.

**Anti-pattern:** terminar fase sem ADR. Lições viram cinza no chat
abandonado depois de 2 meses. Custo pequeno (1-2h por ADR) vs benefício
permanente.

---

## 2. Arquitetura recomendada

### 2.1 Multi-camada com responsabilidade clara

**Princípio:** sistema bem-arquitetado tem camadas com responsabilidades
disjuntas. Camada N depende apenas das camadas N-1, N-2, ... — nunca
N+1. Mudança em uma camada não cascateia além das interfaces conhecidas.

**[ex: Kenji]** Estrutura em 4 camadas:

```
Camada 1 (Read): Edge Function read-data lê Trilhar Finance
   ↓
Camada 2 (Compute): Pipeline calcula Dossier (47 indicadores) + persiste
   ↓
Camada 3 (Multi-agente IA): 4 specialists + orquestrador + crítico geram KenjiAnalysis
   ↓
Camada 4 (Memória): loadClientMemory + extractRecommendations alimentam ciclo cross-mês
```

Cada camada tem:
- Pasta dedicada em `src/`
- Tipos próprios + tipos exportados pra camada acima consumir
- Tests isolados — Camada N testa contra mocks da N-1

**Como aplicar em projeto novo:**
- Mapear domínio em 3-5 camadas máximo (mais que isso vira spaghetti
  vertical)
- Definir contratos entre camadas via TypeScript types
- Tests por camada — não tests E2E como única cobertura

**Anti-pattern:** "service" + "controller" + "model" sem disciplina vira
plate of spaghetti. Toda função vira "X service" que faz tudo.

### 2.2 Repository pattern

**Princípio:** todo acesso a dados (banco, API externa, file system)
isolado em pasta `src/repository/` ou `src/integrations/`. Resto do
código consome funções tipadas, sem importar SDK genérico (Supabase JS,
fetch wrapper, etc.) diretamente.

**Como aplicar:**
- Função pública por operação: `getClient(id)`, `listAllClients()`,
  `insertRecommendation(input)`, `transitionStatus(input)`
- Cada função retorna tipo do projeto, não shape genérico do SDK
- Errors tipados (subclasses de DatabaseError ou similar)
- Cast `as unknown as TipoX` no boundary, com comment explicando
- Sem lógica de domínio — só queries + mapping

**[ex: Kenji]** `src/repository/clients.ts` tem 5 funções públicas
(getClient, listAll, listActiveWithMapping, insertClient,
updateClientFromFinance). Tipos `ClientRecord`, `InsertClientInput`.
Errors `ClientNotFoundError`, `ClientNotMappedError`. Caller (delivery
layer) NUNCA importa `@supabase/supabase-js` diretamente.

**Por que funciona:** trocar Supabase por Drizzle ORM amanhã afeta
APENAS `src/repository/`. Resto do código continua funcionando porque
a interface (`getClient(id): Promise<ClientRecord>`) não mudou.

**Anti-pattern:** importar `supabase.from("clients").select(...)`
direto em route handler ou em business logic. Quando precisar mudar
schema, vai ter que tocar 50 arquivos em vez de 1.

### 2.3 Module testável vs entry-point CLI

**Princípio:** scripts CLI são thin wrappers (15-50 LOC). Toda lógica
testável fica em `src/` com tests dedicados. Script só orquestra:
parse args → chama lógica → formata output.

**[ex: Kenji]** `scripts/seed-clientes.ts` (84 LOC) é 100% formatação
de output e parsing de `--dry-run`. A lógica `runSeedClients()` vive em
`src/integrations/trilhar-finance/seed-clients.ts` (175 LOC) com test
file dedicado (250 LOC, 11 cenários).

**Por que funciona:**
- Script é trivial — não merece test (e seria difícil testar)
- Lógica é composável — pode ser chamada de outro contexto (cron, API
  endpoint) sem reescrever
- Tests cobrem comportamento, não orquestração

**Anti-pattern:** script de 300 LOC com lógica + IO + formatação tudo
junto. Impossível testar. Quando bugar em produção, debug é tentar
reproduzir condições do script localmente.

### 2.4 Adapter pattern pra integrações externas

**Princípio:** cada serviço externo (API, fila, Edge Function de outro
projeto) ganha pasta dedicada em `src/integrations/<servico>/` com:
- `client.ts` — HTTP/SDK client baixo nível
- `types.ts` — types do domínio do serviço externo
- `errors.ts` — hierarquia de erros tipados
- `readers/` ou métodos por entidade
- `index.ts` — re-export público enxuto

**Por que funciona:** mudança de URL, key, ou shape de resposta afeta
APENAS a pasta da integração. Consumers usam interface estável.

**[ex: Kenji]** `src/integrations/trilhar-finance/` tem client (290 LOC,
HTTP com auth dupla, retry, paginação, timeout 30s) + 8 readers
tipados + types + errors. Quando o Trilhar Finance trocou shape de
`Company.cnpj` (string → string|null), apenas 1 type ajustado.

**Como aplicar:**
- 1 integração = 1 pasta. Não misturar dois serviços externos no
  mesmo cliente.
- Auth via env vars validadas no boot do client (lança erro tipado se
  faltar)
- Retry + timeout + logging dentro do cliente, não espalhado em quem
  consome

### 2.5 Promise.allSettled pra operações independentes paralelas

**Princípio:** quando N operações independentes podem falhar
individualmente sem invalidar o todo, usa Promise.allSettled. Captura
falhas em array tipado, segue com o que deu certo.

**Como aplicar:**
```typescript
const results = await Promise.allSettled([opA(), opB(), opC()]);
const successful = [];
const failed = [];
for (const r of results) {
  if (r.status === "fulfilled") successful.push(r.value);
  else failed.push(r.reason);
}
return { successful, failed };
```

**[ex: Kenji]** Pattern usado em 3 contextos:
- `runSpecialists` (4 specialists IA em paralelo — 1 falha não invalida
  os 3)
- `loadClientMemory` (3 fontes de memória — 1 fonte falha vira
  partialFailures, throw apenas se 3/3 falham)
- `extractRecommendations` (N inserts — falha 1 não invalida análise
  já persistida)

**Quando flexibilizar:** quando todas operações dependem de uma
operação anterior, ou quando 1 falha realmente invalida o todo (ex:
DELETE em cascata exige transação, não allSettled).

**Anti-pattern:** Promise.all em operações independentes — 1 falha
aborta tudo, mesmo as N-1 que deram certo. Perde dado e custo.

---

## 3. Stack tech recomendada

### 3.1 Linguagem

**TypeScript em strict mode** (`strict: true` no tsconfig). Sem
exceção em projetos novos. Configurações mínimas:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": false,
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "lib": ["ES2022"]
  }
}
```

`noUncheckedIndexedAccess` força tratar `arr[0]` como `T | undefined` —
elimina classe inteira de bug de array vazio.

### 3.2 Runtime

**Node.js LTS** (atualmente 22.x ou 24.x). Sem versões odd não-LTS em
produção.

### 3.3 Package manager

**npm** ou **pnpm**. Yarn evitado (legacy v1) ou Yarn Berry (complexidade
sem ganho claro). Padrão Trilhar atual: npm pra simplicidade.

### 3.4 Banco de dados + Auth + Storage

**Supabase** como default. Por quê:
- Postgres real (não NoSQL pseudo-relacional)
- Auth integrado (email, OAuth, magic link, MFA)
- Storage S3-compatible
- Edge Functions Deno
- RLS nativo em SQL (segurança fica no banco, não no app)
- Pricing previsível, free tier generoso

**Quando flexibilizar:** projetos sem necessidade de auth + storage +
edge (ex: CLI tool puro) podem usar SQLite + Drizzle. Mas raramente
ganha tempo o suficiente pra justificar.

### 3.5 Testing

**vitest**. Por quê:
- Compatível com Jest API (zero migração se vier de Jest)
- Suporta TypeScript nativo via esbuild (sem ts-jest config caos)
- Mocks via `vi.mock` + `vi.hoisted` (pattern poderoso)
- Snapshots inline ou separado
- Watch mode rápido
- Coverage integrado

### 3.6 Frontend

**Next.js 15+ App Router** + **React Server Components** por padrão.

- App Router (não Pages Router) — App Router é o futuro, RSC reduz JS
  no client
- Server Components default; Client Components apenas pra interatividade
- Server Actions pra mutations (evita criar API routes pra cada form)

### 3.7 UI

**shadcn/ui** + **Tailwind CSS**. Por quê:
- shadcn/ui é template-based, não dependency: copia componentes pro
  projeto, customiza livremente
- Acessibilidade by default (Radix UI por baixo)
- Tailwind elimina arquivos CSS gigantes; classes inline são
  rastreáveis via grep
- Sem CSS-in-JS runtime (perf melhor)

### 3.8 Validação runtime

**Zod**. Por quê:
- TypeScript types derivados do schema (`z.infer<typeof Schema>`)
- Validação API boundaries, env vars, user input
- Mensagens de erro customizáveis em PT-BR

### 3.9 Charts/Dashboards

**Recharts**. Cobre BarChart, LineChart, PieChart, ComposedChart com
API simples React. Pra coisas mais sofisticadas (heatmap, treemap),
considera **visx** ou **D3 direto**.

### 3.10 Deploy

**Vercel** pra Next.js (preview deployments por PR é killer feature).
**Supabase Cloud** pro banco + functions. Hosting de domínio próprio
em **Hostinger** ou **Cloudflare Registrar**.

### 3.11 Versions floor

- Node.js: 22.x LTS
- TypeScript: 5.6+
- vitest: 4.x+
- Next.js: 15+
- React: 19+
- Supabase JS: 2.x+

Versão floor sobe conforme projeto exige. Mantém atualizado mas sem
chasing latest — quebras de breaking change saem caro.

---

## 4. Code quality

### 4.1 Sem `any` ou casts injustificados

**Princípio:** TypeScript existe pra prevenir bugs. `as any` joga essa
proteção fora. `as TipoX` sem justificativa é igual.

**Permitido:**
- `as unknown as TipoX` em boundaries com SDK externo (com comment
  explicando o porquê)
- `as const` pra literal types
- Type assertion em test fixtures controlados

**Proibido:**
- `(x as any).foo` pra "fazer compilar"
- `as TipoX` pra forçar shape que TS não consegue inferir (geralmente
  significa que a estrutura está errada)

**[ex: Kenji]** Repository pattern usa `data as unknown as ClientRecord[]`
no return do Supabase, com comment dizendo "supabase-js retorna shape
genérico, cast manual aqui pq não geramos types via supabase gen types
ainda". Aceito porque é boundary explícito + comentado.

**Anti-pattern:** typecheck falha → solução é `as any`. Solução real é
entender por que TS não consegue inferir e corrigir o tipo.

### 4.2 Defensive coding em boundaries

**Princípio:** validação rigorosa em boundaries (entrada do sistema:
user input, API externa, env vars). Confiança em código interno.

**Em boundaries:**
- Null checks explícitos
- Truncamento defensivo (logs, payloads)
- Validation com Zod ou similar
- Try/catch com erros tipados

**Internamente:**
- Confia no contrato dos types
- Não duplica validação que já rolou no boundary
- Se contrato quebrar, é bug — não mascara

**[ex: Kenji]** `buildClientMemoryBlock` (orquestrador IA) trunca notas
em 2000 chars com suffix `[...truncado, original tem N chars]`. O
suffix existe pra IA não inventar conteúdo sobre o que está fora da
janela. Sem o suffix, IA poderia gerar texto baseado em adivinhação.

### 4.3 Idempotência via lookup-then-insert

**Princípio:** operações de seed/import devem ser idempotentes — rodar
2x não duplica. Pattern: lookup primeiro, decide INSERT vs UPDATE.

**Como aplicar:**
```typescript
const existingMap = new Map(existing.map(e => [e.external_id, e]));
for (const item of incoming) {
  const existing = existingMap.get(item.external_id);
  if (existing) {
    await update(existing.id, item);
  } else {
    await insert(item);
  }
}
```

**[ex: Kenji]** `runSeedClients` em `src/integrations/trilhar-finance/seed-clients.ts`
usa exatamente esse pattern. `listAllClients()` → Map por
`trilhar_finance_company_id` → loop por Company → INSERT vs UPDATE.

**Quando flexibilizar:** quando UNIQUE constraint no banco resolve
(UPSERT SQL com ON CONFLICT). Mas lookup-then-insert é mais legível e
permite logging granular do que está sendo INSERTed vs UPDATEd.

### 4.4 Cache via input_signature hash

**Princípio:** cache de resultado caro (chamada IA, computation pesada)
identifica entrada via SHA-256 hash. Hash enumera APENAS dados de
entrada, nunca derivados.

**[ex: Kenji]** `buildSourceDataHash(rawData)` em
`src/pipeline/dossier/source-data-hash.ts` enumera explicitamente cada
campo de cada entidade do `rawData` (Company, Account, Category,
Transaction, MonthlyGoal, EmergencyReserve). Resultado vai pra
`analyses.dossier.meta.sourceDataHash`. Re-run sem mudança de input
retorna cache hit ($0 custo IA).

**Por que enumera explicitamente:** se enumerar com `JSON.stringify`
ingênuo, ordem de keys pode variar entre runs (depende do produtor do
objeto). Hash diferente sem mudança real → cache invalida toda vez.

**Anti-pattern:** cache via timestamp ("expira em 24h"). Não é
idempotente. Re-run dentro da janela retorna stale; re-run fora
retorna recomputado idêntico. Hash é determinístico.

### 4.5 Truncamento defensivo em logs

**Princípio:** logs nunca crescem sem limite. Texto de usuário,
respostas de API, payloads — tudo passa por truncamento com indicação
de tamanho original.

**Pattern:**
```typescript
const MAX_LEN = 2000;
const rendered = text.length > MAX_LEN
  ? `${text.slice(0, MAX_LEN)}\n[...truncado, original tem ${text.length} chars]`
  : text;
```

**[ex: Kenji]** Notas relacionais do cliente truncadas em 2000 chars no
buildClientMemoryBlock. Suffix indica tamanho original pra IA saber
que não viu tudo.

### 4.6 Mock pattern com `_helpers/` compartilhado

**Princípio:** mocks reutilizáveis vivem em `__tests__/_helpers/`.
Quando precisa estender (novo método mockável), faz UMA vez no helper.
N test files herdam.

**[ex: Kenji]** `src/repository/__tests__/_helpers/mock-supabase.ts`
exporta `mockChain` (singleton) que fake `supabase.from().select().eq()...`.
Quando Sub-onda 4.1 precisou `.limit` e `.in`, adicionou no helper (3
linhas + 3 linhas em reset). Sub-onda Seed precisou `.update`, adicionou
mais 3 linhas. Todos os 27+ test files que usam Supabase consomem o
mesmo singleton.

**Anti-pattern:** cada test file inline com seu próprio mock. Quando
schema muda, precisa atualizar 27 lugares.

---

## 5. Testing strategy

### 5.1 Cobertura agressiva

**Princípio:** cada função pública tem 3+ cenários (happy + edge +
erro). Cobertura ~80%+ é base, não meta. Cobertura baixa esconde
classes inteiras de bug.

**[ex: Kenji]** Cresceu organicamente de 0 → 338 tests. Cada Sub-onda
adicionou 5-20 tests novos. Pattern: nenhum PR mergeado sem tests pros
caminhos novos.

**Como aplicar:**
- Função nova → escreve test ANTES ou JUNTO. TDD-light.
- Bug encontrado → escreve test que reproduz, depois fix
- Refactor → tests preservam contrato; se quebram, ou refactor está
  errado ou test estava testando implementação

### 5.2 Pattern de mock

**vitest** suporta 2 patterns principais:

**Pattern A — `vi.hoisted` + `vi.mock` por path:**
```typescript
const { mockFoo } = vi.hoisted(() => ({ mockFoo: vi.fn() }));
vi.mock("../path/to/module.js", () => ({ foo: mockFoo }));
```
Bom quando precisa controlar mocks de várias funções da mesma source.

**Pattern B — `vi.mock` + `vi.mocked`:**
```typescript
vi.mock("../path/to/module.js");
import { foo } from "../path/to/module.js";
const mockedFoo = vi.mocked(foo);
mockedFoo.mockResolvedValue(...);
```
Bom quando módulo é simples e quer reusar import normal.

Convenção: pega o pattern já usado no test file existente. Não mistura
no mesmo file.

### 5.3 Helpers compartilhados

Estrutura recomendada:
```
src/foo/
  bar.ts
  __tests__/
    bar.test.ts
    _helpers/
      mock-baz.ts
      fixtures.ts
```

`_helpers/` (com underscore) deixa claro que não é test em si.

### 5.4 Snapshots pra estruturas complexas

**Princípio:** quando o output é estrutura grande (Dossier de 47
indicadores, response JSON de API), snapshot test captura shape inteiro.
Mudança de shape exige update consciente (`-u`) que humano revisa.

**[ex: Kenji]** `src/pipeline/__tests__/dossier.test.ts.snap` captura
Dossier completo gerado a partir de fixture canônica. Quando RevenueSnapshot
ganhou campo `runRate` na Sub-onda Refinamento, snapshot quebrou —
correto, era pra quebrar. `npm test -- -u` atualizou + commit incluiu o
diff do snapshot pra revisão.

**Quando flexibilizar:** outputs muito voláteis (dados aleatórios,
timestamps) podem usar snapshot com normalização ou checks específicos
em vez do snapshot inteiro.

### 5.5 Tests rodando rápido

**Meta:** suite completa <30s. Mocks evitam IO real. Vitest paraleliza
por arquivo.

**[ex: Kenji]** 338 tests em 4.3s no fim da Fase 4. Cobertura ampla
SEM penalidade de tempo porque tudo é mock — DB real só roda em
calibração manual ($1-2 cada).

**Quando flexibilizar:** alguns projetos exigem tests de integração com
Supabase real (ou container local). Marca como `*.integration.test.ts`
e roda em pipeline separado, não em todo commit.

### 5.6 Não testar implementação

**Princípio:** test bom valida contrato (entrada → saída). Refactor
interno não quebra tests bons.

**Mau test:** verifica que função X chamou função Y com args específicos
(testa implementação). Se Y é renomeada ou inlined, test quebra sem
nenhuma mudança comportamental.

**Bom test:** chama função X com input + asserta output esperado.
Implementação interna pode mudar — test continua passando.

**Exceção:** wiring tests onde o ponto É verificar que callback foi
invocado. Aí mock do callback é apropriado. Mas isso deve ser minoria.

---

## 6. Database & Migrations

### 6.1 Migrations versionadas com timestamp

**Princípio:** toda mudança de schema vai como migration SQL versionada.
Nome: `YYYYMMDDHHMMSS_descricao_em_snake_case.sql`. Timestamp garante
ordem cronológica e evita conflito de naming.

**[ex: Kenji]** `supabase/migrations/20260423090200_memory_layer1_events.sql`,
`20260512100000_transition_recommendation_status_function.sql`. 14
migrations no total no fim da Fase 4.

**Como aplicar:**
- Cada mudança = 1 migration
- Migration é IMUTÁVEL após push pro main — mudança vira nova migration
- Testa em ambiente staging antes de aplicar em prod
- Backup do banco antes de migration que dropa coluna ou table

### 6.2 Convention de naming snake_case + camelCase

**Princípio:** SQL e Postgres usam snake_case. TypeScript usa camelCase.
Mantém consistência: nomes de coluna nunca em camelCase no SQL, tipos
TS nunca em snake_case.

**Onde converter:**
- Repository layer faz mapping na fronteira: `legal_name` (SQL) →
  `legalName` (TS) ao construir input pra INSERT
- Mas types `ClientRecord` mantém snake_case quando representa row
  literal do banco — economia de mapping pra leitura

**[ex: Kenji]** `ClientRecord.relationship_notes` (snake_case porque
espelha row do banco). `InsertClientInput.relationshipNotes` seria
camelCase se existisse, mas Kenji optou por manter `relationship_notes`
no input também — consistência com ClientRecord (decisão validada na
Sub-onda Refinamento).

**Quando flexibilizar:** se projeto tem ORM que faz mapping automático
(Drizzle, Prisma), aceita o convention da ORM. Não força mistura.

### 6.3 RLS habilitado em TODAS as tabelas SEMPRE

**Princípio:** Row Level Security em toda tabela, mesmo quando
aplicação roda como service_role (bypass). RLS é rede de segurança
extra — se key vazar OU se app code tiver bug expondo dado, RLS
mitiga.

**Pattern mínimo:**
```sql
alter table public.foo enable row level security;
create policy "service_role_full_access" on public.foo
  for all to service_role using (true) with check (true);
```

**[ex: Kenji]** 14 tabelas com RLS habilitado + policy
`service_role_full_access`. Bypass funcional, segurança preservada.
Quando multi-user entrar, policies por user_id substituem service_role
sem migration nova.

### 6.4 Audit trail

**Princípio:** toda tabela tem `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
e `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`. Trigger automática
atualiza updated_at em UPDATE.

**Pattern de trigger:**
```sql
create or replace function public.set_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger trg_foo_set_updated_at
  before update on public.foo
  for each row execute function public.set_updated_at();
```

Reutilizar `set_updated_at()` em todas as tabelas. Definida UMA vez.

### 6.5 Soft delete onde aplicável

**Princípio:** quando histórico importa, evita DELETE. Usa
`deleted_at TIMESTAMPTZ NULL` + queries filtram `WHERE deleted_at IS NULL`.

**Quando aplicar:**
- Recursos que usuário pode "deletar" mas histórico é valioso (ex:
  análises, transações, recomendações)
- Auditoria regulatória

**Quando NÃO aplicar:**
- Dados temporários (cache, sessões)
- Quando GDPR/LGPD exige hard delete

### 6.6 UNIQUE constraints onde precisa

**Princípio:** constraint no banco é defesa em profundidade. App code
pode ter bug que tente inserir duplicata; UNIQUE no banco bloqueia.

**[ex: Kenji]** `clients.tax_id UNIQUE`,
`clients.trilhar_finance_company_id UNIQUE`. Garante que cada CNPJ é
1 cliente, cada Company do Trilhar Finance é 1 cliente.

### 6.7 Foreign keys com ON DELETE pensado

**Princípio:** toda FK declara explicitamente comportamento em DELETE
do parent. Default (sem cláusula) é RESTRICT — geralmente é o que
queremos, mas vale explicitar.

Opções:
- `ON DELETE CASCADE` — child também deletado (raramente o que se
  quer)
- `ON DELETE SET NULL` — child mantém, FK vira null
- `ON DELETE RESTRICT` — bloqueia delete do parent enquanto houver
  child (default)

**[ex: Kenji]** `recommendations.client_id REFERENCES clients(id) ON DELETE CASCADE`
porque rec sem cliente não faz sentido. Mas `recommendation_status_history.recommendation_id REFERENCES recommendations(id) ON DELETE CASCADE` também — quando rec é deletada, histórico vai junto.

### 6.8 Comments em SQL

**Princípio:** documenta colunas + tabelas em produção via `comment on`.
Devs futuros (e queries ad-hoc no Studio) leem documentação direto.

```sql
comment on column public.clients.relationship_notes is
  'Histórico relacional em texto livre. Editado via Studio em V1.0.
   Lido pelo Kenji no orquestrador como contexto pra evitar repetir
   recomendações.';
```

**Anti-pattern:** schema sem comments, devs precisam grep no código pra
entender o que cada coluna significa.

---

## 7. Security baseline ⚠️ CRÍTICO

### 7.1 `.env` NUNCA commitado

**Princípio:** secrets em `.env`, sempre. `.env` no `.gitignore`,
sempre. Verificação: `git log --all -- ".env"` deve retornar VAZIO.

**Verificação periódica:**
```bash
git log --all -- ".env" ".env.local" ".env.production"
# Deve retornar vazio. Se retornar commit, .env vazou — TODOS os
# secrets daquele commit precisam ser rotacionados.
```

**[ex: Kenji]** `.env` nunca foi commitado. Audit security V1.0
confirmou. Rotação preventiva da `ANTHROPIC_API_KEY` quando ela apareceu
em log de chat (não no git, mas sessão exposta).

### 7.2 2FA mandatório

**Princípio:** 2FA em TODAS as contas com acesso a dados ou produção:
- GitHub (org + pessoal)
- Supabase (conta + projeto)
- Vercel
- Provider IA (Anthropic, OpenAI)
- Hosting (Hostinger, Cloudflare)
- Email (Google, ProtonMail)
- Domain registrar

**Recovery codes:** 1Password / Bitwarden / cofre físico. NUNCA arquivo
no Mac.

**Por que crítico:** 2FA elimina ~99% dos vetores de credential
stuffing. Sem 2FA, é questão de tempo até ser comprometido.

### 7.3 Credenciais de leitura ≠ escrita

**Princípio:** se aplicação só lê de um serviço externo, key tem
permissão SÓ DE LEITURA. Service_role com escrita só onde precisa.

**[ex: Kenji]** Kenji autentica no Trilhar Finance com keys read-only
(Edge Function `read-data` é endpoint único, sem write). Service_role
do Kenji é usado APENAS no banco do próprio Kenji, nunca no banco do
Trilhar Finance.

### 7.4 Edge Functions com auth dupla

**Princípio:** Edge Function pública tem 2 camadas de auth:
- API key custom (header `x-api-key`) — controlada por você, rotacionável
- Anon key Supabase (header `apikey`) — exigida pelo gateway

Cliente deve apresentar ambas. Falha em uma → 401.

**[ex: Kenji]** `read-data` do Trilhar Finance exige `x-api-key:
kenji_live_*` + `apikey: <anon>`. Header `kenji_live_*` é rotacionável
pelo Trilhar Finance se Kenji for comprometido, sem afetar outros
consumers.

### 7.5 npm audit fix preventivo

**Princípio:** roda `npm audit fix` antes de cada release. Vulnerabilidades
em transitive deps acumulam silenciosamente.

**[ex: Kenji]** Fix de `basic-ftp` (HIGH, DoS) e `ip-address` (MODERATE,
XSS) descobertos no security audit V1.0. Ambos transitive deps,
provavelmente via Puppeteer (a entrar na Fase 5). `npm audit fix`
resolveu sem breaking change.

### 7.6 `.gitignore` cobre IDE + OS

```
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store
Thumbs.db
```

Settings de IDE com paths absolutos, swap files de Vim/Emacs, lixo do
SO — nada disso pertence ao repo.

### 7.7 Logs SEM PII / financial / secrets

**Princípio:** se vaza em log, vaza em todo lugar (terminal compartilhado,
ferramenta de log, screenshot). Logs nunca contêm:
- CNPJ, CPF, RG, passport completos
- Saldos, valores, números financeiros sensíveis
- Senhas, API keys, tokens
- Email + nome juntos (PII)

**Pattern de mascaramento:**
- CNPJ → status `✓ válido` ou `✗ inválido` (NÃO os dígitos)
- Valor financeiro → log de presença / ausência, não o número
- API key → primeiros 10 chars + `...`

**[ex: Kenji]** `maskTaxIdStatus` em `seed-clients.ts` retorna `"✓ válido" | "✗ inválido"` em vez de mostrar dígitos do CNPJ. Decisão D-Seed-7b
do Kenji.

### 7.8 dotenv override pra preservar `.env`

**Princípio:** scripts CLI que dependem de `.env` usam:

```typescript
import { config } from "dotenv";
config({ override: true });
```

NÃO usa `import "dotenv/config"` (default sem override).

**Por que:** sem `override: true`, dotenv NÃO sobrescreve env vars já
definidas no shell. Em contextos onde shell injeta var (ex: rodando
dentro de Claude Code que injeta `ANTHROPIC_API_KEY` proxy), script
herda credencial errada e quebra silenciosamente OU pior, usa credencial
de outro contexto sem aviso.

**[ex: Kenji]** Descoberto durante calibração 4.3+4.4 (Resolucar). 4
specialists falharam com `ANTHROPIC_API_KEY não está definida` mesmo
com key correta no `.env`. Diagnóstico revelou Claude Code injetando
proxy. Fix em 6 arquivos: 5 scripts CLI + `src/main.ts`. Documentado
como decisão permanente.

### 7.9 NÃO mostrar valores reais de credenciais em STDOUT

**Princípio:** debugging de env vars usa contagem ou prefixo, NÃO o
valor. STDOUT vai pra log de chat, ferramenta CI, screenshot — não
controlamos onde acaba.

**Mau:**
```bash
grep "ANTHROPIC_API_KEY" .env
# Mostra valor completo no STDOUT
```

**Bom:**
```bash
grep -c "^ANTHROPIC_API_KEY=" .env
# Conta linhas (1 = presente, 0 = ausente). Sem valor.

grep "^ANTHROPIC_API_KEY=" .env | awk -F'=' '{print "len="length($2)", prefix="substr($2,1,10)"..."}'
# Mostra estrutura sem expor valor.
```

**[ex: Kenji]** Durante diagnóstico de auth, primeiro grep mostrou key
completa no log. Ação imediata: rotação da key + correção de pattern
pra debug futuro.

### 7.10 Backup do banco

**Princípio:** Supabase Pro tem backup diário automático. Free tier
não tem — usa export manual via CSV antes de mudanças críticas
(migration que dropa coluna, DELETE em massa).

**Verificação:**
- Dashboard Supabase → Database → Backups → confirma frequência e
  retention
- Antes de mudança crítica: trigger backup manual ("Create backup"
  button) ou export CSV

### 7.11 Rotação periódica de keys

**Princípio:** keys rotacionadas trimestralmente em produção, OU
imediatamente após qualquer exposição (logs, screenshots, terminal
compartilhado).

**Como aplicar:**
- Provider issue nova key
- Atualiza `.env` local + secrets em CI/Vercel
- Revoga key antiga
- Verifica que app continua funcionando

---

## 8. Auth & multi-user

(Esta seção expande além do Kenji V1.0, que é serviço interno
single-tenant. Aplica-se a projetos multi-user futuros.)

### 8.1 Supabase Auth como base

**Princípio:** Supabase Auth resolve signup, login, password reset,
OAuth, magic link, MFA. Não reinventa.

**Métodos a habilitar:**
- Email + password (sempre)
- Google OAuth (se audiência espera)
- Magic link (UX moderna pra B2C)
- TOTP MFA (opcional pra usuário, recomendado pra admin)

### 8.2 JWT propagado pra Edge Functions

**Princípio:** Supabase Auth gera JWT no login. JWT vai automaticamente
em headers de qualquer request a Postgres ou Edge Function. Postgres lê
via `auth.uid()` em RLS policies.

**Pattern:**
```sql
create policy "users_can_read_own_data"
on public.transactions
for select
using (auth.uid() = user_id);
```

### 8.3 Profile table linkada a auth.users

**Princípio:** `auth.users` é gerenciado pelo Supabase (não tocar). Pra
estender perfil (display name, avatar, role), cria `public.profiles`
com FK pra `auth.users.id`.

```sql
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  display_name text not null,
  avatar_url text,
  role public.app_role not null default 'staff',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

### 8.4 Roles via enum

**Princípio:** roles são fixos por design. Postgres enum é o tipo
correto.

```sql
create type public.app_role as enum ('owner', 'manager', 'staff', 'viewer');
```

Adicionar role nova exige migration (`alter type ... add value`). Isso
é feature, não bug — força revisão consciente.

### 8.5 Policies RLS por role

**Pattern:**
```sql
create policy "managers_can_manage_inventory"
on public.inventory
for all
using (
  exists (
    select 1 from public.profiles
    where id = auth.uid()
    and role in ('owner', 'manager')
  )
);
```

Policies em Postgres: zero código aplicação envolvido. Segurança fica
no banco, sem possibilidade de bypass por bug em controller.

### 8.6 Audit log de ações sensíveis

**Princípio:** ações reversíveis ou contestáveis (alteração de preço,
delete de cliente, mudança de role) registram em tabela de audit.

```sql
create table public.audit_log (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  action text not null,
  resource_type text not null,
  resource_id uuid,
  changes jsonb,
  created_at timestamptz not null default now()
);
```

Insert via trigger automático ou via função do app que precisa registrar.

### 8.7 Password reset flow

Built-in do Supabase Auth. Customização do email template via
Dashboard. Confirmar URL de redirect aponta pra `/auth/reset` da app.

---

## 9. Frontend patterns

### 9.1 Next.js 15+ App Router

**Princípio:** App Router (não Pages Router). Server Components por
padrão, Client Components quando interatividade exige.

**Estrutura:**
```
app/
  layout.tsx                  # Root layout
  page.tsx                    # /
  (marketing)/
    layout.tsx                # Layout específico do grupo
    sobre/page.tsx            # /sobre
  (app)/
    layout.tsx                # Layout app autenticado
    dashboard/page.tsx        # /dashboard
    clientes/[id]/page.tsx    # /clientes/[id]
  api/
    webhook/route.ts          # POST /api/webhook
```

Route groups `(name)/` agrupam rotas sem afetar URL.

### 9.2 Server Components por padrão

**Princípio:** `"use client"` só quando necessário (estado local,
event handler, browser API). Server Components fazem await direto sem
useEffect.

**Bom (Server Component):**
```tsx
export default async function Page() {
  const data = await fetchData();
  return <div>{data.name}</div>;
}
```

**Quando precisa Client Component:**
```tsx
"use client";
import { useState } from "react";

export function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

### 9.3 Forms com React Hook Form + Zod

**Princípio:** validation tipada client + server. Schema Zod único pra
ambos.

```typescript
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});
type FormData = z.infer<typeof schema>;
```

Server Action ou API route reusa o mesmo schema pra validar antes de
persistir.

### 9.4 Dashboards com Recharts

**Princípio:** charts financeiros usam BarChart, LineChart, ComposedChart
(line+bar misto), PieChart pra share. Tooltips formatadas em moeda BR.

**Helper de formatação:**
```typescript
const formatCurrency = (v: number) =>
  new Intl.NumberFormat("pt-BR", {
    style: "currency",
    currency: "BRL",
  }).format(v);
```

**Truncamento de labels longos:** charts com muitas categorias precisam
truncamento + tooltip mostrando label completo.

### 9.5 Tailwind + shadcn/ui

**Princípio:** shadcn/ui é template-based — copia componente pro
projeto via `npx shadcn add button`. Customização livre.

**Quando usar shadcn base:** Button, Input, Dialog, Sheet, Select, Tabs,
Card, Badge, Separator, Tooltip, Avatar, DropdownMenu — quase tudo.

**Quando estender:** componente de domínio (FinancialMetricCard,
ClientHeader). Compõe sobre primitivos da shadcn.

**Quando NÃO usar:** UI muito específica (3D, canvas, streaming chat).
Aí Tailwind direto + lib específica.

### 9.6 Loading states

**Princípio:** `loading.tsx` por rota mostra skeleton enquanto Server
Component está rendering. Sem flash de tela vazia.

```
app/dashboard/
  page.tsx        # data fetching
  loading.tsx     # skeleton enquanto await
```

Suspense boundaries dentro de page se subcomponentes têm await
independentes.

### 9.7 Error boundaries

**Princípio:** `error.tsx` por rota captura unhandled errors, mostra
fallback graceful + botão "tentar novamente".

```tsx
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <h2>Algo deu errado</h2>
      <button onClick={reset}>Tentar novamente</button>
    </div>
  );
}
```

Sentry ou similar pra reportar errors em produção.

### 9.8 Otimização de imagens

**Princípio:** `<Image />` do Next.js (não `<img>`). Lazy loading +
responsive + WebP/AVIF automático.

```tsx
import Image from "next/image";
<Image src="/logo.png" alt="Logo" width={200} height={50} priority />
```

`priority` em above-the-fold, demais lazy by default.

### 9.9 SEO básico

```typescript
export const metadata: Metadata = {
  title: "Trilhar - Gestão Financeira",
  description: "Consultoria de gestão financeira pra PMEs",
  openGraph: { ... },
};
```

Tags importantes: title, description, og:image, og:title, og:description.
Sitemap.xml + robots.txt em `app/sitemap.ts` e `app/robots.ts`.

### 9.10 Deploy Vercel

**Princípio:** preview deployments por PR é killer feature. Cada PR
gera URL própria com env vars de preview.

**Configuração:**
- Production branch: `main`
- Env vars: dashboard Vercel (Production / Preview / Development
  separados)
- Branch protection no GitHub: require checks passing antes de merge

---

## 10. Integrações externas

### 10.1 Edge Function pattern

**Princípio:** integrações que rodam perto do banco (consulta a outra
DB Supabase, webhook handler) ficam em Edge Functions. Latência menor
que API tradicional Vercel.

**Estrutura:**
```
supabase/functions/
  meu-handler/
    index.ts        # entry point
    deno.json       # config
```

Deno runtime, syntax similar a Node mas com algumas diferenças (sem
process.env — usa `Deno.env.get()`).

### 10.2 Auth dupla

Detalhado em §7.4. Custom API key + anon key padrão Supabase.

### 10.3 Idempotência via input_signature hash

**Princípio:** request com mesmo input retorna mesmo output, sem
side effect duplicado. Pattern de cache.

**[ex: Kenji]** `analyses.cached_hash` armazena SHA-256 do `dossier`.
Re-run com mesmo cliente + mesma semana retorna cache se hash bate.
$0 custo IA.

### 10.4 Read-only por design

**Princípio:** se integração só lê, expõe APENAS read. Não cria
endpoint write "pra futuro". YAGNI aplica forte aqui — write endpoint
é vetor de ataque permanente.

**[ex: Kenji]** `read-data` do Trilhar Finance é endpoint único,
read-only por design. Auditado em security audit V1.0 (Check 1
verificou ausência de métodos `.insert`, `.update`, `.delete`,
`POST/PUT/PATCH/DELETE` no client TS).

### 10.5 Rate limiting + retry com backoff

**Princípio:** assume rate limit existe. Cliente HTTP retenta com
backoff exponencial em 429 e erros de rede.

**Pattern:**
```typescript
const MAX_RETRIES = 3;
const BACKOFF_MS = [1000, 2000, 4000];

for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
  try {
    return await request();
  } catch (err) {
    if (!shouldRetry(err) || attempt === MAX_RETRIES) throw err;
    await sleep(BACKOFF_MS[attempt]);
  }
}
```

`shouldRetry` distingue erros transientes (429, 503) de permanentes
(401, 400) — não retenta o que não vai mudar.

### 10.6 Timeout via AbortController

**Princípio:** request HTTP sem timeout pode travar processo. Sempre
declara timeout máximo.

```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30_000);
try {
  const res = await fetch(url, { signal: controller.signal });
} finally {
  clearTimeout(timeout);
}
```

### 10.7 Webhook delivery

**Princípio:** webhook confiável tem retry + dead letter queue.
Receiver acknowledges com 2xx; sender retenta em 4xx/5xx.

**Pattern Supabase:** trigger Postgres → fila (pg_net) → Edge Function
processa → ack ou re-enqueue.

---

## 11. Process patterns

### 11.1 FASE A audit antes de mudar

Detalhado em §1.3. Sem audit, propostas batem em surpresas.

### 11.2 FASE B+C plano antes de codar

**Princípio:** apresenta plano + decisões + riscos ANTES de aplicar
DIFFs. Operador aprova ou ajusta.

**Estrutura típica do plano:**
- Lista de arquivos a tocar (path + tipo de mudança)
- DIFFs propostos (mostrar code real, não pseudo-código)
- Decisões pendentes (com inclinação + alternativas)
- Riscos identificados (com mitigação se houver)
- Sub-colas previstas (FASE D em N partes)

### 11.3 FASE D em sub-colas isoladas

Detalhado em §1.2. Uma mudança coerente por sub-cola.

### 11.4 `git commit -F /tmp/msg.txt`

**Princípio:** commit message multilínea via heredoc no shell sofre com
escape de `$`. Solução: salva mensagem em arquivo, usa `-F`.

**Mau:**
```bash
git commit -m "$(cat <<EOF
chore: foo

Custo: \$1.50  # vira \$1.50 literal em algumas configs
EOF
)"
```

**Bom:**
```bash
cat > /tmp/commit-msg.txt <<'COMMIT_MSG_EOF'
chore: foo

Custo: $1.50
COMMIT_MSG_EOF
git commit -F /tmp/commit-msg.txt
rm /tmp/commit-msg.txt
```

`<<'COMMIT_MSG_EOF'` (com aspas) preserva literais. `git commit -F`
não invoca shell expansion.

**[ex: Kenji]** Bug encontrado no commit da Sub-onda 4.1. Pattern de
mitigação aplicado em todos os commits subsequentes (4.2, 4.3, 4.4,
4.5, Seed, Refinamento) — zero recorrência.

### 11.5 Commit messages estruturados

**Princípio:** commit message tem header curto (<70 chars) + body com
seções. Não "fix typo" ou "WIP".

**Estrutura típica:**
```
{tipo}: {titulo curto}

{Parágrafo de contexto: o que motivou a mudança.}

## Adições
- Item 1
- Item 2

## Decisões fechadas (D-{N})
- D-Foo-1: ...
- D-Foo-2: ...

## Pegada técnica
- Tests: X → Y verde (+Z novos)
- Typecheck: 0 erros
- LOC: ~A inserções, ~B deleções
- Custo IA: $X.XX

## Próximo passo
{Sub-onda seguinte ou ADR consolidador}
```

`tipo` segue Conventional Commits: `feat`, `fix`, `docs`, `chore`,
`refactor`, `style`, `test`, `perf`.

### 11.6 ADRs consolidadores

Detalhado em §1.6. Doc `docs/decisoes/NNNN-nome.md` por fase.

### 11.7 Numeração de decisões

**Convenções:**
- `D-IA-{N}` pra decisões focadas em integração IA
- `D-Fase{X}-{N}` pra decisões dentro de fase específica
- `D-{Tema}-{N}` pra sub-ondas focadas (D-Refino-1, D-Seed-3)

Numeração sequencial dentro do escopo. ADR consolidador lista todas em
ordem.

### 11.8 Convenção de commits (Conventional Commits)

```
feat:     nova feature
fix:      bug fix
docs:     documentação only
chore:    config, deps, tooling
refactor: mudança sem comportamento novo
style:    formatação, sem mudança de código
test:     adiciona/atualiza tests
perf:     otimização de performance
```

### 11.9 Pre-commit hooks (opcional)

**Husky + lint-staged** pra rodar typecheck + tests rápidos antes de
commit. Reduz commits quebrados.

```json
{
  "lint-staged": {
    "*.ts": ["tsc --noEmit", "vitest related --run"]
  }
}
```

**Quando flexibilizar:** projetos solo onde dev é disciplinado podem
pular. Time grande ou múltiplos contributors: hooks valem.

---

## 12. Anti-patterns

Lista do que NÃO fazer, com origem real do Kenji onde aplicável.

### 12.1 Credenciais hardcoded em código fonte

❌ `const apiKey = "sk-..."` em `src/`. Sempre via `process.env`.
Audit security: `grep -rE "sk-ant-|sk-proj-|eyJ[A-Za-z0-9_-]{20,}" src/`
deve retornar zero.

### 12.2 `console.log` com PII / financial / secrets

❌ `console.log("Cliente", client)` sem sanitização. Logs vão pra
ferramenta agregadora — vaza em todo lugar.

**[ex: Kenji]** Pattern correto: `maskTaxIdStatus` em vez de logar
CNPJ; LogPayload com metadata operacional (durationMs, attempt) sem
client data financial.

### 12.3 `as any` casts pra "fazer compilar"

❌ TypeScript existe pra prevenir bugs. `as any` joga essa proteção
fora. Resolve a raiz do problema (entender por que TS não infere) em
vez de mascarar.

### 12.4 Mock returns sem helper compartilhado

❌ Cada test file inline com seu próprio mock duplica código. Quando
schema muda, atualiza N lugares.

**[ex: Kenji]** `mock-supabase.ts` singleton consumido por 27+ tests.
Estender uma vez, beneficia todos.

### 12.5 Skip de FASE A audit "porque parece simples"

❌ Mudança "simples" que vira refactor de 3 dias porque dependência
não-óbvia foi descoberta no meio.

**[ex: Kenji]** Sub-onda 4.1 começou com "vou criar tabela
client_decisions" — FASE A audit revelou que `recommendations` JÁ
EXISTIA. Reescopo eliminou 40% do trabalho.

### 12.6 10 mudanças num turno só sem sub-cola

❌ STDOUT trunca, validação fica difícil, race conditions aparecem.
Micro-turnos eliminam.

**[ex: Kenji]** Fase 3 teve 7 falhas de STDOUT antes de adotar
micro-turnos. D-Fase4-9 padronizou — zero recorrência em 16+ sub-colas.

### 12.7 ADRs ausentes no fim de fase

❌ Conhecimento volátil no chat. Próximo dev (ou tu mesmo daqui 3
meses) não sabe por que decisão foi tomada.

### 12.8 Commit message vazia ou "fix"

❌ Perde contexto. `git log` vira ruído. Commits de 1 linha são OK pra
typo; mudanças significativas merecem corpo.

### 12.9 `.env` commitado em branch privada "só pra teste"

❌ Branch privada eventualmente é mergeada ou compartilhada. Secret
fica no histórico permanente do repo (mesmo após `git rm`).

### 12.10 Mock IA com saídas inventadas

❌ Mock de chamada IA com resposta sintética que NÃO bate com o que IA
real produz. Test passa, prod quebra.

**Como evitar:** captura saída real uma vez (seed da fixture), usa em
mock. Atualiza fixture quando IA muda.

### 12.11 Cache invalidation por timestamp

❌ "Cache expira em 24h" não é idempotente. Re-run dentro da janela
retorna stale; re-run fora retorna recomputado idêntico (custo IA
desnecessário).

**Pattern correto:** hash de input. Re-run com mesmo input = cache hit
sempre.

### 12.12 Migration que dropa coluna sem backup

❌ Irreversível. Mesmo migration "pequena" pode esconder dado crítico.
Backup antes, sempre.

### 12.13 Cron sem idempotência

❌ Job que roda 2x por dia sem ser idempotente cria duplicatas. Sempre
estrutura cron pra ser safe re-rodar.

**Pattern:** lookup-then-insert OU UPSERT com ON CONFLICT.

### 12.14 Logging valor de credencial pra "debug"

❌ Visto durante calibração 4.3 do Kenji: `grep "ANTHROPIC_API_KEY" .env`
mostrou valor completo no STDOUT. Ação: rotação imediata.

**Pattern correto:** `grep -c "VAR=" .env` (só conta) OU
`awk '{print "len="length($2)", prefix="substr($2,1,10)"..."}'` (estrutura
sem dígitos).

### 12.15 Skip de test "porque é trivial"

❌ "Esse helper é simples demais pra testar." Helpers simples são os
que mais causam bug silencioso porque ninguém revisa.

**[ex: Kenji]** `normalizeTaxId` e `maskTaxIdStatus` (helpers de 5
linhas) ganharam 6 cenários de teste. Custo: 30min. Benefício: garantia
de que comportamento default não muda em refactor.

---

## 13. Como começar um projeto do zero

Checklist sequencial. Ordem importa — passos posteriores dependem dos
anteriores.

### Setup inicial

1. ☐ Cria repo GitHub privado em `trilhargestao-oss/<projeto>`
2. ☐ Clone localmente em `~/Documents/Projetos/<projeto>/`
3. ☐ Configura git user.email e user.name (herda global ou specifica
       no repo)

### Configuração de TypeScript + Node

4. ☐ `npm init -y` (ou `pnpm init`)
5. ☐ `npm install -D typescript @types/node tsx vitest @vitest/coverage-v8`
6. ☐ Cria `tsconfig.json` com `strict: true`, `noUncheckedIndexedAccess: true`,
       `module: NodeNext`, `target: ES2022`
7. ☐ Cria `package.json scripts`: `typecheck`, `test`, `dev`, `build`
8. ☐ Cria `.gitignore` cobrindo `node_modules`, `dist`, `.env`, `.env.*`,
       `!.env.example`, `.vscode/`, `.idea/`, `*.swp`, `.DS_Store`,
       `coverage/`, `*.log`

### Env + secrets

9. ☐ Cria `.env.example` com TODAS env vars que projeto usa
        (placeholder, NUNCA valor real)
10. ☐ Cria `.env` localmente com valores reais
11. ☐ Verifica: `git status` deve mostrar `.env.example` mas NÃO `.env`
12. ☐ Verifica: `git log --all -- ".env"` deve retornar vazio

### Supabase + Auth

13. ☐ Cria projeto Supabase em supabase.com
14. ☐ Habilita 2FA na conta Supabase
15. ☐ Anota `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`
        em `.env`
16. ☐ Cria `src/repository/supabase-client.ts` com singleton lazy
17. ☐ Habilita Auth providers (email + password mínimo)

### Migrations setup

18. ☐ Instala `supabase` CLI: `npm install -D supabase`
19. ☐ `npx supabase init`
20. ☐ Cria primeira migration: `npx supabase migration new initial_schema`
21. ☐ Aplica via `npx supabase db push` (modo dev) ou via Studio (prod)

### Github org + protections

22. ☐ Habilita 2FA no GitHub pessoal
23. ☐ Habilita 2FA enforcement na org (Settings → Authentication
        security)
24. ☐ Branch protection no `main`:
        - ☐ Require pull request before merging
        - ☐ Require status checks (CI)
        - ☐ Require linear history (squash merges)
        - ☐ Block force pushes
25. ☐ Adiciona Cauê (e outros owners) com role `Owner`

### CI

26. ☐ Cria `.github/workflows/ci.yml` rodando typecheck + tests em
        push/PR
27. ☐ Confirma que CI passa em PR de teste

### Deploy (se Next.js)

28. ☐ Cria projeto Vercel apontando pro repo
29. ☐ Configura env vars no Vercel (Production / Preview / Development)
30. ☐ Push pra branch nova → confirma preview deployment
31. ☐ Merge → confirma production deployment

### Documentação inicial

32. ☐ `README.md` curto: o que é, quem usa, como rodar
33. ☐ `CLAUDE.md` (se vai usar Claude Code) com contexto do projeto +
        convenções específicas
34. ☐ `docs/decisoes/0001-stack-inicial.md` ADR documentando escolhas
        de stack

### Teste end-to-end

35. ☐ Roda `npm test` localmente — deve passar
36. ☐ Roda `npm run typecheck` — 0 erros
37. ☐ Cria PR de teste com mudança trivial (typo em README)
38. ☐ CI passa, merge funciona, deploy roda
39. ☐ Confirma 2FA funciona em logout/login
40. ☐ Confirma `.env` não está commitado: `git ls-files | grep .env`
        deve retornar APENAS `.env.example`

### Scripts úteis pra criar conforme projeto cresce

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "lint": "eslint .",
    "format": "prettier --write ."
  }
}
```

### Marcos de saúde do projeto

- ✅ 0 vulnerabilidades em `npm audit`
- ✅ 0 erros em `npm run typecheck`
- ✅ Tests passando
- ✅ Cobertura ≥80%
- ✅ Sem credenciais hardcoded (`grep -rE "sk-ant-|sk-proj-|eyJ" src/`
       vazio)
- ✅ ADRs documentando decisões > 0

---

## Apêndice — Referências cruzadas

- **Kenji repo:** `trilhargestao-oss/kenji` — exemplo concreto de tudo
  acima aplicado em produção
- **ADRs Kenji relevantes:**
  - 0010 — Fase 3 foundation + cache (D-IA-13)
  - 0011 — Schema mismatch incidente (origina regra preventiva ADR)
  - 0012 — Sub-onda 3.2 (4 specialists IA)
  - 0013 — Fase 3 consolidador
  - 0014 — Fase 4 consolidador (memória multi-tier)

## Histórico de revisões

| Data | Autor | Mudança |
|---|---|---|
| 2026-05-13 | Cauê + Claude | Versão inicial 1.0 — destilação de lições do Kenji (jan-mai 2026) |

# CLAUDE.md

> **Como usar este arquivo.** Este `CLAUDE.md` fica na raiz do repositório e é
> lido pelo Claude Code (e por você) a cada interação. Contexto é caro: mantenha-o
> **enxuto e específico do seu projeto**. Este documento é um _template abrangente_
> — cobre Django, Flask e FastAPI de uma vez para servir de referência. **Depois
> de copiá-lo, apague a seção de framework que você não usa** e preencha os blocos
> marcados com `<...>`. Um `CLAUDE.md` genérico e gigante rende respostas piores
> que um curto e preciso.

---

## 1. Contexto do projeto

- **Nome:** `<nome do projeto>`
- **O que faz:** `<uma ou duas frases: domínio, usuários, objetivo>`
- **Estágio:** `<protótipo | produção | legado em migração>`
- **Domínios centrais:** `<ex.: cobrança, catálogo, integrações fiscais>`
- **Restrições de negócio importantes:** `<ex.: multi-tenant, LGPD, SLA de X ms>`

> Preencha isto com honestidade. É o que evita que decisões técnicas sejam
> tomadas no vácuo.

---

## 2. Stack e versões

- **Python:** 3.12+ (use recursos modernos: `match`, generics `list[int]`, `type` aliases, `Self`)
- **Gerenciador de pacotes/ambiente:** `uv` (padrão atual; substitui pip/poetry/pipenv)
- **Lint + format:** `ruff` (substitui black, isort, flake8, pylint)
- **Type checker:** `mypy` (ou `pyright`/`basedpyright`) — modo estrito quando possível
- **Testes:** `pytest` + `pytest-cov`
- **Validação/serialização:** `pydantic` v2
- **Banco/ORM:** `<Django ORM | SQLAlchemy 2.0 | Tortoise>`
- **Task queue:** `<Celery | Dramatiq | arq | RQ>` (se houver)
- **Observabilidade:** `<structlog | OpenTelemetry | Sentry>`

**Regra de ouro:** antes de sugerir uma dependência nova, verifique se algo
equivalente já está no `pyproject.toml`. Não adicione biblioteca para o que a
stdlib resolve bem.

---

## 3. Comandos essenciais

> **Sempre use estes comandos** em vez de invocar ferramentas diretamente. Se um
> comando não existe aqui e você precisou dele, proponha adicioná-lo.

```bash
# Setup
uv sync                          # instala deps (respeita uv.lock)
uv run <cmd>                     # roda algo no ambiente do projeto

# Qualidade — rode ANTES de considerar qualquer tarefa concluída
uv run ruff format .             # formata
uv run ruff check --fix .        # lint + autofix
uv run mypy .                    # type check
uv run pytest                    # testes
uv run pytest -x -q              # para no 1º erro, saída curta (loop de dev)
uv run pytest --cov --cov-report=term-missing

# Um alvo só
uv run pytest tests/unit/test_pagamentos.py::test_estorno -q

# App
<ex.: uv run uvicorn app.main:app --reload>
<ex.: uv run python manage.py runserver>

# Banco
<ex.: uv run alembic upgrade head        (SQLAlchemy)>
<ex.: uv run python manage.py migrate    (Django)>
```

**Definition of Done para qualquer mudança de código:** `ruff format` + `ruff
check` + `mypy` + `pytest` passando. Não entregue código que quebra qualquer um.

---

## 4. Estrutura do projeto

> Ajuste ao layout real. O importante é a **direção das dependências**: camadas de
> fora dependem das de dentro, nunca o contrário.

```
src/<pacote>/
├── domain/          # entidades, regras de negócio puras (sem framework, sem I/O)
├── application/     # casos de uso / services; orquestra domínio + portas
├── infrastructure/  # implementações concretas: repos, clients HTTP, ORM, cache
├── interfaces/      # entrada: rotas HTTP, CLI, handlers de fila, schemas (DTOs)
└── config/          # settings, injeção de dependências, wiring
tests/
├── unit/            # rápidos, sem I/O
├── integration/     # tocam banco/rede reais (ou containers)
└── conftest.py      # fixtures compartilhadas
```

Regras:
- `domain/` **não importa** nada de `infrastructure/` nem de framework.
- Dependências apontam para dentro (domínio é o núcleo estável).
- Um módulo `interfaces/` fino: converte request → chama caso de uso → converte resposta.

---

## 5. Convenções de código

- **Type hints são obrigatórias** em toda função pública e assinatura de método.
  Prefira tipos precisos (`Sequence`, `Mapping`, `Protocol`) a `Any`.
- **Imports:** absolutos, ordenados pelo ruff. Nada de `from x import *`.
- **Nomes:** `snake_case` funções/variáveis, `PascalCase` classes,
  `UPPER_SNAKE` constantes. Nomes descritivos > comentários.
- **Docstrings** em módulos, classes e funções não triviais (estilo Google ou
  NumPy — escolha um). Documente o _porquê_, não o _o quê_.
- **Funções pequenas e coesas.** Uma responsabilidade. Se passa de ~30 linhas ou
  aninha muito, quebre.
- **Imutabilidade quando possível:** `@dataclass(frozen=True)`, tuplas, evitar
  mutação de argumentos.
- **Evite estado global mutável.** Configuração via injeção, não singletons ocultos.
- **`pathlib` em vez de `os.path`**; **f-strings** para formatação.
- **Nunca engula exceções** com `except: pass`. Capture o tipo específico e trate
  ou relance com contexto (`raise ... from e`).
- **Comentários** só quando o código não consegue se explicar. Comentário
  desatualizado é pior que ausência.

---

## 6. Princípios de arquitetura

- **Separação em camadas** (ver seção 4): domínio puro no centro; framework e I/O
  na borda.
- **Dependa de abstrações, não de implementações.** Defina _portas_ com
  `typing.Protocol` no domínio/aplicação; implemente na infraestrutura.
- **Regra de negócio não sabe onde os dados moram.** O caso de uso recebe um
  repositório (interface); quem decide se é Postgres ou memória é o wiring.
- **DTOs nas bordas.** Modelos de domínio ≠ schemas de request/response ≠ modelos
  de ORM. Não vaze entidade do ORM direto para a API.
- **Efeitos colaterais explícitos e nas pontas.** Lógica pura no meio, I/O
  concentrado onde dá para testar e observar.
- **Fail fast e valide na entrada.** Depois que o dado entra no domínio, ele já
  deve estar validado (Pydantic na borda faz esse trabalho).
- **YAGNI + KISS.** Não crie abstração para um único uso. Padrão só quando o
  problema que ele resolve realmente existe.

---

## 7. Padrões de projeto (idiomáticos em Python)

> Aplique com parcimônia. Em Python, muita coisa que em Java pede uma classe se
> resolve com função, decorator ou closure. Prefira o simples.

- **Repository** — isola persistência atrás de uma interface. Domínio fala com
  `UserRepository` (Protocol); a implementação usa ORM/SQL.

  ```python
  from typing import Protocol

  class UserRepository(Protocol):
      def get(self, user_id: int) -> User | None: ...
      def add(self, user: User) -> None: ...

  class SqlAlchemyUserRepository:  # infra — satisfaz o Protocol por estrutura
      def __init__(self, session: Session) -> None:
          self._session = session
      def get(self, user_id: int) -> User | None:
          return self._session.get(UserModel, user_id)
      def add(self, user: User) -> None:
          self._session.add(user)
  ```

- **Service / Use Case** — uma operação de negócio orquestrando repositórios e
  regras. Sem detalhes de HTTP.
- **Unit of Work** — agrupa mudanças numa transação atômica (commit/rollback);
  encaixa bem com o Repository.
- **Dependency Injection** — injete dependências pelo construtor/argumentos.
  Use o sistema nativo do framework (FastAPI `Depends`, Django settings) ou um
  container (`dependency-injector`, `svcs`, `punq`) só se o wiring crescer.
- **Adapter** — envolve API/SDK de terceiros atrás da sua interface. Isola você de
  mudanças externas e facilita mockar em teste. (Padrão-chave para integrações.)
- **Strategy** — comportamentos intercambiáveis; em Python, muitas vezes só passar
  uma função/callable já é a estratégia.
- **Factory** — centraliza criação complexa; frequentemente uma `@classmethod`
  `from_...` ou uma função simples resolve.
- **DTO** — `pydantic.BaseModel` ou `@dataclass` para transportar dados entre
  camadas sem acoplar ao ORM.
- **Context manager** (`with`/`@contextmanager`) para gerenciar recursos (conexão,
  lock, transação) — o padrão RAII do Python.

**Anti-uso comum:** Singleton disfarçado de módulo global mutável, herança
profunda quando composição bastaria, e "manager" que faz de tudo (God object).

---

## 8. Guia por framework

> **Mantenha apenas a subseção do seu projeto.**

### 8a. FastAPI

- **Async por padrão** em rotas que fazem I/O; use libs async (`asyncpg`,
  `httpx.AsyncClient`). **Nunca chame função bloqueante dentro de rota async** —
  jogue em `run_in_threadpool` ou torne o handler `def` normal.
- **Pydantic v2** para request/response models; declare `response_model` para
  filtrar campos e documentar.
- **`Depends`** para injeção (sessão de DB, usuário autenticado, repositórios).
  Uma dependência por responsabilidade.
- **Roteadores por domínio** (`APIRouter`), não um `main.py` gigante.
- **Sessão de banco por request** via dependency com `yield` (garante fechamento).
- **`BackgroundTasks`** para trabalho leve pós-resposta; task queue para o pesado.
- **Não coloque regra de negócio na rota.** Rota → chama use case → retorna DTO.

### 8b. Django

- **Fat models / service layer, thin views.** Regras no model ou em módulos
  `services.py`; views só orquestram.
- **N+1 é o inimigo nº 1:** use `select_related` (FK/OneToOne) e
  `prefetch_related` (M2M/reverse). Rode com `django-debug-toolbar` no dev.
- **Migrations sempre versionadas e revisadas.** Nunca edite migration já aplicada
  em produção. Evite operações que travam tabela grande sem planejar.
- **`QuerySet` é preguiçoso** — encadeie filtros, materialize só quando precisar.
  Use `.only()`/`.defer()`, `.values()`, `annotate`/`aggregate` para empurrar
  trabalho ao banco.
- **`bulk_create`/`bulk_update`** para volume; nada de `save()` em loop.
- **DRF:** serializers como DTO/validação; `ViewSet` + `Router`; paginação sempre
  ativada; `SerializerMethodField` com cuidado (pode reintroduzir N+1).
- **Settings por ambiente** (`django-environ`), segredos fora do repositório.
- **`transaction.atomic`** para operações que precisam ser tudo-ou-nada.

### 8c. Flask

- **App factory** (`create_app()`) + **Blueprints** por domínio. Sem `app` global.
- **Extensões inicializadas no factory** (`db.init_app(app)`), não no import.
- **Camada de serviço explícita** — Flask não impõe estrutura, então a disciplina
  é sua. Não jogue lógica na view.
- **SQLAlchemy 2.0 style** (`select()`, `session.scalars(...)`), sessão com escopo
  de request e fechada no teardown.
- **Config por objeto/ambiente**; `pydantic-settings` para validar env vars.
- **Marshmallow ou Pydantic** para (de)serialização e validação de entrada.
- Para I/O concorrente pesado, considere ASGI (Quart tem API compatível) ou uma
  task queue — o modelo síncrono do Flask limita concorrência.

---

## 9. Performance

> Regra zero: **meça antes de otimizar.** Perfil primeiro (`cProfile`,
> `py-spy top`, `line_profiler`, `django-debug-toolbar`), otimize o gargalo real.

- **Banco costuma ser o gargalo, não a CPU:**
  - Elimine **N+1** (eager loading: `select_related`/`prefetch_related`,
    `joinedload`/`selectinload` no SQLAlchemy).
  - **Índices** nas colunas de filtro/join/ordenação; confira o plano com
    `EXPLAIN ANALYZE`.
  - **Paginação sempre** em listagens; nunca `SELECT *` sem limite.
  - Operações em **lote** (`bulk_*`), não linha a linha.
  - **Connection pooling** (PgBouncer / pool do driver) sob carga.
- **Async só onde há I/O concorrente** e a stack toda é async. Async não acelera
  CPU-bound — para isso use `ProcessPoolExecutor` ou mova para serviço dedicado.
- **Não bloqueie o event loop:** chamada bloqueante em código async mata a
  concorrência.
- **Cache com intenção:** `functools.lru_cache`/`cache` para pura computação;
  Redis para dados compartilhados; **defina invalidação** desde o início.
- **Generators/iteradores** para grandes volumes — não carregue tudo em memória.
- **Estruturas de dados certas:** `set`/`dict` para lookup O(1); `deque` para
  fila; evite `in` em lista grande.
- **Serialização importa:** `orjson` onde JSON é hot path.
- **Task queue** para trabalho pesado/lento fora do ciclo de request.
- Evite micro-otimização que sacrifica legibilidade sem ganho medido.

---

## 10. Testes

- **Pirâmide:** muitos testes unitários (rápidos, sem I/O), alguns de integração,
  poucos e2e.
- **`pytest`** com fixtures; nomes descritivos (`test_estorno_falha_se_saldo_insuficiente`).
- **Arrange–Act–Assert**; um comportamento por teste.
- **Injete dependências** para testar sem tocar rede/banco — o Repository/Adapter
  atrás de Protocol torna o mock trivial.
- **`pytest.mark.parametrize`** para cobrir casos de borda sem duplicar.
- **Fábricas de dados** (`factory_boy`/`model_bakery` no Django) em vez de fixtures
  gigantes.
- **Teste comportamento observável, não implementação.** Não asserte detalhes
  internos que vão mudar.
- **Cobertura é sinal, não meta.** Priorize caminhos críticos e regras de negócio.
- Teste integração com banco/serviços reais via containers (`testcontainers`)
  quando fizer diferença.

---

## 11. Segurança

- **Segredos fora do repositório** — sempre env vars/secret manager; nunca commit.
  Valide-os no boot (`pydantic-settings`).
- **Confie em nada da entrada.** Valide/sanitize toda entrada externa na borda.
- **SQL sempre parametrizado / via ORM.** Nunca concatene input em query.
- **Senhas com hash forte** (`argon2`/`bcrypt`), nunca em texto puro nem log.
- **Não logue dados sensíveis** (PII, tokens, cartão). Atenção à LGPD.
- **AuthN/AuthZ explícitas** em cada endpoint; negue por padrão.
- **Dependências auditadas** (`uv`/`pip-audit`); atualize CVEs conhecidos.
- **Timeouts e limites** em toda chamada externa e upload.

---

## 12. Erros e logging

- **Logging estruturado** (`structlog`), não `print`. Nível certo: `debug` para
  diagnóstico, `info` para eventos de negócio, `error` para falhas acionáveis.
- **Correlacione logs** (request id / trace id) para rastrear um fluxo.
- **Exceções específicas do domínio** (`SaldoInsuficienteError`) em vez de
  `Exception` genérica; deixe a borda traduzir para HTTP status.
- **`raise ... from e`** para preservar a causa.
- **Não use exceção como fluxo de controle normal.**
- Erros ao usuário: mensagem clara e segura; detalhes técnicos só no log.

---

## 13. Anti-padrões a evitar

- Regra de negócio dentro de view/rota/serializer.
- Entidade do ORM vazando direto para a API (sem DTO).
- `except Exception: pass` e engolir erros silenciosamente.
- Query em loop (N+1) e `.all()` sem paginação.
- Estado global mutável / singletons ocultos.
- Otimizar sem medir; abstrair para um único uso (over-engineering).
- Mutar argumentos mutáveis; usar `[]`/`{}` como default de parâmetro.
- Dependência nova para o que a stdlib já faz.
- Comentário/docstring desatualizado.

---

## 14. Git e commits

- **Conventional Commits:** `feat:`, `fix:`, `refactor:`, `test:`, `docs:`,
  `perf:`, `chore:`. Mensagem no imperativo, explica o _porquê_.
- **Commits pequenos e atômicos**; um assunto por commit.
- **Nada de segredo, arquivo gerado ou `.env` no histórico.**
- Rode a suíte de qualidade (seção 3) antes de commitar.
- `<sua política de branch/PR aqui>`

---

## 15. Instruções operacionais para o assistente

- **Antes de concluir qualquer tarefa**, rode format + lint + mypy + testes
  (seção 3) e reporte o resultado.
- **Não introduza dependências** sem justificar e checar o que já existe.
- **Respeite a arquitetura em camadas**; se precisar quebrar uma regra deste
  arquivo, diga explicitamente por quê.
- **Prefira editar o mínimo necessário**; não refatore código não relacionado
  junto de uma correção.
- **Ao mudar comportamento, atualize ou adicione teste.**
- Se algo neste `CLAUDE.md` estiver desatualizado em relação ao código real,
  aponte em vez de seguir cegamente.
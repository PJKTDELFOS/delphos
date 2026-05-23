# Delfos

**Análise estática para projetos FastAPI — detecte problemas antes de subir a aplicação.**

Delfos é um framework de verificação de integridade para projetos construídos com FastAPI e frameworks ASGI assíncronos similares. Inspirado no `python manage.py check` do Django, ele preenche uma lacuna real: frameworks unopinionated como FastAPI não têm visão global da aplicação e, portanto, não podem verificar a consistência entre suas partes.

Delfos lê seu código via AST — sem importar, sem executar, sem efeitos colaterais — e reporta erros, inconsistências e problemas de configuração antes que eles se manifestem em runtime.

---

## O problema que ele resolve

O Django consegue verificar a integridade da aplicação porque controla todo o stack: ORM, migrations, roteamento, admin e configuração estão todos declarados dentro do mesmo ecossistema. O framework sabe o que esperar de cada parte.

O FastAPI não tem esse controle. Você escolhe livremente o ORM, o sistema de migrations, a estrutura de projeto e a estratégia de configuração. Essa liberdade é uma das maiores vantagens do framework — e também significa que ninguém está verificando se as peças se encaixam corretamente.

O resultado são erros silenciosos comuns em projetos FastAPI:

```python
# O endpoint declara response_model=UserResponse
# UserResponse não tem o campo 'role'
# O model ORM tem 'role'
# Nenhum erro em runtime — o campo simplesmente desaparece da resposta
# Ninguém avisa

class UserResponse(BaseModel):
    id: int
    email: str
    # 'role' sumiu aqui

@router.get("/users/{id}", response_model=UserResponse)
async def get_user(id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == id).first()
```

Delfos detecta isso antes de você subir o servidor.

---

## Instalação

```bash
pip install delfos
```

---

## Uso

```bash
# verificar o projeto no diretório atual
delfos check .

# verificar um projeto em outro path
delfos check /caminho/do/projeto

# exibir apenas erros críticos
delfos check . --level error

# saída em JSON (para integração com CI/CD)
delfos check . --format json
```

### Exemplo de saída

```
Delfos — analisando projeto em ./meu_projeto

  [ERRO]     SCH001  schemas.py:14     Campo 'role' presente em UserModel mas ausente em UserResponse
  [ERRO]     RTE002  routers/users.py  Rota duplicada: GET /users definida também em routers/legacy.py:87
  [AVISO]    RTE004  routers/orders.py Endpoint POST /orders sem response_model declarado
  [ERRO]     ORM003  models/order.py   ForeignKey 'orders.user_id' aponta para tabela não mapeada
  [AVISO]    SEC001  main.py:8         CORS allow_origins=["*"] fora de contexto de desenvolvimento
  [INFO]     ASY001  services/auth.py  Função async sem nenhum await interno

6 problema(s) encontrado(s): 3 erro(s), 2 aviso(s), 1 info
```

O processo retorna exit code `1` quando há erros — integrável diretamente com pipelines CI/CD.

---

## Checks implementados

Delfos executa 35 checks organizados em três grupos.

### Schemas e Models

| ID | Descrição | Severidade |
|---|---|---|
| SCH001 | Campo presente no ORM model mas ausente no Pydantic schema de resposta | ERROR |
| SCH002 | Campo com tipos incompatíveis entre ORM model e schema | ERROR |
| SCH003 | Endpoint sem `response_model` declarado | WARNING |
| SCH004 | Schema Pydantic definido sem nenhum campo | WARNING |
| SCH005 | Campo obrigatório sem default e sem `Optional` | WARNING |
| SCH006 | Herança de schema com campos conflitantes entre pai e filho | WARNING |
| SCH007 | Dois schemas com estrutura idêntica — candidatos a refatoração | INFO |
| SCH008 | Campos como `email`, `cpf`, `url` sem uso de `EmailStr`, `HttpUrl` ou validators | INFO |
| SCH009 | Conflito estrutural entre schemas de rotas versionadas (`/v1` vs `/v2`) | INFO |

### Rotas e Endpoints

| ID | Descrição | Severidade |
|---|---|---|
| RTE001 | Dois endpoints com mesmo path e mesmo método HTTP | ERROR |
| RTE002 | Parâmetro de path declarado na URL mas ausente na assinatura da função | ERROR |
| RTE003 | `APIRouter` criado mas nunca incluído via `app.include_router()` | WARNING |
| RTE004 | Endpoint sem `response_model` declarado | WARNING |
| RTE005 | Endpoint definido com `def` em vez de `async def` | WARNING |
| RTE006 | Endpoint sem nenhuma dependência de autenticação declarada | INFO |

### Dependências

| ID | Descrição | Severidade |
|---|---|---|
| DEP001 | Função passada ao `Depends()` não foi definida nem importada | ERROR |
| DEP002 | Função de dependência síncrona em contexto assíncrono | WARNING |
| DEP003 | Cadeia de dependências transitivas com elo não rastreável (best-effort) | INFO |

### ORM e Models

| ID | Descrição | Severidade |
|---|---|---|
| ORM001 | SQLAlchemy model sem `__tablename__` | ERROR |
| ORM002 | `ForeignKey` apontando para tabela não mapeada em nenhum model | ERROR |
| ORM003 | `relationship()` definido sem `back_populates` correspondente | WARNING |
| ORM004 | Model sem migration correspondente no Alembic | WARNING |
| ORM005 | Herança múltipla de ORM com cadeia não resolvível via AST | WARNING |

### Ambiente e Configuração

| ID | Descrição | Severidade |
|---|---|---|
| ENV001 | Variável de ambiente referenciada sem default e sem declaração em `.env` | ERROR |
| ENV002 | Mais de um módulo de configuração sem relação clara entre eles | INFO |
| ENV003 | Segredo hardcoded — string com padrão de senha, token ou chave API | ERROR |

### Imports

| ID | Descrição | Severidade |
|---|---|---|
| IMP001 | Módulo importado que não existe no projeto | ERROR |
| IMP002 | Circular import detectado entre módulos | ERROR |
| IMP003 | Import declarado mas nunca utilizado no arquivo | INFO |

### Segurança

| ID | Descrição | Severidade |
|---|---|---|
| SEC001 | `allow_origins=["*"]` fora de contexto de desenvolvimento | WARNING |
| SEC002 | Middleware de autenticação declarado após middleware que depende do usuário autenticado | WARNING |

### Qualidade e Consistência

| ID | Descrição | Severidade |
|---|---|---|
| QUA001 | Função declarada `async def` sem nenhum `await` interno | INFO |
| QUA002 | Pydantic v1 (`class Config`) e v2 (`model_config`) misturados no mesmo projeto | WARNING |
| QUA003 | Endpoint com lógica de negócio extensa — recomendado mover para camada de serviço | INFO |
| QUA004 | Tipos genéricos profundamente aninhados com possível incompatibilidade | WARNING |

---

## Configuração

Crie um arquivo `delfos.toml` na raiz do projeto para customizar o comportamento:

```toml
[delfos]
# diretórios a ignorar
exclude = ["tests/", "migrations/", "alembic/"]

# severidade mínima para exibir (debug, info, warning, error)
min_level = "info"

# checks para desabilitar
disable = ["QUA003", "SCH007"]

# threshold para QUA003 (linhas de lógica no endpoint)
[delfos.checks.QUA003]
max_lines = 20
```

---

## Integração com CI/CD

### GitHub Actions

```yaml
- name: Verificar integridade do projeto
  run: |
    pip install delfos
    delfos check . --level warning
```

### Pre-commit hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: delfos
        name: Delfos — verificação estática
        entry: delfos check .
        language: system
        types: [python]
        pass_filenames: false
```

---

## Como funciona

Delfos nunca importa nem executa seu código. Todo o processo é baseado em leitura e análise da árvore sintática abstrata (AST) dos arquivos `.py` do projeto.

```
Arquivos .py do projeto
        ↓
   Walker — descobre arquivos
        ↓
   Parser — gera AST de cada arquivo
        ↓
   ProjectMap — visão global consolidada
        ↓
   Checks — 35 verificações sobre o ProjectMap
        ↓
   Resultado — issues com severidade, arquivo e linha
```

Isso significa que Delfos funciona sem banco de dados disponível, sem variáveis de ambiente configuradas, sem dependências instaladas. Pode rodar em qualquer máquina com acesso ao código-fonte.

---

## Frameworks suportados

| Framework | Suporte |
|---|---|
| FastAPI | ✅ Completo |
| Starlette | ✅ Rotas e middleware |
| Litestar | 🔄 Planejado |
| Sanic | 🔄 Planejado |

### ORMs suportados

| ORM | Suporte |
|---|---|
| SQLAlchemy | ✅ Completo |
| Tortoise ORM | 🔄 Planejado |
| Beanie (MongoDB) | 🔄 Planejado |

---

## O que Delfos não verifica

Por design, Delfos não verifica situações que exigem execução do código:

- **Campos definidos dinamicamente** via `setattr`, metaclasses ou `__init_subclass__`
- **Validators Pydantic** que transformam o tipo do campo em runtime
- **Queries N+1** — requer análise de fluxo de execução
- **Herança fora do projeto** — classes definidas em dependências externas

Esses casos podem ser revisitados em versões futuras com um modo opcional de inspeção em runtime.

---

## Roadmap

- **v0.1** — checks do Grupo 1 (totalmente implementáveis via AST)
- **v0.2** — checks do Grupo 2 (esforço moderado, grafo de projeto)
- **v0.3** — checks do Grupo 3 (complexos selecionados)
- **v1.0** — suporte completo a Litestar e Tortoise ORM, plugin para VS Code

---

## Contribuindo

Contribuições são bem-vindas. Cada check vive em seu próprio módulo dentro de `delfos/checks/` e implementa uma interface simples — adicionar um novo check não exige modificar o núcleo do projeto.

Consulte [CONTRIBUTING.md](CONTRIBUTING.md) para detalhes.

---

## Licença

Apache 2.0 — veja [LICENSE](LICENSE) para os termos completos.
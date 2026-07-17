# Organização Tramar — Estrutura de Engenharia e Automação

> Documento de apresentação da estrutura técnica da organização **`tramar-br`** no GitHub.
> Objetivo: dar ao gestor uma visão clara de **como o desenvolvimento, a publicação (deploy) e o controle de qualidade** estão organizados — sem necessidade de conhecimento técnico aprofundado.
>
> _Última atualização: julho de 2026._

---

## 1. Visão geral

A organização `tramar-br` reúne todos os sistemas da Tramar no GitHub. Toda a engenharia segue um **modelo padronizado e centralizado**: em vez de cada sistema ter sua própria configuração de testes e publicação, **todos compartilham a mesma "esteira" de automação**.

Essa esteira mora num repositório central chamado **`cicd-templates`** (mantido pela equipe Ebravo). Os sistemas da Tramar apenas "se conectam" a ela. Na prática isso significa:

- **Padronização**: todo sistema é construído, testado e publicado da mesma forma.
- **Manutenção única**: uma melhoria na esteira vale para todos os sistemas de uma vez, sem retrabalho repo a repo.
- **Rastreabilidade**: cada entrega fica registrada e versionada automaticamente.

```
   Desenvolvedor abre uma tarefa  ─►  cria branch  ─►  abre Pull Request  ─►  code review  ─►  merge  ─►  deploy
          │                            │                    │                     │            │          │
          └──────────────────── processo padronizado e automatizado pela esteira central ────────────────┘
```

---

## 2. Sistemas (repositórios)

A organização possui **9 repositórios ativos**, todos **privados** (mais 4 arquivados — ver 2.3). Eles se dividem em dois grupos:

### 2.1 Sistemas com esteira de deploy configurada — **monorepos**

Cada módulo (Pedidos e TPS) foi consolidado num **único repositório (monorepo)** que reúne o backend (`api`) e o frontend (`web`) lado a lado (pastas `apps/api` e `apps/web`), compartilhando a mesma esteira. Cada camada é construída, versionada e publicada de forma independente (é possível deployar só a `api`, só o `web`, ou ambos). A **forma de publicação difere por módulo** (ver seção 4):

| Sistema (monorepo) | Módulo | Camadas | Tecnologia | Publicação |
|---|---|---|---|---|
| `tramar-pedidos` | Pedidos | `api` + `web` | Java (Spring Boot) + Angular | **Automática** — servidores EC2 da Ebravo (AWS) |
| `tramar-tps` | TPS | `api` + `web` | Java (Spring Boot) + Angular | **Manual** — servidores on-premises da Tramar |

### 2.2 Repositórios com automação parcial

Têm o controle de qualidade (CI) e a padronização de branches ativos, mas **ainda não têm a esteira de deploy configurada** (sem o arquivo `pipeline.yml`). Quando precisarem de deploy, basta concluir a configuração.

| Repositório | Observação |
|---|---|
| `web-base` | Base/modelo para novos frontends |
| `dispatch` | — |
| `desconecta-sapiens` | — |
| `formulas-borracha` | — |
| `chao-de-fabrica` | — |
| `move-track` | — |
| `sdc-javaswing` | Sistema legado (Java Swing, desktop); usa branch `master` |

### 2.3 Repositórios arquivados (consolidados em monorepos)

Os quatro repositórios abaixo foram **migrados para monorepos** (seção 2.1) e ficaram **arquivados** (somente leitura). Todo o histórico de código foi preservado dentro do monorepo correspondente, em `apps/api` / `apps/web`.

| Repositório arquivado | Consolidado em |
|---|---|
| `tramar-pedidos-api` | `tramar-pedidos` (`apps/api`) |
| `tramar-pedidos-web` | `tramar-pedidos` (`apps/web`) |
| `tramar-tps-api` | `tramar-tps` (`apps/api`) |
| `tramar-tps-web` | `tramar-tps` (`apps/web`) |

---

## 3. Fluxo de trabalho, Merge e Release

### Modelo de versionamento de código

Utilizamos o modelo **trunk-based gitflow**: existe **uma branch principal (`main`)**, e o trabalho é feito em **branches curtas** que são integradas de volta à `main` com frequência. Cada ramo de trabalho segue o padrão por tipo de tarefa:

- `feature/…` → nova funcionalidade
- `bug/…` → correção de erro
- `task/…` → tarefa técnica

### Procedimento de Merge e Release

Após a **abertura do Pull Request**, é **obrigatório um code review** antes de aprovar e fazer o merge. Isso garante que os desenvolvedores estão seguindo os procedimentos padrão e mantendo a qualidade de código. A regra é reforçada tecnicamente pela proteção da branch principal, que **exige a aprovação de pelo menos 1 revisor** (sem exceções, inclusive para administradores).

### Passo a passo de uma entrega

1. **Tarefa (Issue)** — toda demanda começa registrada como uma tarefa no GitHub.
2. **Branch (ramo de trabalho)** — criada para a tarefa e **renomeada automaticamente** seguindo o padrão `feature/`, `bug/` ou `task/`.
3. **Verificação automática (CI)** — a cada envio de código, o sistema **compila e valida** automaticamente; se algo quebra, o autor é avisado na hora.
4. **Pull Request + Code Review** — ao propor a junção, é **obrigatória** a revisão e aprovação de outro desenvolvedor.
5. **Merge** — ao aprovar e juntar na `main`, o ramo de trabalho é **excluído automaticamente** para manter o repositório limpo.
6. **Deploy** — feito de forma **intencional e controlada** (acionado manualmente), nunca por acidente (ver seção 4).

---

## 4. Procedimento de Deploy

> Aplica-se **aos 2 monorepos com esteira configurada**: `tramar-pedidos` e `tramar-tps` (cada um com as camadas `api` e `web`).

O deploy é **sempre acionado manualmente** por um desenvolvedor, pela aba **Actions** do GitHub. Ao acionar, escolhe-se **qual camada publicar** — `api`, `web` ou `ambos` (parâmetro `deploy_apps`). A esteira **gera uma imagem versionada** de cada camada e a publica no repositório de imagens da AWS (**ECR**), em repositórios separados por camada (`tramar-pedidos-api`, `tramar-pedidos-web`, etc.). O que acontece depois depende do módulo (ver 4.3).

### 4.1 Deploy em Homologação — após push em branch `feature`/`bug`/`task`

1. Acessar a aba **Actions** do repositório.
2. Garantir que o **CI da branch executou com sucesso**.
3. Selecionar o workflow **CD (Deploy)** → **Run workflow**.
4. Selecionar a **branch** desejada e o parâmetro **ambiente = `homologacao`**.

### 4.2 Deploy em Produção — após o merge na `main`

1. Acessar a aba **Actions** do repositório.
2. Garantir que o **CI executou com sucesso**.
3. Selecionar o workflow **CD (Deploy)** → **Run workflow**.
4. Selecionar a branch **`main`** e o parâmetro **ambiente = `producao`**.

> O deploy em produção é **restrito aos usuários autorizados** (ver seção 6).

### 4.3 Conclusão do deploy — específico por projeto

**`tramar-pedidos` (camadas `api` / `web`) — Automático**
O deploy é **concluído automaticamente** nos **servidores EC2 da Ebravo (AWS)**. Além de acionar o workflow (4.1 / 4.2), **nada é manual**.

**`tramar-tps` (camadas `api` / `web`) — Manual (on-premises)**
A esteira **apenas gera a imagem versionada no ECR**. O deploy nos **servidores locais da Tramar** (datacenter próprio) é feito **manualmente** por um desenvolvedor:

1. **Conectar no servidor** via SSH:
   - **Homologação**: `ssh persefone@homologtps`
   - **Produção**: `ssh persefone@persefone`
2. **Executar o script de deploy**, substituindo `VERSAO_IMAGEM` pela versão gerada no deploy:
   - **Web**: `/sdc/deploy_sdc_web/deploy-web.sh VERSAO_IMAGEM`
   - **API**: `/sdc/deploy_sdc_api/deploy-api.sh VERSAO_IMAGEM`

---

## 5. Ambientes, infraestrutura e versionamento

### Infraestrutura

- **ECR (AWS, região São Paulo `sa-east-1`)** — repositório central de **imagens versionadas**, com um repositório por camada (`tramar-pedidos-api`, `tramar-pedidos-web`, `tramar-tps-api`, `tramar-tps-web`).
- **`tramar-pedidos`** — roda nos **servidores EC2 da Ebravo (AWS)**, com deploy automático.
- **`tramar-tps`** — roda no **datacenter próprio da Tramar (on-premises)**: servidores locais `homologtps` (homologação) e `persefone` (produção), com deploy manual.
- **Não há ECS/Kubernetes na AWS** — o ambiente da Tramar é **todo on-premises**.

### Versionamento automático

A cada deploy, a versão do sistema é incrementada automaticamente:

- **Homologação** — gera uma versão de teste rastreável, identificando o número da entrega e a tarefa de origem.
  Exemplo: `1.52.0-B34-F1` (entrega nº 34, referente à funcionalidade da tarefa nº 1).
- **Produção** — gera uma versão "oficial" limpa e cria um **registro de release** no GitHub.
  Exemplo: `1.52.0` → `1.53.0`. Nos monorepos, a tag/release é **prefixada pela camada** para distinguir api e web na aba *Tags* — ex.: `api/1.39.0`, `web/1.30.0`.

Isso garante que sempre se saiba **exatamente qual versão está em cada ambiente** e **de onde ela veio** — inclusive a versão usada no deploy manual on-premises (`VERSAO_IMAGEM`).

---

## 6. Rollback (voltar para uma versão anterior)

- **`tramar-pedidos` (camadas `api` / `web`)** — rollback **habilitado pela esteira**: é possível voltar para uma imagem anterior do ECR de forma automatizada (por camada).
- **Demais sistemas** (`tramar-tps` e os que rodam on-premises) — o rollback é feito **manualmente, na rede interna**, pela equipe.

---

## 7. Segurança e controle de acesso

A organização foi configurada com **proteções para garantir que nada vá ao ar sem controle**:

### Proteção do código principal

Todos os repositórios têm a branch principal (`main`/`master`) **protegida**, com as seguintes regras:

- ✅ **Revisão obrigatória**: nenhum código entra na linha principal sem aprovação de **pelo menos 1 revisor**.
- ✅ **Regra vale para todos, inclusive administradores** (sem exceções).
- ✅ Apenas usuários autorizados podem integrar código diretamente.

### Quem pode publicar em PRODUÇÃO

A publicação em produção é **restrita a uma lista controlada de pessoas**. Hoje, estão autorizados:

| Usuário |
|---|
| `juniorebravo` |

> Qualquer tentativa de publicar em produção por alguém fora dessa lista é **automaticamente bloqueada** pelo sistema.

### Membros da organização

| Usuário | Papel |
|---|---|
| `juniorebravo` | Administrador |
| `murilo-tramar` | Administrador |
| `carloscarvalho-debug` | Membro |
| `gustavo-souza-ebravo` | Membro |
| `MatheusIsidio` | Membro |
| `sophiaenzo` | Membro |

---

## 8. Configurações técnicas (referência)

Esta seção é uma referência para a equipe técnica. Por segurança, **valores sensíveis (senhas, chaves) nunca ficam expostos** — ficam guardados de forma criptografada nos "segredos" da organização.

### Variáveis da organização

| Variável | Função |
|---|---|
| `DEPLOY_PROD_ALLOWED` | Lista de usuários autorizados a publicar em produção |

### Segredos da organização (valores ocultos)

| Segredo | Função |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Acesso à infraestrutura AWS (imagens dos sistemas no ECR) |
| `SSH_KEY_HOMOL` | Chave de acesso ao servidor de homologação |
| `SSH_KEY_PROD` | Chave de acesso ao servidor de produção |
| `GH_TOKEN_BUMP` | Permite o versionamento automático |
| `PROJECT_PAT` | Permite as automações entre repositórios/organizações |

### Esteira de automação (workflows)

Os repositórios instalam apenas "atalhos" leves que apontam para a esteira central `ebravo-br/cicd-templates`. Os principais são:

| Automação | O que faz |
|---|---|
| `ci` | Compila e valida o código a cada envio |
| `deploy` | Gera a imagem versionada e publica (acionado manualmente) |
| `rollback` | Volta para uma versão anterior (projetos `tramar-pedidos-*`) |
| `branch-naming` | Padroniza o nome dos ramos de trabalho |
| `issue-epic` | Classifica e registra a tarefa ao ser criada |
| `project-automation` | Limpa branches automaticamente após o merge |

---

## 9. Governança e manutenção

- **Centralização**: toda a lógica de automação vive em **um único lugar** (`cicd-templates`). Atualizações são aplicadas a todos os sistemas simultaneamente, reduzindo custo de manutenção e risco de inconsistência.
- **Padrão único entre Ebravo e Tramar**: as duas organizações compartilham a mesma esteira, facilitando a operação por uma equipe comum.
- **Modelo híbrido de infraestrutura**: os sistemas de **Pedidos** rodam na nuvem (EC2 da Ebravo, deploy automático) e os de **TPS** rodam no **datacenter on-premises da Tramar** (deploy manual), atendidos pela mesma esteira de build/versionamento.
- **Consolidação em monorepos**: cada módulo (Pedidos e TPS) passou a viver num **único repositório** com backend e frontend juntos (`apps/api` + `apps/web`), simplificando a manutenção e permitindo entregas coordenadas — com deploy e versionamento ainda **independentes por camada**. Os repositórios antigos (`*-api`/`*-web`) foram arquivados com o histórico preservado.

---

### Resumo executivo

> A organização Tramar opera com uma **esteira de desenvolvimento padronizada, automatizada e auditável**. Da abertura da tarefa até a publicação, o processo é controlado, com **revisão de código obrigatória antes de cada merge** (modelo trunk-based gitflow) e **deploy acionado de forma intencional**. A publicação dos sistemas de **Pedidos** é automática na nuvem (EC2 da Ebravo); a dos sistemas **TPS** é feita manualmente nos servidores **on-premises** da Tramar a partir da imagem versionada gerada pela esteira. A publicação em produção é **restrita a usuários autorizados**, e há **capacidade de rollback** (automatizada para Pedidos; manual para o ambiente on-premises).

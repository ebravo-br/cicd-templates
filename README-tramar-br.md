# Organização Tramar — Estrutura de Engenharia e Automação

> Documento de apresentação da estrutura técnica da organização **`tramar-br`** no GitHub.
> Objetivo: dar ao gestor uma visão clara de **como o desenvolvimento, a publicação (deploy) e o controle de qualidade** estão organizados — sem necessidade de conhecimento técnico aprofundado.
>
> _Última atualização: junho de 2026._

---

## 1. Visão geral

A organização `tramar-br` reúne todos os sistemas da Tramar no GitHub. Toda a engenharia segue um **modelo padronizado e centralizado**: em vez de cada sistema ter sua própria configuração de testes e publicação, **todos compartilham a mesma "esteira" de automação**.

Essa esteira mora num repositório central chamado **`cicd-templates`** (mantido pela equipe Ebravo). Os sistemas da Tramar apenas "se conectam" a ela. Na prática isso significa:

- **Padronização**: todo sistema é construído, testado e publicado da mesma forma.
- **Manutenção única**: uma melhoria na esteira vale para todos os sistemas de uma vez, sem retrabalho repo a repo.
- **Rastreabilidade**: cada entrega fica registrada e versionada automaticamente.

```
   Desenvolvedor abre uma tarefa  ─►  cria branch  ─►  abre Pull Request  ─►  revisão  ─►  publica
          │                              │                  │                  │            │
          └──────────────── tudo monitorado e automatizado no GitHub Projects ─────────────┘
```

---

## 2. Sistemas (repositórios)

A organização possui **11 repositórios**, todos **privados**. Eles se dividem em dois grupos:

### 2.1 Sistemas com publicação automatizada (deploy)

Estes têm a esteira completa: testes, versionamento, geração de imagem e publicação em servidor.

| Sistema | Função | Tecnologia |
|---|---|---|
| `tramar-pedidos-api` | Backend do módulo de Pedidos | Java (Spring Boot) |
| `tramar-pedidos-web` | Frontend do módulo de Pedidos | Angular |
| `tramar-tps-api` | Backend do módulo TPS | Java (Spring Boot) |
| `tramar-tps-web` | Frontend do módulo TPS | Angular |

### 2.2 Repositórios com automação parcial

Têm o controle de qualidade e a gestão de tarefas ativos, mas **ainda não têm a publicação automática configurada** (sem o arquivo `pipeline.yml`). Quando precisarem de deploy automatizado, basta concluir a configuração.

| Repositório | Observação |
|---|---|
| `web-base` | Base/modelo para novos frontends |
| `dispatch` | — |
| `desconecta-sapiens` | — |
| `formulas-borracha` | — |
| `chao-de-fabrica` | — |
| `move-track` | — |
| `sdc-javaswing` | Sistema legado (Java Swing, desktop); usa branch `master` |

---

## 3. Como uma entrega acontece (fluxo de trabalho)

O processo é o mesmo para todos os sistemas e foi desenhado para ser **simples e à prova de erro**:

1. **Tarefa (Issue)** — toda demanda começa como uma tarefa registrada no GitHub. Ao ser criada, ela é **automaticamente** adicionada ao quadro de acompanhamento e classificada por Épico (área/produto).
2. **Branch (ramo de trabalho)** — o desenvolvedor cria um ramo para a tarefa. O sistema **renomeia o ramo automaticamente** seguindo o padrão por tipo de tarefa:
   - `feature/…` → nova funcionalidade
   - `bug/…` → correção de erro
   - `task/…` → tarefa técnica
3. **Verificação automática (CI)** — a cada envio de código, o sistema **compila e valida** automaticamente. Se algo quebra, o autor é avisado na hora.
4. **Pull Request (revisão)** — ao propor a junção do trabalho, a tarefa muda sozinha para **"Code Review"** no quadro. **É obrigatória a aprovação de pelo menos 1 revisor** antes de juntar.
5. **Merge (conclusão)** — ao aprovar e juntar, a tarefa vai para **"Finalizado"** e o ramo de trabalho é **excluído automaticamente** para manter o repositório limpo.
6. **Publicação (deploy)** — feita de forma **intencional e controlada** (acionada manualmente), nunca por acidente.

> Os passos 1, 2, 3, 4 e 5 são **100% automáticos**. A equipe foca no desenvolvimento; o controle acontece sozinho.

---

## 4. Ambientes e publicação

Existem **dois ambientes**, hospedados na nuvem AWS (região São Paulo — `sa-east-1`):

| Ambiente | Para que serve | Quem pode publicar |
|---|---|---|
| **Homologação** | Testes e validação antes de ir ao ar | Qualquer desenvolvedor da equipe |
| **Produção** | Ambiente real, usado pelos clientes | **Apenas pessoas autorizadas** (ver seção 6) |

Cada publicação gera uma **imagem versionada** do sistema (guardada no repositório de imagens da AWS, o "ECR"), o que permite **voltar para uma versão anterior (rollback) a qualquer momento** caso algo dê errado.

### Versionamento automático

A cada publicação, a versão do sistema é incrementada automaticamente:

- **Homologação** — gera uma versão de teste rastreável, identificando o número da entrega e a tarefa de origem.
  Exemplo: `1.52.0-B34-F1` (entrega nº 34, referente à funcionalidade da tarefa nº 1).
- **Produção** — gera uma versão "oficial" limpa e cria um **registro de release** no GitHub.
  Exemplo: `1.52.0` → `1.53.0`.

Isso garante que sempre se saiba **exatamente qual versão está em cada ambiente** e **de onde ela veio**.

---

## 5. Gestão e acompanhamento (GitHub Projects)

Todas as tarefas dos sistemas da Tramar são centralizadas num **único quadro de gestão** chamado **"Ebravo Projetos"**, compartilhado entre as organizações Ebravo e Tramar. Isso dá ao gestor **uma visão consolidada** de tudo que está em andamento.

O quadro funciona como um **Kanban** com colunas de status que se movem **automaticamente** conforme o trabalho avança:

```
   Backlog  ─►  (em desenvolvimento)  ─►  Code Review  ─►  Finalizado
```

Cada tarefa também recebe automaticamente um **Épico** (a área/produto a que pertence — ex.: TRAMAR), facilitando o agrupamento e os relatórios.

---

## 6. Segurança e controle de acesso

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
| `mario-pedrao` |

> Qualquer tentativa de publicar em produção por alguém fora dessa lista é **automaticamente bloqueada** pelo sistema. A mesma regra vale para rollbacks (voltar versão).

### Membros da organização

| Usuário | Papel |
|---|---|
| `juniorebravo` | Administrador |
| `murilo-tramar` | Administrador |
| `carloscarvalho-debug` | Membro |
| `gustavo-souza-ebravo` | Membro |
| `mario-pedrao` | Membro |
| `MatheusIsidio` | Membro |
| `sophiaenzo` | Membro |

---

## 7. Configurações técnicas (referência)

Esta seção é uma referência para a equipe técnica. Por segurança, **valores sensíveis (senhas, chaves) nunca ficam expostos** — ficam guardados de forma criptografada nos "segredos" da organização.

### Variáveis da organização

| Variável | Função |
|---|---|
| `DEPLOY_PROD_ALLOWED` | Lista de usuários autorizados a publicar em produção |

### Segredos da organização (valores ocultos)

| Segredo | Função |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Acesso à infraestrutura AWS (imagens dos sistemas) |
| `SSH_KEY_HOMOL` | Chave de acesso ao servidor de homologação |
| `SSH_KEY_PROD` | Chave de acesso ao servidor de produção |
| `GH_TOKEN_BUMP` | Permite o versionamento automático |
| `PROJECT_PAT` | Permite a automação do quadro de gestão (Kanban) |

### Esteira de automação (workflows)

Os repositórios instalam apenas "atalhos" leves que apontam para a esteira central `ebravo-br/cicd-templates`. Os principais são:

| Automação | O que faz |
|---|---|
| `ci` | Compila e valida o código a cada envio |
| `deploy` | Publica em homologação ou produção (acionado manualmente) |
| `rollback` | Volta para uma versão anterior |
| `issue-epic` | Adiciona a tarefa ao quadro e define o Épico |
| `project-automation` | Move a tarefa no Kanban e limpa branches após o merge |
| `branch-naming` | Padroniza o nome dos ramos de trabalho |

---

## 8. Governança e manutenção

- **Centralização**: toda a lógica de automação vive em **um único lugar** (`cicd-templates`). Atualizações são aplicadas a todos os sistemas simultaneamente, reduzindo custo de manutenção e risco de inconsistência.
- **Padrão único entre Ebravo e Tramar**: as duas organizações compartilham a mesma esteira e o mesmo quadro de gestão, o que facilita a operação por uma equipe comum.
- **Evolução planejada**: a estrutura já está preparada para publicar em outras plataformas (ex.: containers gerenciados/Kubernetes) sem reescrever a esteira; hoje a publicação é feita em servidores dedicados na AWS.

---

### Resumo executivo

> A organização Tramar opera com uma **esteira de desenvolvimento padronizada, automatizada e auditável**. Da abertura da tarefa até a publicação, o processo é monitorado e controlado, com **revisão obrigatória de código**, **publicação em produção restrita a pessoas autorizadas** e **capacidade de reverter qualquer versão**. A gestão é acompanhada em tempo real por um quadro Kanban consolidado, exigindo mínima intervenção manual da equipe.

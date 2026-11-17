# Sistema de Gestão de Chamados – Equipe 3

Sistema de help desk para uma **empresa de desenvolvimento de software** (software house) que mantém aplicações web, APIs e bancos de dados para vários clientes. Permite **registrar, classificar, priorizar, encaminhar, acompanhar, escalonar e encerrar chamados**, mantendo o histórico completo de cada atendimento e a rastreabilidade **do usuário até a correção técnica** (release / commit / hotfix).

> Projeto da atividade *Projeto de Sistema de Gestão de Chamados* – **Equipe 3 – Empresa de Desenvolvimento de Software**.

---

## Sumário

1. [Visão geral](#visão-geral)
2. [Decisão tecnológica](#decisão-tecnológica)
3. [Tecnologias](#tecnologias)
4. [Como executar](#como-executar)
5. [Usuários de teste](#usuários-de-teste)
6. [Funcionalidades](#funcionalidades)
7. [Fluxo de atendimento](#fluxo-de-atendimento)
8. [Prioridade e SLA](#prioridade-e-sla)
9. [Encaminhamento e escalonamento](#encaminhamento-e-escalonamento)
10. [Chamados de teste](#chamados-de-teste)
11. [Roteiro de demonstração](#roteiro-de-demonstração)
12. [Atendimento aos requisitos da atividade](#atendimento-aos-requisitos-da-atividade)
13. [Configurações](#configurações)
14. [Estrutura do projeto](#estrutura-do-projeto)
15. [Modelo de dados](#modelo-de-dados)
16. [Testes automatizados](#testes-automatizados)

---

## Visão geral

O suporte da software house recebe **incidentes, bugs, dúvidas e solicitações de melhoria**. Parte desses chamados precisa chegar à equipe de desenvolvimento, por isso o sistema modela três níveis de atendimento:

| Nível | Papel | Escopo |
|---|---|---|
| **N1** – Service Desk | Triagem | Dúvidas, acessos, procedimentos da base de conhecimento, suporte remoto simples |
| **N2** – Suporte técnico / Infra | Análise técnica | Logs, banco de dados, servidores, integrações, configuração |
| **N3** – Especialista / Desenvolvimento | Correção | Bugs de código, melhorias e novas funcionalidades |

## Decisão tecnológica

Foi desenvolvido um **sistema próprio** em vez de configurar o GLPI.

| Critério | Justificativa |
|---|---|
| **Tempo** | Escopo pequeno e conhecido; as regras viram código direto, sem adaptar configurações genéricas. |
| **Complexidade** | O GLPI traz inventário, financeiro, contratos etc., que não seriam usados. O sistema próprio contém só o que o cenário pede. |
| **Manutenção** | Código Java da própria equipe, versionado e testável; qualquer regra muda em um único `Service`. |
| **Flexibilidade** | Regras específicas (release obrigatória no N3, autorização do solicitante para acesso remoto, matriz própria) implementadas exatamente como especificado. |
| **Implantação** | Um único `.jar` (Java 17) + MySQL; banco e massa de teste são criados automaticamente na subida. |

A justificativa completa está em [`docs/DOCUMENTACAO.md`](docs/DOCUMENTACAO.md).

## Tecnologias

- **Java 17** e **Spring Boot 3.3.5**
- Spring Web, Spring Data JPA (Hibernate), Spring Security (login por formulário, BCrypt), Bean Validation
- **Thymeleaf** + **Bootstrap 5** (interface)
- **MySQL 8**
- Maven

## Como executar

### Requisitos
- Java 17+
- Maven 3.8+ (ou uma IDE: IntelliJ, Eclipse, VS Code)
- MySQL 8 rodando em `localhost:3306`

### Passo a passo

```bash
# 1) (opcional) ajuste usuário/senha do MySQL
export DB_USER=root
export DB_PASSWORD=root

# 2) suba a aplicação
mvn spring-boot:run
```

Acesse **http://localhost:8080**.

### Gerar o `.jar`

```bash
mvn clean package
java -jar target/sistema-chamados-1.0.0.jar
```

### Banco de dados

- O banco `chamados_db` é criado automaticamente (`createDatabaseIfNotExist=true`).
- Na subida, o Spring executa `schema.sql` (DDL) e `data.sql` (seed) de `src/main/resources`.
- ⚠️ Com `DB_INIT=always` (padrão) o banco é **recriado a cada inicialização**, ideal para demonstração. Para manter os dados cadastrados, suba com `DB_INIT=never` após a primeira execução.
- Para rodar os scripts manualmente (cópias em `sql/`):
  ```bash
  mysql -u root -p chamados_db < sql/01_schema.sql
  mysql -u root -p chamados_db < sql/02_data.sql
  ```

### Variáveis de ambiente

| Variável | Padrão | Descrição |
|---|---|---|
| `DB_HOST` | `localhost` | Host do MySQL |
| `DB_PORT` | `3306` | Porta |
| `DB_NAME` | `chamados_db` | Nome do banco |
| `DB_USER` | `root` | Usuário |
| `DB_PASSWORD` | *(vazio)* | Senha |
| `DB_INIT` | `always` | `always` recria o banco com o seed; `never` mantém os dados |

## Usuários de teste

Senha de **todos** os usuários: **`123456`**

| Perfil | Nível | Login |
|---|---|---|
| Administrador / Supervisor | – | `supervisor@softhouse.com` |
| Técnico | N1 | `ana.n1@softhouse.com`, `bruno.n1@softhouse.com` |
| Técnico | N2 | `carla.n2@softhouse.com`, `gustavo.n2@softhouse.com` |
| Técnico | N3 (desenvolvimento) | `daniel.n3@softhouse.com`, `eduardo.n3@softhouse.com` |
| Solicitante (9 usuários, 5 empresas-cliente) | – | `joao@` / `maria@alfavarejo.com.br`, `pedro@` / `juliana@betalogistica.com.br`, `lucas@` / `patricia@gamasaude.com.br`, `rafael@` / `camila@deltaeducacao.com.br`, `thiago@epsilonfinanceira.com.br` |

**Total: 15 usuários** (9 solicitantes, 6 técnicos, 1 supervisor).

### Permissões por perfil

| Perfil | O que pode fazer |
|---|---|
| `SOLICITANTE` | Abrir chamados, acompanhar os seus, comentar, autorizar/recusar suporte remoto e validar (aprovar/reprovar) a solução |
| `TECNICO` | Ver a fila do seu nível, assumir, reclassificar, escalar/transferir, atender remotamente, registrar solução, mantém a base de conhecimento |
| `ADMINISTRADOR_SUPERVISOR` | Tudo do técnico em qualquer nível + encerramento direto + dashboard + teste de prazos + horário de atendimento |

## Funcionalidades

- **Autenticação e perfis** com Spring Security (BCrypt) e logout automático por inatividade.
- **Abertura de chamado** com cálculo ao vivo de pontos, prioridade, SLA e nível responsável.
- **Tipos de chamado** diferenciados: Incidente, Bug, Solicitação de Melhoria, Dúvida e Solicitação de Serviço.
- **8 categorias**: Aplicação web, API, Banco de dados, Acesso, Bug, Solicitação de melhoria, Integrações, Infraestrutura de aplicação.
- **Matriz de prioridade** (Impacto × Urgência) com 4 níveis e **SLA** por prioridade.
- **Encaminhamento automático** na abertura e **escalonamento** manual (com justificativa obrigatória) ou **automático** quando o SLA do N1 vence.
- **Transferência para técnico específico**, com aceite ou recusa justificada.
- **Controle de concorrência** (`@Version`): dois técnicos nunca assumem o mesmo chamado.
- **Suporte remoto simulado**: técnico solicita → solicitante autoriza → técnico registra o diagnóstico.
- **Registro de solução** com referência de **versão / release / commit / hotfix** (obrigatória no N3) e vínculo com artigo da base de conhecimento.
- **Validação pelo solicitante** (aprova → `FECHADO`; reprova com motivo → volta ao atendimento).
- **Histórico completo** (linha do tempo) de todas as ações, inclusive as automáticas ("Sistema").
- **Alertas de SLA**: aviso aos 75% do prazo e alerta geral quando o prazo vence (sino, faixa vermelha, etiquetas nas listas).
- **Base de conhecimento** com criação, edição e busca de artigos.
- **Dashboard** com indicadores por status, prioridade, categoria, nível e desempenho.
- **Horário de funcionamento** para abertura de chamados (07:00–20:00), com relógio simulável para demonstração.
- **Teste de prazos** (supervisor): força vencimento de um chamado para demonstrar alertas.

## Fluxo de atendimento

```
Usuário → Abertura → Classificação → Impacto/Urgência → Prioridade
        → N1 → N2 (quando necessário) → N3/Especialista (quando necessário)
        → Solução → Validação → Encerramento
```

Status: `ABERTO` → `EM_ATENDIMENTO` → `SOLUCIONADO` → `FECHADO`
Auxiliares: `AGUARDANDO_USUARIO` e `CANCELADO`.

O fluxograma completo (Mermaid) está em [`docs/DOCUMENTACAO.md`](docs/DOCUMENTACAO.md#5-fluxograma-de-atendimento) e a imagem em [`docs/fluxograma-atendimento.png`](docs/fluxograma-atendimento.png).

![Fluxograma de atendimento](docs/fluxograma-atendimento.png)

## Prioridade e SLA

Pesos: Baixo/Baixa = 1 · Médio/Média = 2 · Alto/Alta = 3 · Crítico/Crítica = 4.
**Pontos = peso do Impacto + peso da Urgência.**

| Impacto \ Urgência | Baixa | Média | Alta | Crítica |
|---|---|---|---|---|
| **Baixo** | Baixa | Baixa | Média | Alta |
| **Médio** | Baixa | Média | Alta | Alta |
| **Alto** | Média | Alta | Alta | Crítica |
| **Crítico** | Alta | Alta | Crítica | Crítica |

| Prioridade | Pontos | Prazo (SLA) de solução |
|---|---|---|
| **Crítica** | 7–8 | 2 h |
| **Alta** | 5–6 | 8 h |
| **Média** | 4 | 24 h |
| **Baixa** | 2–3 | 48 h |

Prazo = `data de abertura + horas do SLA` (tabela `sla`, editável). Se o técnico reclassificar o chamado, prioridade e prazo são recalculados.

## Encaminhamento e escalonamento

**Entrada (calculada na abertura):**

| Tipo | Pontos | Nível inicial |
|---|---|---|
| Solicitação de Melhoria | qualquer | **N3** |
| Dúvida / Solicitação de Serviço | qualquer | **N1** |
| Incidente / Bug | 2 a 6 | **N1** |
| Incidente / Bug | 7 a 8 (crítico) | **N2** |

**Regras de movimentação:**

1. **Transferência manual** com justificativa obrigatória (subir = escalonamento, descer = devolução), para o nível inteiro ou para um técnico específico.
2. **Nível superior assume** chamados livres de níveis inferiores (N2 assume do N1; N3 assume do N1 e N2). O contrário não é permitido.
3. **N1 → N3 direto** quando um bug de código já foi reproduzido com evidência.
4. **Escalonamento automático N1 → N2** a cada 5 minutos para chamados com SLA vencido.
5. **N3 só registra solução** com a referência de versão/release/commit/hotfix preenchida.
6. **Suporte remoto** só é registrado após a autorização do solicitante.

## Chamados de teste

Carregados automaticamente pelo `data.sql`, com histórico:

| # | Chamado | Tipo | Prioridade | Caminho | Status final |
|---|---|---|---|---|---|
| 1 | Erro 500 na aplicação web | Incidente | Alta | N1 | Em Atendimento |
| 2 | API retornando dados incorretos | Bug | Média | N1 → N3 | Em Atendimento |
| 3 | Usuário sem acesso | Solicitação de Serviço | Média | N1 (KB #1) | Fechado |
| 4 | Consulta lenta no banco | Incidente | Alta | N1 → N2 | Solucionado |
| 5 | Bug que afeta um cliente | Bug | Alta | N1 | Aberto |
| 6 | Bug crítico que afeta vários clientes | Bug | Crítica | N2 → N3 | Fechado (Hotfix v3.4.1) |
| 7 | Dúvida funcional | Dúvida | Baixa | N1 (KB #3) | Fechado |
| 8 | Chamado com suporte remoto | Incidente | Média | N1 + remoto | Fechado |
| 9 | Escalonamento para desenvolvimento | Incidente | Alta | N1 → N2 → N3 | Fechado (v2.8.3) |
| 10 | Solicitação de nova funcionalidade | Melhoria | Baixa | N3 direto | Aberto |

Detalhes e evidências de cada um em [`docs/DOCUMENTACAO.md`](docs/DOCUMENTACAO.md#6-os-10-chamados-de-teste-carregados-pelo-datasql).

## Roteiro de demonstração

1. **Solicitante** `joao@alfavarejo.com.br` → *Abrir chamado*. Ao escolher tipo, impacto e urgência aparece o cálculo (pontos, prioridade, SLA e nível). Ex.: Bug Crítico × Crítica = 8 pontos → N2.
2. **Técnico N1** `ana.n1@softhouse.com` → *Minha fila* → assumir → *Escalonar* para N2 com justificativa.
3. **Técnico N2** `carla.n2@softhouse.com` → assumir → escalar para N3.
4. **Técnico N3** `daniel.n3@softhouse.com` → assumir → registrar solução (Release/Commit/Hotfix obrigatório).
5. **Solicitante** → aprova ou reprova a solução → chamado **Fechado**. Todo o caminho aparece no histórico.
6. **Suporte remoto**: técnico *Solicitar acesso* → solicitante **Aceita** ou **Recusa** → técnico registra o diagnóstico.
7. **Supervisor** → *Dashboard*.

Chamados de referência: **#9** (N1→N2→N3), **#6** (crítico com hotfix), **#8** (remoto).

## Atendimento aos requisitos da atividade

### Estrutura mínima

| Requisito | Entrega |
|---|---|
| 8+ solicitantes | 9 solicitantes |
| 4+ técnicos/supervisores | 6 técnicos + 1 supervisor |
| Grupos N1, N2 e N3 | Tabela `grupo` (N1, N2, N3) |
| Perfis separados | `SOLICITANTE`, `TECNICO`, `ADMINISTRADOR_SUPERVISOR` |
| 4 níveis de prioridade | Baixa, Média, Alta, Crítica |
| Regras de encaminhamento/escalonamento | `EncaminhamentoService`, `ChamadoService`, `EscalonamentoAutomaticoJob` |
| Histórico completo | Tabela `historico_chamado` (somente inserção) |
| Base de conhecimento | Módulo `/kb` |
| Dashboard / relatório | `/dashboard` |
| Prazos / SLA | Tabela `sla` + `PrioridadeService` |
| Registro da solução | Campos `solucao` e `referencia_versao` |

### Requisitos específicos da Equipe 3

| Requisito | Entrega |
|---|---|
| Diferenciar incidente, bug e melhoria | Enum `TipoChamado`, com impacto direto no encaminhamento |
| N3 = desenvolvimento/especialista | Grupo N3 – Especialistas / Desenvolvimento |
| Referência de versão/release/correção no encerramento | Obrigatória ao registrar solução no N3 |
| Rastreabilidade usuário → correção técnica | Histórico + campo `referencia_versao` (ex.: chamados #6 e #9) |

### Evidências obrigatórias

| Evidência | Onde ver no sistema |
|---|---|
| Tela inicial | `/login` e `/` |
| Usuários e perfis | Seção [Usuários de teste](#usuários-de-teste) / tabelas `usuario` e `perfil` |
| Grupos / níveis | Tabela `grupo`; coluna "Nível" nas listas |
| Categorias configuradas | Formulário *Abrir chamado* / tabela `categoria` |
| Matriz de prioridade e prazos | Seção [Prioridade e SLA](#prioridade-e-sla) / tela *Abrir chamado* / tabela `sla` |
| Chamados de teste | *Todos os chamados* (supervisor), #1 a #10 |
| Escalonamento N1 → N2 | Chamados **#4** e **#9** |
| Escalonamento N2 → N3 | Chamados **#6** e **#9** |
| Atendimento remoto simulado | Chamado **#8** |
| Solução registrada antes do fechamento | Chamados **#3**, **#6**, **#8**, **#9** (`SOLUCIONADO` → `FECHADO`) |

## Configurações

Principais propriedades em `src/main/resources/application.properties`:

| Propriedade | Padrão | Descrição |
|---|---|---|
| `app.escalonamento-automatico.habilitado` | `true` | Liga o escalonamento automático N1 → N2 |
| `app.escalonamento-automatico.intervalo-ms` | `300000` | Intervalo de verificação (5 min) |
| `app.inatividade.segundos` | `20` | Logout automático por inatividade |
| `app.encaminhamento.pontos-n2` | `7` | Pontos a partir dos quais o chamado entra direto no N2 |
| `app.encaminhamento.pontos-n3` | `9` | Pontos para entrada direta no N3 (9 = desativado) |
| `app.horario.inicio` / `app.horario.fim` | `07:00` / `20:00` | Horário de abertura de chamados |
| `app.horario.simulado` | *(vazio)* | Relógio simulado para demonstração |
| `app.sla.alerta-percentual` | `75` | % do SLA consumido para disparar o alerta |
| `app.sla.verificacao-vencimento-ms` | `60000` | Verificação de SLA vencido (1 min) |

## Estrutura do projeto

```
sistema-chamados
├── docs/
│   ├── DOCUMENTACAO.md              documento técnico completo
│   └── fluxograma-atendimento.png
├── sql/                             cópias de schema e seed para execução manual
├── src/main
│   ├── java/br/com/equipe3/chamados
│   │   ├── model/                   entidades JPA e enums
│   │   ├── repository/              Spring Data JPA
│   │   ├── service/                 regras de negócio (prioridade/SLA, fluxo, jobs, dashboard)
│   │   ├── web/                     controllers e formulários
│   │   └── config/                  Spring Security
│   └── resources
│       ├── templates/               telas Thymeleaf (chamados, kb, dashboard, login…)
│       ├── static/css/app.css
│       ├── schema.sql               DDL MySQL
│       ├── data.sql                 carga inicial
│       └── application.properties
├── src/test/                        testes unitários
└── pom.xml
```

## Modelo de dados

```
perfil (id, nome, descricao)
grupo (id, sigla, nome, descricao, nivel)
categoria (id, nome, descricao)
sla (prioridade PK, horas_solucao)
usuario (id, nome, email UK, senha BCrypt, empresa, perfil_id FK, grupo_id FK, ativo)
artigo_conhecimento (id, titulo, problema, solucao, palavras_chave, categoria_id FK, autor_id FK, ...)
chamado (id, titulo, descricao, tipo, categoria_id FK, impacto, urgencia, prioridade, status,
         solicitante_id FK, grupo_atual_id FK, tecnico_id FK, tecnico_designado_id FK, versao,
         data_abertura, prazo_sla, data_solucao, data_fechamento, solucao, referencia_versao,
         artigo_id FK, suporte_remoto, remoto_autorizado, remoto_ferramenta, remoto_diagnostico)
historico_chamado (id, chamado_id FK, usuario_id FK, data_hora, acao, grupo_origem_id FK,
                   grupo_destino_id FK, status_anterior, status_novo, descricao)
```

## Testes automatizados

```bash
mvn test
```

`PrioridadeServiceTest` valida a matriz de prioridade e o cálculo de prazo por SLA.

---

**Equipe 3 – Empresa de Desenvolvimento de Software**

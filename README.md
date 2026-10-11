## 🫀 Circadia

> Copilot de IA para laudos de **MAPA (Monitorização Ambulatorial da Pressão Arterial)**: importação do exame → cálculos clínicos determinísticos → rascunho do laudo com Claude → revisão do médico → laudo final com rastreabilidade.

![.NET](https://img.shields.io/badge/.NET-10-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![SQL Server](https://img.shields.io/badge/SQL%20Server-EF%20Core-CC2927?style=for-the-badge&logo=microsoftsqlserver&logoColor=white)
![Aspire](https://img.shields.io/badge/.NET%20Aspire-Orquestração-5C2D91?style=for-the-badge)
![Claude](https://img.shields.io/badge/Claude-Anthropic-D97757?style=for-the-badge&logo=anthropic&logoColor=white)

> ⚠️ **POC, não é dispositivo médico.** Todas as regras clínicas estão **pendentes de validação clínica** por um cardiologista. O sistema só gera **rascunhos**; a decisão e a assinatura do laudo são sempre do médico.

---

## 📑 Sumário

- [Sobre o Projeto](#-sobre-o-projeto)
- [Como a IA é usada](#-como-a-ia-é-usada)
- [Arquitetura](#️-arquitetura)
- [Pré-requisitos](#-pré-requisitos)
- [Quick Start](#️-quick-start)
- [API REST](#-api-rest)
- [Testes](#-testes)
- [Configuração](#️-configuração)
- [Roadmap](#️-roadmap)
- [Contribuições](#-contribuições)
- [Documentação adicional](#-documentação-adicional)
- [Autor](#-autor)

---

## 📌 Sobre o Projeto

O **MAPA** mede a pressão arterial ao longo de 24 horas (ou mais) com um manguito no braço ligado a um pequeno aparelho na cintura, que infla automaticamente a cada 20 ou 30 minutos. Serve para diagnosticar hipertensão, avaliar o efeito do tratamento, investigar picos de pressão e observar o comportamento da pressão durante o sono.

O **Circadia** é uma POC que mostra como colocar IA generativa em um fluxo clínico com rigor de engenharia: a matemática do exame é calculada em código testável, e o LLM só redige texto a partir de números já calculados.

O projeto foi gerado a partir do template [Clean Architecture](https://github.com/jasontaylordev/CleanArchitecture) (Jason Taylor) com **React** e **SQL Server**, e é independente dos projetos legados `air-api` e `air2-web`. Dos legados, apenas os padrões foram reaproveitados como referência: Keycloak com RBAC por clínica, laudo versionado, presets e exportação em docx/PDF.

---

## 🧠 Como a IA é usada

Princípio central: **o LLM nunca calcula e nunca decide.**

```text
Leituras (PAS/PAD/FC) ──► Calculadoras do Domain ──► Métricas ──► Claude ──► Rascunho (JSON) ──► Médico revisa ──► Laudo
        │                  (determinísticas,          │            │             │                   │
        │                   100% testadas)            │            │             └─ validado por     └─ aceite/edição
        └─ gerador sintético / importação CSV         │            └─ só recebe métricas              vira métrica de qualidade
                                                      └─ guardadas para auditoria
```

| Capacidade | Descrição | Status |
|------------|-----------|--------|
| 🧮 Cálculos clínicos | Qualidade do exame, médias de 24h/vigília/sono, descenso noturno, carga pressórica, classificação | 📋 Planejado |
| ✍️ Rascunho do laudo | Claude redige a conclusão a partir das métricas, em JSON estruturado | 📋 Planejado |
| 🔎 Triagem de artefatos | Regras e estatística marcam leituras suspeitas; o LLM explica o motivo | 📋 Planejado |
| 📓 Diário do paciente | Extrai medicações e sintomas de texto livre | 📋 Planejado |
| 💬 Copilot de chat | Perguntas em linguagem natural com tools somente leitura, respeitando RBAC | 📋 Planejado |
| ✅ Guardrail numérico | Todo número escrito pelo LLM precisa existir nas métricas calculadas (teste automático no CI) | 📋 Planejado |

---

## 🏗️ Arquitetura

Clean Architecture com dependências apontando para dentro: `Web → Application → Domain`, e `Infrastructure → Application`.

```text
circadia/
├── 📄 circadia.slnx              # Solução
├── 📄 global.json                # SDK .NET fixado
├── 📄 Directory.Packages.props   # Versões centralizadas (NuGet)
├── 📁 docs/                      # ADRs e documentação (a criar)
├── 📁 src/
│   ├── Domain/                   # Entidades e regras clínicas puras (sem dependências)
│   ├── Application/              # Casos de uso (MediatR), validação, portas (interfaces)
│   ├── Infrastructure/           # EF Core + SQL Server, Identity, adapter do Claude
│   ├── Web/                      # API (Minimal APIs) + SPA
│   │   └── ClientApp/            # React 19 + TypeScript + Vite
│   ├── Shared/                   # Nomes de serviços compartilhados
│   ├── ServiceDefaults/          # Observabilidade (OpenTelemetry), health checks, resiliência
│   └── AppHost/                  # Orquestração local com Aspire
└── 📁 tests/
    ├── Domain.UnitTests/
    ├── Application.UnitTests/
    ├── Infrastructure.IntegrationTests/
    ├── Application.FunctionalTests/
    ├── Web.AcceptanceTests/
    └── TestAppHost/
```

**Domínio MAPA (planejado):**

| Entidade | Responsabilidade |
|----------|------------------|
| `AbpmExam` | Paciente, aparelho, intervalo de medição, duração e indicação clínica |
| `Reading` | Horário, PAS, PAD, FC, flag de artefato e período (vigília ou sono) |
| `DiaryEntry` | Horário de dormir e acordar, medicações, sintomas e atividades |
| `AbpmReport` | Métricas, conclusão, status (rascunho, revisado, assinado) e versão |

> O template ainda traz exemplos (`TodoLists`, `TodoItems`, `WeatherForecasts`). Eles serão removidos conforme o domínio de MAPA entrar.

---

## 🔧 Pré-requisitos

| Ferramenta | Versão mínima |
|------------|---------------|
| **.NET SDK** | 10.0 (ver `global.json`) |
| **Node.js** | LTS (para o React/Vite) |
| **Docker Desktop** (ou Podman) | recente. Sobe o SQL Server em contêiner |
| **ANTHROPIC_API_KEY** | [console.anthropic.com](https://console.anthropic.com) (necessária só para as funções de IA) |

---

## ▶️ Quick Start

### 1️⃣ Clonar e compilar

```powershell
cd C:\Projetos\POCs\circadia
dotnet build
```

### 2️⃣ Configurar a chave da Anthropic

Use *user secrets*, para a chave nunca entrar no Git:

```powershell
dotnet user-secrets set "Anthropic:ApiKey" "sua-chave-aqui" --project .\src\Web
```

> 📋 O adapter do Claude ainda é planejado. Sem a chave, o restante da aplicação funciona normalmente.

### 3️⃣ Subir tudo com o Aspire

```powershell
dotnet run --project .\src\AppHost
```

O dashboard do Aspire abre automaticamente com as URLs, logs e traces. A primeira execução baixa a imagem do SQL Server, então demora mais. O contêiner é **persistente**, o que mantém os dados entre execuções.

| Serviço | Onde |
|---------|------|
| Dashboard Aspire | URL exibida no terminal |
| API + referência Scalar | `webapi` → `/scalar` |
| Frontend React | `webfrontend` |
| SQL Server | contêiner gerenciado pelo AppHost (`circadiaDb`) |

### 4️⃣ Gerar exames sintéticos

📋 *Planejado:* um gerador cria perfis de exame (normotenso, *non-dipper*, avental branco etc.) para demonstração e para o golden set dos evals. Nenhum dado real de paciente é usado.

---

## 🌐 API REST

A documentação interativa fica em `/scalar` do serviço `webapi`.

| Método | Endpoint | Descrição | Status |
|--------|----------|-----------|--------|
| `POST` | `/api/abpm-exams/import` | Importa um exame (CSV genérico) | 📋 Planejado |
| `GET` | `/api/abpm-exams/{id}` | Detalhe do exame com leituras e métricas | 📋 Planejado |
| `POST` | `/api/abpm-exams/{id}/report/draft` | Gera o rascunho do laudo com Claude | 📋 Planejado |
| `PUT` | `/api/abpm-exams/{id}/report` | Salva a revisão do médico (nova versão) | 📋 Planejado |
| `POST` | `/api/copilot/ask` | Pergunta em linguagem natural | 📋 Planejado |

> Enquanto isso, os endpoints de exemplo do template (`/api/TodoLists`, `/api/TodoItems`, `/api/WeatherForecasts`) continuam disponíveis.

---

## 🧪 Testes

```powershell
# Tudo
dotnet test

# Só o que roda sem Docker
dotnet test .\tests\Domain.UnitTests
dotnet test .\tests\Application.UnitTests
```

| Projeto | O que cobre | Precisa de Docker? |
|---------|-------------|--------------------|
| `Domain.UnitTests` | Calculadoras e classificação clínica | Não |
| `Application.UnitTests` | Handlers e validações (com LLM falso) | Não |
| `Infrastructure.IntegrationTests` | EF Core contra SQL Server real | Sim |
| `Application.FunctionalTests` | Casos de uso ponta a ponta (Respawn) | Sim |
| `Web.AcceptanceTests` | Fluxos de UI | Sim |

📋 *Planejado:* **gate de eval no CI**, com golden set sintético, checagens determinísticas e verificação de que cada número do texto gerado existe nas métricas. O pipeline falha se a qualidade cair.

---

## ⚙️ Configuração

| Chave | Efeito |
|-------|--------|
| `Anthropic:ApiKey` | Chave da API (use *user secrets*, nunca o Git) |
| `Anthropic:Model` | Modelo do Claude usado no rascunho |
| `Anthropic:Temperature` | `0` = determinístico (recomendado para laudos) |
| `Clinical:Thresholds` | Valores de referência das classificações, **pendentes de validação clínica** |
| `Features:AiDraft` | Liga ou desliga o rascunho por IA (feature flag) |

> 📋 As chaves acima serão criadas junto com os respectivos módulos. Para trocar de provedor de LLM, basta um novo adapter da porta `ILlmClient`, sem mexer em Domain nem Application.

---

## 🗺️ Roadmap

```text
✅ Fase 0 — Esqueleto
   ├── Solução gerada (Clean Architecture + React + SQL Server)
   └── Orquestração local com Aspire

🚧 Fase 1 — Base
   ├── ADR 001 (por que .NET ao lado do Java legado; SQL Server)
   ├── Autenticação Keycloak (JWT) e RBAC por clínica
   ├── CI no GitHub Actions (build, testes)
   └── Remoção dos exemplos do template

🔮 Fase 2 — Domínio e cálculos
   ├── Entidades de MAPA e migrations
   ├── Calculadoras clínicas (regras pendentes de validação)
   ├── Gerador de exames sintéticos
   └── Curva de pressão de 24h (vigília e sono) no React

🔮 Fase 3 — IA e qualidade
   ├── Rascunho do laudo com Claude (saída estruturada)
   ├── Guardrail numérico + golden set + gate de eval no CI
   ├── Telas de revisão lado a lado e painel de qualidade
   └── Métricas de latência, tokens, custo e taxa de aceitação

🔮 Fase 4 — Copilot
   ├── Chat com tools somente leitura (RBAC e clínica)
   ├── Triagem de artefatos e leitura do diário do paciente
   └── Exportação do laudo em PDF e docx
```

---

## 🤝 Contribuições

1. Crie uma branch: `git checkout -b feature/minha-feature`
2. Faça o commit: `git commit -m "feat: adiciona calculadora de descenso noturno"`
3. Faça o push: `git push origin feature/minha-feature`
4. Abra um **Pull Request** 🚀

Para novos casos de uso, o template tem um gerador. Rode a partir de `.\src\Application\`:

```powershell
dotnet new ca-usecase --name GenerateReportDraft --feature-name AbpmReports --usecase-type command --return-type int
```

---

## 📚 Documentação adicional

| Documento | Descrição | Status |
|-----------|-----------|--------|
| 🧭 ADR 001: decisões iniciais | .NET ao lado do Java legado, SQL Server, MAPA como domínio | 📋 A criar |
| 🩺 Regras clínicas do MAPA | Critérios e limiares, com status de validação clínica | 📋 A criar |
| 🧪 Estratégia de evals | Golden set, guardrail numérico e gate no CI | 📋 A criar |

---

## 👤 Autor

Feito com 💙 por **Fábio Muniz**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/fabiomunizdeveloper/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/ntitsolutins01?tab=repositories)

---

**Versão:** 0.1.0 · **Status:** 🚧 Esqueleto gerado, domínio em construção

# 📄 Relatório Técnico: Sistema de Gestão da Vigilância Sanitária (DEVISA/CVISA)

**Departamento de Vigilância Sanitária - DEVISA/CVISA/SESMA**
**Prefeitura Municipal de Belém**

**Data:** 08 de Janeiro de 2026 | **Versão:** 1.0

---

## 📋 Sumário

Este documento apresenta a arquitetura do **Sistema de Gestão de Processos da Vigilância Sanitária de Belém**, visando subsidiar a **solicitação de infraestrutura dedicada (VPS)** à CINBESA, considerando:
- Volume atual de operações
- Potencial de crescimento: **100 mil usuários** (teto teórico considerando todos os públicos)
- Plano de escalabilidade para migração de sistemas terceiros

---

**Índice de Tópicos:**
1. [Visão Geral](#1--visão-geral)
2. [Arquitetura](#2-️-arquitetura)
3. [Fluxos de Processos](#3--fluxos-de-processos)
4. [Regras de Negócio](#4--regras-de-negócio)
5. [Integrações Externas](#5--integrações-externas)
6. [Infraestrutura Atual](#6-️-infraestrutura-atual)
7. [Projeção de Crescimento](#7--projeção-de-crescimento)
8. [Anexos Técnicos](#8--anexos-técnicos)

---

## 1. 🎯 Visão Geral

### 1.1 Objetivo
Automatizar o ciclo completo de licenciamento sanitário, desde a entrada do requerimento até a emissão da licença certificada, sob regime de **segurança em camadas**.

### 1.2 Público-Alvo

| Segmento | Público | Volume Estimado |
| :--- | :--- | :--- |
| **Interno** | Servidores das divisões (DVSA, DVSDM, DVSCEP, DVSE, VISAMB, Casa do Açaí) | ~100 |
| **Licenciamento** | Empresas, contadores e responsáveis técnicos | 80.000+ CNPJs sujeitos |
| **Capacitação** | Manipuladores de alimentos | 10.000+/ano |
| **Fiscalização** | Denunciantes e denunciados (pessoas físicas) | Variável |
| **Regulação** | Prescritores e instituições (médicos, farmacêuticos) | Variável |

### 1.3 Estrutura de Rotas
| Serviço | Função | Rota |
| :--- | :--- | :--- |
| Portal de Processos | Interface principal | `/` |
| API Backend | PocketBase + Banco | `/api` |
| Realtime | Conexões SSE | `/api/realtime` |
| Sistema Alimentos | Sistema legado | `/alimentos` |

---

## 2. 🏗️ Arquitetura

### 2.1 Diagrama
```mermaid
graph TD
    Internet((INTERNET)) -->|HTTPS| CF[CLOUDFLARE<br/>WAF+CDN]
    CF -->|HTTPS| Traefik[TRAEFIK<br/>Reverse Proxy]
    
    subgraph Services [Serviços Containerizados]
        Traefik --> Portal[PORTAL<br/>React+Vite]
        Traefik --> API[API BACKEND<br/>PocketBase+JSVM]
        Traefik --> n8n[n8n<br/>Automação]
        Traefik --> Evolution[EVOLUTION API<br/>WhatsApp]
        Traefik --> Legado[SIST. LEGADO<br/>Capacitação]
    end

    subgraph Data [Persistência]
        API --> SQLite[(SQLite<br/>WAL)]
        API --> Redis[(Redis<br/>Cache/Pub-Sub)]
        API --> Hooks([Hooks JSVM])
        n8n --> PG[(PostgreSQL)]
        Evolution --> PG
        Evolution --> Redis
    end
```

### 2.2 Componentes
| Componente | Tecnologia | Função |
| :--- | :--- | :--- |
| **Portal Principal** | React 19, Vite, TypeScript | Interface principal |
| **Sistema Legado** | React, Vite, TailwindCSS | Sistema de capacitação |
| **API Backend** | PocketBase (Go), SQLite (WAL), Redis | API REST, Auth, Banco, Cache e Realtime |
| **Hooks de Integração** | 25 hooks JSVM (imutáveis) | Validação, gatilhos e webhooks |
| **Automação** | n8n (container isolado) | E-mails transacionais |

### 2.3 Extensibilidade (Hooks JSVM)
O backend estende sua capacidade nativa através de hooks em Engine V8 isolada: rotas públicas customizadas, triggers de banco (webhooks para n8n), cron jobs de manutenção e realtime via SSE + Redis Pub/Sub.

### 2.4 Segurança
**Defesa em camadas:**
1. **Borda (Cloudflare):** WAF, DDoS, rate limiting por rota
2. **Gateway (Traefik/Nginx):** SSL/TLS, headers de segurança, rate limiting
3. **Aplicação:** API Rules por coleção/método, proteção ReDoS (Gatekeeper)
4. **Arquivos:** Tokens de curta duração para downloads protegidos

### 2.5 Otimização
- **Banco:** Índices em campos de busca, views materializadas para dashboards
- **Manutenção:** Cron jobs nativos (VACUUM diário, limpeza de logs, atualização de caches)

---

## 3. 🔄 Fluxos de Processos

### 3.1 Entrada via Portal
```mermaid
graph LR
    User(Usuário) -->|Preenche| Front[Frontend]
    Front -->|API POST| API[API /records]
    API -->|Valida| Triagem[Fila Triagem]
    Triagem -->|Avalia| Decisao{Decisão}
    Decisao -->|Deferido| GDOC[GDOC]
    GDOC -->|Finaliza| Licenca[Licença Emitida]
    Decisao -->|Indeferido| Fim((Finaliza))
    Decisao -->|Pendente| Pend[Notifica Pendência]
```

**Etapas:**
1. Usuário preenche formulário e anexa documentos
2. Pré-validação na interface + API Rules + hooks de integridade
3. Solicitação entra na fila de triagem da divisão
4. Servidor avalia: **Deferido**, **Indeferido** ou **Pendente**
5. Se deferido, cria processo no GDOC e finaliza a solicitação
6. Se indeferido, finaliza a solicitação
7. Se pendente, manda para a fila de pendentes
8. Usuário é notificado por e-mail

### 3.2 Entrada via Webhooks (Formulários Externos)
Rotas `/api/public/webhook/*` recebem payloads externos processados por:
1. **Validação de Autenticidade:** Tokens nos headers
2. **Sanitização:** Remoção de dados redundantes
3. **Gatekeeper:** Proteção ReDoS
4. **Extração de Metadados:** Mapeamento dinâmico

### 3.3 Sistema de Comunicação

#### 3.3.1 Atual: E-mail via Automação
Sem autenticação pública, e-mail é o canal principal. Todos os e-mails transacionais são disparados via **webhooks para n8n**:

| Momento | Gatilho | Conteúdo |
| :--- | :--- | :--- |
| **Solicitação** | Webhook | Confirmação + protocolo |
| **Triagem** | Webhook | Status (Deferido/Indeferido/Pendente) |
| **Licença** | Webhook | PDF anexo + QR Code |

#### 3.3.2 Futuro: E-mail como Canal Auxiliar
Com autenticação pública:

| Aspecto | Atual | Futuro |
| :--- | :--- | :--- |
| **Canal Principal** | E-mail | Conta do usuário |
| **Canal Secundário** | - | E-mail + WhatsApp |
| **Status** | Via e-mail | Dashboard + SSE |
| **Licença** | Anexo | Área do usuário |

Automação: n8n (atual) → *WhatsApp Business API* (futuro)

### 3.4 Emissão de DAM

#### 3.4.1 Motivo do Uso de DAM Avulso
A Vigilância Sanitária emite exclusivamente **DAM Avulso** devido à defasagem nos valores tipificados do SIAT. A empresa terceirizada responsável pelo sistema não atualiza os valores das taxas há mais de uma década, enquanto o Decreto Municipal reajusta os valores anualmente pelo **IPCA-E**. Como resultado, os DAMs tipificados apresentam valores incorretos, forçando a VISA a utilizar o DAM Avulso para garantir a cobrança conforme legislação vigente.

#### 3.4.2 Gargalo Operacional
A emissão de DAM Avulso no SIAT exige:
- Autenticação em `http://siat.belem.pa.gov.br/`
- Preenchimento manual de **16 campos** (JSF/PrimeFaces)
- Aguardar renderização AJAX entre campos interdependentes
- Registrar número do DAM no sistema DEVISA/CVISA

**Métricas:**
- **Tempo médio:** 6-8 min/DAM (pior caso)
- **Volume (2025):** 8.000+ solicitações
- **Impacto:** ~933 horas/ano em tarefa passível de automação

#### 3.4.3 Problema: Duplicidade de DAMs
Empresas podem pagar até 3x o valor devido quando:
1. Emitem DAM via REGIN
2. Trocam de contador
3. Novo contador solicita novo DAM à DEVISA/CVISA

Não há verificação automatizada de DAM existente no exercício.

---

## 4. 📜 Regras de Negócio

### 4.1 Classificação por CNAE
| Divisão | Sigla | Escopo | CNAEs |
| :--- | :--- | :--- | :--- |
| Alimentos | DVSA/GALE | Restaurantes, mercados, açaiterias | ~100 |
| Drogas e Medicamentos | DVSDM/GVDM | Farmácias, distribuidoras | ~50 |
| Saúde | DVSCEP/GVSS | Hospitais, clínicas, estética | ~100 |
| Estabelecimentos | DVSE/GALE | Escolas, hotéis, lavanderias | ~76 |
| Casa do Açaí | Açaí | Batedores artesanais | Específico |
| Vigilância Ambiental | VISAMB | Água para consumo humano | N/A |

### 4.2 Tipos de Processos
**Licenciamento:** Primeira Licença e Renovação

**Diversos (25 tipos):** 2ª via, autenticação de livro, parecer de evento, aprovação de projeto, relatório AFE, desinterdição, inutilização, encerramento, etc.

**Taxas (DAM):** De quase todos os tipos de processos existentes

### 4.3 Isenções
MEI, Órgão Público e Serviço de Utilidade Pública são isentos de taxa, mas não são isentos de fiscalização.

### 4.4 Documentos Obrigatórios
Lista dinâmica conforme: tipo de processo, natureza jurídica, categoria de atividade e se primeira licença ou renovação.

### 4.5 Workflow de Triagem
```mermaid
graph TD
    Pendente((Pendente)) -->|Análise| Servidor{Servidor}
    Servidor -->|Aprova| Deferido[Deferido]
    Deferido --> GDOC[GDOC]
    GDOC --> Licenca[Licença Emitida]
    
    Servidor -->|Rejeita| Indeferido[Indeferido]
    Indeferido -.->|Email| User[Usuário]
    
    Servidor -->|Solicita Correção| Pendencia[Pendente]
    Pendencia -.->|Email| User
```

> **Cada transição de status dispara e-mail automático ao solicitante**, garantindo transparência durante todo o ciclo do processo.

---

## 5. 🔗 Integrações Externas

| Sistema | Função | Integração |
| :--- | :--- | :--- |
| **GDOC** (gdoc.dominioprefeitura) | Gestão de processos | Manual |
| **SIAT** (siat.dominioprefeitura) | Emissão de DAMs | Manual (JSF) |
| **Certifica** (dentro do Portal do Servidor) | Assinatura digital | Manual |

**Sobre o SIAT:** Desenvolvido por terceirizada. Banco de dados pertence à SEFIN. Responsabilidade técnica da CINBESA. Solicitações de integração devem ser direcionadas à CINBESA.

### 5.1 Proposta: Integração para DAM

Para consulta/validação/prevenção de duplicidades, pode haver consumo de views. Para emissão (gravação), pode-se integrar via REST, SOAP, stored procedures, troca de arquivos... conforme disponibilidade da CINBESA.

#### 5.1.1 Requisitos (Modelo REST)

**Rotas de Escrita (se disponível):**
| Método | Endpoint | Função |
| :--- | :--- | :--- |
| `POST` | `/api/v1/dam/avulso` | Emissão individual |
| `POST` | `/api/v1/dam/lote` | Emissão em lote |

**Rotas de Consulta (Views):**
| Método | Endpoint | Função |
| :--- | :--- | :--- |
| `GET` | `/api/v1/dam/{numero}` | Consulta DAM específico |
| `GET` | `/api/v1/dam/contribuinte/{cnpj}` | DAMs de um contribuinte |
| `GET` | `/api/v1/dam/visa` | DAMs da Vigilância Sanitária (cod. 307) |

**Parâmetros de Filtro:**
| Param | Tipo | Descrição |
| :--- | :--- | :--- |
| `data_emissao_inicio` | `YYYY-MM-DD` | Filtrar por data de emissão (início) |
| `data_emissao_fim` | `YYYY-MM-DD` | Filtrar por data de emissão (fim) |
| `situacao` | `0` ou `1` | 0 = Emitido, 1 = Baixado |
| `page` | `int` | Página (paginação) |
| `limit` | `int` | Registros por página |

**Resposta (dados brutos para agregação local):**
```json
{
  "total": 8423,
  "page": 1,
  "limit": 100,
  "items": [
    {
      "num_documento_dam": "20260001234567",
      "num_identificacao": "19769575000100",
      "valor_total": 984.94,
      "data_emissao": "2026-01-15",
      "situacao_pagamento": "1",
      "data_pagamento": "2026-01-20"
    }
  ]
}
```

> **Nota:** A agregação (totais, somas) pode ser feita pelo sistema após consumo da view.

#### 5.1.2 Payload de Emissão (Exemplo)
```json
{
  "tipo_pessoa": "PJ",
  "cnp": "19769575000100",
  "razao_social": "NOVA MEDICA COMÉRCIO LTDA",
  "unidade": "SESMA",
  "tributo": 509,
  "receita": 1614,
  "valor_principal": 984.94,
  "data_vencimento": 1770587283,
  "informacoes_adicionais": "Protocolo: 2E9yr9e. CNAE: 4645-1/01. Exercício 2026."
}
```

#### 5.1.3 Casos de Uso
1. **Emissão Avulsa:** Servidor seleciona contribuinte e taxa; sistema retorna DAM avulso instantaneamente
2. **Emissão em Lote:** Renovações automáticas para empresas com licença a vencer (exceto isentos)
3. **Prevenção de Duplicidade:** Consulta prévia ao contribuinte antes de emitir

**Economia estimada:** ~933 horas/ano

### 5.2 Proposta: View de Empresas Sujeitas à VISA

View no sistema fazendário filtrando empresas ativas em Belém com CNAE sujeito à Vigilância Sanitária.

> **Nota:** Estrutura baseada no layout público da Receita Federal. O esquema real pode diferir.

**Tabela: Empresas**
| Campo | Tipo | Descrição |
| :--- | :--- | :--- |
| `cnpj_basico` | VARCHAR(8) | Raiz do CNPJ |
| `razao_social` | VARCHAR(200) | Nome empresarial |
| `natureza_juridica` | VARCHAR(4) | Código |
| `porte_empresa` | VARCHAR(2) | ME, EPP, Demais |
| `capital_social` | DECIMAL(15,2) | Capital declarado |
| `opcao_simples` | CHAR(1) | Se optante Simples ('S'/'N') |
| `data_opcao_simples` | DATE | Data de opção pelo Simples |
| `data_exclusao_simples` | DATE | Data de exclusão do Simples |
| `opcao_mei` | CHAR(1) | Se optante MEI ('S'/'N') |
| `data_opcao_mei` | DATE | Data de opção pelo MEI |
| `data_exclusao_mei` | DATE | Data de exclusão do MEI |

> **Identificação de MEI ativo:** `opcao_mei = 'S'` **E** (`data_exclusao_mei` é `NULL` ou futura)

**Tabela: Estabelecimentos**
| Campo | Tipo | Descrição |
| :--- | :--- | :--- |
| `cnpj` | VARCHAR(14) | CNPJ completo |
| `situacao_cadastral` | VARCHAR(2) | 02=Ativa |
| `cnae_fiscal_principal` | VARCHAR(7) | CNAE principal |
| `cnae_fiscal_secundaria` | TEXT | CNAEs secundários |
| `bairro` | VARCHAR(100) | Bairro |
| `municipio` | VARCHAR(100) | Município |
| `uf` | CHAR(2) | UF |

**Tabela: DAMs Emitidos**
| Campo | Tipo | Descrição |
| :--- | :--- | :--- |
| `num_documento_dam` | VARCHAR(20) | PK |
| `codigo_barras` | VARCHAR(48) | Linha digitável |
| `valor_total` | DECIMAL(10,2) | Valor |
| `num_identificacao` | VARCHAR(20) | CNPJ/CPF |
| `situacao_pagamento` | VARCHAR(100) | Status ("0 - Emitido", "1 - Baixado") |
| `data_pagamento` | DATE | Data da baixa |

**Filtros:**
- `situacao_cadastral = '02'` (Ativa)
- `uf = 'PA'` e `municipio = 'BELEM'`
- CNAE principal ou secundário na lista VISA

**Benefícios:**
1. Emissão proativa antes do vencimento
2. Prevenção de DAMs duplicados
3. Identificação automática de isentos (MEIs)
4. Base sincronizada com Receita Federal

### 5.3 Plano de Internalização (Longo Prazo)

**Objetivo:** Centralizar todos os fluxos dentro do sistema, eliminando dependências externas.

| Atual | Futuro |
| :--- | :--- |
| Comunicação via e-mail | Notificações no sistema + e-mail auxiliar |
| Formulários externos e nativos | Processos agrupados por usuário |
| Emissão manual de DAM (SIAT) | Integração via Web Service |
| Despachos no GDOC | Tramitação interna com workflow |

**Módulos planejados:**
- **Autenticação Pública:** Conta do usuário para acompanhamento e histórico
- **Módulo Financeiro:** Integração SIAT para DAMs
- **Protocolo e Tramitação:** Substituição do GDOC (despachos internos)
- **Instrumentos Fiscais:** Emissão de termos (intimação, apreensão, etc.)
- **Verificação de Autenticidade:** Validação nativa via QR Code (já existente, carece de domínio oficial)

#### 5.3.1 Fluxo Futuro de Tramitação
```mermaid
graph TD
    User(Usuário) -->|Solicita Licenciamento| Triagem(Secretaria/Triagem)
    Triagem -->|Despacho| Chefia(Chefia do Setor)
    Chefia -->|Despacho| Fiscal(Fiscal/Inspeção)
    
    Fiscal --> Decisao{Adequado?}
    
    Decisao -->|Sim| Libera[Libera p/ Licença]
    Libera --> Valida(Chefia Valida)
    Valida --> Emite[Licença Emitida]
    Emite --> Entrega(Entrega ao Usuário)
    
    Decisao -->|Não| Inst[Emite Instrumento<br/>Termo de Intimação]
    Inst -->|Prazo| Adequa(Empresa se Adequa)
    Adequa --> Fiscal
```

**Benefícios:**
- Rastreabilidade completa de despachos
- Prazos controlados por workflow
- Instrumentos fiscais digitais
- Usuário acompanha status em tempo real

---

## 6. 🖥️ Infraestrutura Atual

### 6.1 VPS Atual
| Recurso | Especificação | Uso |
| :--- | :--- | :--- |
| SO | Ubuntu 24.04 LTS | - |
| RAM | 8 GB | ~35% |
| vCPUs | 3 Cores | Load: 0.47 |
| SSD (SATA) | 145 GB | 10% (14 GB) |
| Uptime | 125 dias | - |
| Produção | 05/01/2026 | 3 dias |

### 6.2 Consumo por Container
| Serviço | Memória | Função |
| :--- | :--- | :--- |
| *PocketBase* | 470.5 MB | API + Banco + Auth |
| Frontend | 5.8 MB | SPA estático |
| *n8n* | 600.0 MB | Orquestração |
| *Redis* | 9.7 MB | Filas |
| *PostgreSQL* | 100.4 MB | Banco de Dados (n8n + Evolution) |
| *Traefik* | 99.6 MB | Proxy reverso |
| *Easypanel* | 251.0 MB | Gerenciamento |

### 6.3 Volumes Persistentes
| Volume | Conteúdo |
| :--- | :--- |
| **Dados** | Banco SQLite + Anexos (uploads) |
| **Migrações** | Scripts de evolução do schema |

> Hooks de integração estão embarcados na imagem Docker (imutáveis em runtime).

### 6.4 Backup (Planejado)
- **Destino:** E2 Cloud Object Storage (S3)
- **Frequência:** Snapshots incrementais diários
- **Recuperação:** Restore nativo do PocketBase

---

## 7. 📈 Projeção de Crescimento

### 7.1 Base de Usuários

**Volume Atual de Operações:**
| Indicador | Volume | Observação |
| :--- | :--- | :--- |
| Empresas licenciadas | ~5.000 | Licenças ativas |
| Processos de licenciamento | ~6.000/ano | Primeira licença + renovações |
| Manipuladores de alimentos | 10.000+/ano | Capacitação e certificação |
| CNPJs sujeitos à VISA | 80.000+ | Potencial de licenciamento |

**Projeção de Usuários Autenticados:**
| Cenário | Usuários | Composição |
| :--- | :--- | :--- |
| Atual | ~100 | Servidores internos |
| Curto Prazo | 5.000-10.000 | + Contadores, RT, manipuladores |
| Médio Prazo | 20.000-30.000 | + Empresas licenciadas |
| Potencial Máximo | 100.000+ | Todos os públicos (teto teórico) |

### 7.2 Funcionalidades Planejadas
| Recurso | Impacto |
| :--- | :--- |
| Autenticação externa | Conexões simultâneas |
| WhatsApp Business | Novo container + filas |
| Chatbot com IA | Processamento NLP |
| **Migração PostgreSQL** | Escala horizontal |

**Escalabilidade:**
- **Atual:** Vertical (Go + SQLite monolítico)
- **Futuro:** Horizontal (Elixir + PostgreSQL distribuído - ex. Supabase)

### 7.3 Proposta de Observabilidade (Sugestão)
O sistema conta com **logging nativo** de requisições e erros (armazenado internamente). Caso a infraestrutura de destino não possua monitoramento centralizado, adiciona-se nota de ferramenta complementar para:
- **Logs:** Externalização para maior segurança e retenção.
- **Uptime:** Monitoramento ativo da disponibilidade da API e integridade dos backups.
- **Alertas:** Notificação proativa da equipe técnica em caso de anomalias.

### 7.4 Recursos Necessários
| Cenário | RAM | vCPUs | SSD | Justificativa |
| :--- | :--- | :--- | :--- | :--- |
| Atual | 8 GB | 3 | 145 GB | Operação estável (~35% uso) |
| 1 ano | 16 GB | 4 | 200 GB | Autenticação pública + WhatsApp + 10k usuários |
| 3 anos | 16-32 GB | 4-6 | 300 GB | 20-30k usuários + integrações |

> **Nota:** Projeções baseadas em crescimento realista. O potencial máximo (100k+ usuários) exigiria migração para arquitetura distribuída (PostgreSQL), avaliada conforme demanda.

---

## 8. 📎 Anexos Técnicos

### 8.1 Stack
**Frontend:** *React* 19, *Vite* 7, *TypeScript* 5.9, *TailwindCSS*, *Lucide*, *ECharts*, *React Hook Form*

**Backend:** *PocketBase* 0.28+ (Go), SQLite 3 (WAL) + Redis cache, *Nginx*, *Supervisord*

**Automação:** *n8n*, *Redis*, *PostgreSQL*

**DevOps:** *Docker Swarm*, *Easypanel*, *Traefik*, *Cloudflare* (CDN+WAF)

### 8.2 Webhooks Ativos
| Rota | Método | Função |
| :--- | :--- | :--- |
| `/api/public/webhook/processos` | POST | Formulários de licenciamento |
| `/api/public/webhook/taxas` | POST | Solicitações de taxa |
| `/api/realtime` | GET (SSE) | Tempo real |

### 8.3 Coleções Principais
| Coleção | Registros | Função |
| :--- | :--- | :--- |
| `w_processos_visa` | Milhares | Solicitações via sistema |
| `w_processos_externos` | Milhares | Solicitações externas |
| `w_taxas_visa` | Milhares | Solicitações de taxa |
| `w_triagem_visa` | Milhares | Triagens |
| `w_licencas` | ~50.000+ | Licenças emitidas |
| `w_empresas` | ~100.000 | CNPJs Ativos de Belém - amostra de dez/25|
| `w_estabelecimentos` | ~100.000 | Estabelecimentos |

---

*Documento gerado em 08/01/2026. Sujeito a atualizações.*

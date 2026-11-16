# Contexto do projeto — Alaga Alerta · Secure DevOps & Cloud (Produção)

> Documento de contexto para o agente de DevSecOps. Leia tudo antes de criar ou alterar qualquer arquivo. Idioma dos entregáveis (README, comentários, mensagens de log, documentação): **português**. Conversa com o usuário: ele escreve em português e prefere respostas em inglês no chat.

## 1. O produto

**Alaga Alerta** avisa moradores e motoristas, por bairro e em tempo útil, quando uma rua está prestes a alagar e o que fazer.

- Cidadão e território: moradores e motoristas de bairros com histórico de alagamento (exemplo: Vila Maria, São Paulo).
- Problema: os alertas de chuva são genéricos e chegam tarde. Falta um dado local, em tempo real, de quanto a água subiu e com que velocidade.
- ODS: 11 e 13 (primários), 3 e 9 (secundários).
- Contexto acadêmico: FIAP Global Solution 2026.2, tema CitizenTech, Engenharia de Software 4º ano. Um único MVP integra sete disciplinas; esta tarefa cobre a **Produção** (disciplina 6 de 7).

Arquitetura do produto (cada disciplina entrega um componente e todos conversam por uma **API central**):

```
Fontes de dados → Big Data (Bronze/Prata/Ouro) → API central ← LLM local · Visão · RPA · Sensor ESP32
                                                      │
                                                 Front do MVP (mapa do bairro)
DevSecOps publica e monitora tudo · BI demonstra no pitch
```

| Componente | O que entrega |
|---|---|
| Sensor físico (Maker Lab) | ESP32 + HC-SR04 (nível), MQ-2 (gás), DHT22; IA embarcada; `POST /leituras` na API central |
| Dados | Airflow + Databricks, Bronze/Prata/Ouro; o Ouro alimenta o mapa |
| IA generativa | LLM local com RAG, JSON válido e citações |
| Visão | Detecção de objetos para o problema de alagamento |
| RPA | Fluxo que usa `classe == "PERIGO"` como gatilho do aviso |
| Produção (**esta tarefa**) | Containers, Terraform, GitHub Actions, observabilidade |

> **A stack e o repositório do MVP (API, front) são de outros integrantes.** Antes de escrever o Dockerfile, o agente deve perguntar ou ler o repositório para saber a linguagem, o framework, as portas e as variáveis de ambiente. Não presumir.

## 2. O que a disciplina exige (fonte: PDF do desafio, "Deploy Blindado", 14 pontos)

Professor: Gabriel Henrique Boyczuk Arcanjo. **Objetivo:** colocar o MVP no ar por um pipeline DevSecOps de ponta a ponta: build, Gitleaks e Semgrep, deploy automático em infraestrutura criada por código e observabilidade com dashboard e alerta.

**Entregáveis**
1. Repositório público com Dockerfile, workflow e Terraform.
2. URL pública do MVP.
3. PDF com arquitetura, evidências (prints) e custo.
4. Ficha Técnica do MVP no README.

**Formato e prazo:** PDF único com links clicáveis, no portal da disciplina, até **19/11/2026 às 23h55**, um envio por equipe. Confirmar o horário no portal.

**Critérios de avaliação**

| Critério | Pontos | O que o PDF pede |
|---|---:|---|
| Container e IaC | 5,0 | Dockerfile enxuto, **usuário não root**, sem segredo na imagem, stack subindo com `docker compose` (2,0). Nuvem criada por Terraform, **state e `.tfvars` fora do repo**, **porta admin nunca aberta para `0.0.0.0/0`** (3,0) |
| Pipeline DevSecOps | 4,0 | GitHub Actions com **build, Gitleaks, Semgrep e Trivy a cada push**, segredos em **GitHub Secrets** e **deploy automático**. **Mostrar uma execução barrada e a corrigida** |
| Observabilidade | 3,0 | Dashboard com as **4 métricas de ouro** (Prometheus e Grafana, Application Insights ou CloudWatch) e **um alerta disparado**. Prometheus e Grafana podem rodar local apontando para o MVP publicado |
| Documentação | 2,0 | Diagrama da arquitetura, responsabilidade compartilhada, **onde ficam os dados pessoais (LGPD)**, estimativa de custo e README reproduzível |

As 4 métricas de ouro: latência, tráfego, erros e saturação.

## 3. Regras do agente

1. **Não inventar** resultados, prints, custos, URLs, métricas ou execuções. Onde faltar informação, deixar um placeholder claro e avisar.
2. **Simples e viável.** Prazo curto. Sem Kubernetes, microsserviços extras ou nuvem cara sem justificativa. Preferir uma VM pequena ou serviço gerenciado barato, com `docker compose`.
3. **Segredos nunca no git** (nem em imagem, `.tfvars`, state ou histórico). Chaves vão para GitHub Secrets. `dev-key-demo` do sensor é só placeholder de demonstração.
4. **Provar a barreira:** gerar uma execução do pipeline que falha de propósito (por exemplo, um segredo falso detectado pelo Gitleaks ou uma vulnerabilidade apontada pelo Trivy) e a execução corrigida, com links e prints reais feitos pelo usuário.
5. **Não concordar automaticamente.** Se uma decisão ameaçar a entrega de outra disciplina ou o prazo, sinalizar na hora.
6. **Conferir antes de afirmar.** O usuário corrige quando recebe informação errada.
7. Entregar arquivos completos e prontos para uso, não trechos soltos.

## 4. O que o agente deve produzir

- `Dockerfile` por serviço (multi-stage, imagem enxuta, `USER` não root, sem segredo, `HEALTHCHECK`) e `.dockerignore`.
- `docker-compose.yml` com o MVP, o banco (se houver) e, opcionalmente, Prometheus e Grafana. Variáveis em `.env.example`; o `.env` real fica no `.gitignore`.
- `infra/` em Terraform: rede, VM ou serviço, security group com só as portas necessárias (80/443; SSH restrito ao IP do dono, **nunca** `0.0.0.0/0` em porta admin). `*.tfstate`, `*.tfvars` e `.terraform/` no `.gitignore`; variáveis sensíveis marcadas `sensitive = true`.
- `.github/workflows/ci.yml`: build, **Gitleaks**, **Semgrep**, **Trivy** (imagem e `fs`) a cada push; falha o job em achado alto ou crítico. `deploy.yml` (ou job final) com deploy automático após as verificações, usando GitHub Secrets.
- Observabilidade: endpoint `/metrics` (ou `/health`) no MVP, `prometheus.yml`, dashboard Grafana (JSON) com as 4 métricas de ouro e uma regra de alerta que dispare de verdade (por exemplo, taxa de erro ou latência alta). Para o alerta do sensor, usar a classe `PERIGO` ou a ausência de leituras (sensor sem heartbeat por mais de 60 s) como exemplo de regra de negócio.
- Documentação: diagrama da arquitetura, **responsabilidade compartilhada** (o que é da nuvem e o que é da equipe), **mapa dos dados pessoais** (LGPD), estimativa de custo mensal, README reproduzível (do clone ao deploy) e a **Ficha Técnica do MVP**.
- `.gitignore`: `.env`, `*.tfstate*`, `*.tfvars`, `.terraform/`, `secrets.h`, `__pycache__/`.

## 5. Pontos de integração relevantes para a Produção

**Endpoint do sensor (proposta da disciplina Física; fechar com o time da API):** `POST /leituras`, cabeçalhos `Content-Type: application/json`, `X-Device-Id`, `X-Device-Key`.

```json
{
  "sensor_id": "alaga-sensor-01",
  "event_id": "alaga-sensor-01-12",
  "ts": "2026-11-13T21:15:03Z",
  "tipo": "alerta",
  "bairro": "Vila Maria",
  "lat": -23.52140, "lon": -46.58780,
  "classe": "PERIGO",
  "nivel_cm": 112.0, "nivel_pct": 74.7, "taxa_pct_min": 9.3,
  "gas_pct": 12, "gas_alto": false, "motivo": "nivel",
  "janelas_confirmadas": 3,
  "temperatura_c": 22.1, "umidade_pct": 88,
  "fonte": "sensor"
}
```

- `tipo`: `alerta` (ATENCAO ou PERIGO confirmado) ou `leitura` (rotina a cada 30 s com tudo NORMAL).
- `event_id` é igual no reenvio: a API deve **descartar duplicados** (idempotência).
- O dispositivo usa HTTPS; no simulador usa `setInsecure()` só para demonstração. A URL pública do MVP precisa ter **certificado válido**.
- A autenticação do sensor é por `X-Device-Key`: guardar a chave real em GitHub Secrets e no ambiente, nunca no código.

**Dados pessoais (LGPD):** o sensor não envia dado pessoal (só nível, gás, temperatura, umidade, hora e posição do ponto de instalação). Dados pessoais, se existirem, ficam no cadastro de usuários do MVP (alertas por bairro); o agente deve **perguntar ao time do MVP** quais dados são coletados e onde ficam antes de documentar.

## 6. Identidade visual (dashboards, diagramas e documentos)

Paleta usada no deck do projeto. Usar a mesma nos dashboards do Grafana, nos diagramas e no PDF, para a entrega parecer um produto só.

| Uso | Cor | Hex |
|---|---|---|
| Fundo escuro (azul petróleo) | | `#10283B` |
| Painel/cartão escuro | | `#16374F` |
| Borda/realce escuro | | `#2B5C78` |
| **Destaque (ciano água)** | | `#4FD1E0` |
| Texto sobre fundo escuro | | `#E8F1F4` |
| Texto secundário sobre escuro | | `#B5CBD3` |
| Fundo claro | | `#F2F6F7` |
| Texto sobre fundo claro | | `#14293A` |
| Primária sobre claro (títulos de tabela) | | `#0B6E80` |
| Linha/borda clara | | `#D3E1E6` |

Cores de estado (usar em LEDs, alertas e painéis):

| Estado | Cor | Hex |
|---|---|---|
| NORMAL | verde | `#3FB58A` |
| ATENCAO | âmbar | `#F5B041` |
| PERIGO | vermelho | `#C0392B` |

Tipografia: **Space Grotesk** (títulos) e **Inter** (texto); código em fonte monoespaçada. Tom: direto, em português, sem jargão desnecessário. Toda tela de monitoramento deve usar verde, âmbar e vermelho **sempre acompanhados de texto** (NORMAL, ATENCAO, PERIGO), não só cor.

## 7. Informações pendentes (perguntar ao usuário antes de assumir)

- Stack, portas e variáveis de ambiente da API central e do front (repositório do MVP).
- Provedor de nuvem e orçamento (AWS, Azure ou GCP; conta de estudante?).
- Domínio e certificado (ou o endereço fornecido pelo provedor).
- Quais dados pessoais o MVP coleta e onde ficam.
- Nomes completos e RMs dos integrantes para o README e o PDF: **Manoella Herrerias Waideman (98906), Felipe Capriotti da Silva Santos (98460), Victor Hugo Andrade (550996), Gustavo Kawamura Christofani (99679)**.
- Links: repositório principal, URL pública do MVP, vídeo do pitch.
- Quem envia o PDF no portal (um envio por equipe).

## 8. Checklist final

- [ ] Repositório público, sem nenhum segredo (rodar `gitleaks detect` no histórico inteiro).
- [ ] `docker compose up` sobe a stack do zero, com usuário não root.
- [ ] `terraform plan` e `apply` funcionam; state e `.tfvars` fora do repo; nenhuma porta admin aberta para `0.0.0.0/0`.
- [ ] Pipeline com build, Gitleaks, Semgrep e Trivy; uma execução barrada e uma corrigida, ambas com link.
- [ ] Deploy automático funcionando e URL pública respondendo.
- [ ] Dashboard com as 4 métricas de ouro e um alerta efetivamente disparado (print).
- [ ] Documentação: diagrama, responsabilidade compartilhada, LGPD, custo, README reproduzível, Ficha Técnica.
- [ ] PDF único com links clicáveis, enviado até 19/11/2026, 23h55.
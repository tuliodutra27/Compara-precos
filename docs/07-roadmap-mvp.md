# 07 — Roadmap do MVP

Premissa: **1 desenvolvedor, meio período** (~15–20 h/semana). Com dedicação integral,
divida os prazos por dois. Sprints de 1 semana.

## Sprint 0 — Validação técnica (1 semana) — *bloqueante*

Nada mais começa antes disso. O objetivo é responder: **existe dado utilizável?**

| # | Tarefa | Entrega |
|---|---|---|
| S0-1 | Interceptar o tráfego do app Menor Preço Brasil com mitmproxy e mapear o endpoint de busca por GTIN | `docs/adr/001-endpoints-menor-preco-brasil.md` com curl reprodutível |
| S0-2 | Testar Preço da Hora BA (handshake de cookie/CSRF + busca por GTIN) | script `spikes/preco_hora_ba.py` funcionando |
| S0-3 | Testar `menorpreco.notaparana.pr.gov.br/api/v1/produtos` | script `spikes/nota_parana.py` funcionando |
| S0-4 | Ler os termos de uso dos portais testados | seção preenchida no doc 08 |
| S0-5 | Medir cobertura: 30 GTINs reais da sua despensa × 3 regiões | planilha com % de acerto e nº médio de lojas |

**Critério de saída:** ao menos uma fonte devolve, para ≥ 60% dos GTINs testados,
**5 ou mais lojas** dentro de 10 km. Se falhar → o projeto vira "base colaborativa
primeiro" e o roadmap muda (a fase 2 vira fase 1).

**Decisão da Sprint 0:** qual UF é a de lançamento.

## Sprint 1 — Backend: núcleo

- Projeto FastAPI + docker-compose (Postgres/PostGIS + Redis).
- Migrations do doc [04](04-modelo-de-dados.md).
- Interface `FontePrecos` + **o primeiro adapter** (a fonte vencedora da Sprint 0).
- `GET /v1/precos` com `lat`/`lon`/`raio_km`, gravando as ofertas no banco.
- **DoD:** `curl` local devolve o JSON do doc [05](05-contrato-api.md) para um GTIN real.

## Sprint 2 — Backend: preço justo, cache e localidades

- `services/preco_justo.py` conforme doc [06](06-algoritmo-preco-justo.md) + testes unitários.
- Cache Redis (L2) e fallback para o Postgres quando a fonte falha.
- `/v1/localidades/*` (proxy IBGE) e filtro por `codigo_ibge`.
- Rate limiting, envelope de erro, Sentry, `/v1/health`.
- **DoD:** teste de contrato rodando na CI; p95 < 800 ms com cache quente.

## Sprint 3 — App: scanner e resultado

- Expo + expo-router; permissões de câmera e localização com telas de recusa tratadas.
- Scanner (`expo-camera`) com leitura de EAN-13/EAN-8/UPC-A/ITF-14 e feedback tátil.
- Tela de resultado: componente `FaixaPrecoJusto` (o semáforo de 4 faixas do print) +
  lista de lojas com preço, distância e data.
- Estados de erro/vazio/carregando desenhados — não improvise depois.
- **DoD:** escanear um produto de verdade num mercado de verdade e ver o preço.

## Sprint 4 — App: filtros, histórico e polimento

- Seletor de raio (1/5/10/20 km) e seletor manual UF → município.
- Busca por nome quando o GTIN não retorna nada.
- Histórico local dos últimos 20 scans.
- Ícone, splash, textos, tela "de onde vêm os preços" (transparência).
- **DoD:** navegação completa sem travar; funciona com GPS negado.

## Sprint 5 — Beta fechado

- Build EAS → Google Play **teste interno** (até 100 testadores).
- 10–15 pessoas reais usando por 1 semana.
- Instrumentar as métricas do doc [01](01-produto-e-escopo.md), seção 6.
- Corrigir os 5 problemas mais citados.
- **DoD:** taxa de scan com resultado útil ≥ 60% medida em campo.

## Sprint 6 — Lançamento

- Política de privacidade publicada (obrigatória na Play Store) — doc [08](08-legal-lgpd-e-riscos.md).
- Ficha da loja, prints, descrição.
- Submissão à produção (Android).
- Monitoramento: alerta de Sentry + alerta de queda de fonte.

## Depois do MVP (ordem sugerida)

1. **Envio de NFC-e por QR Code** (a ideia original) — resolve os estados sem API e
   cria base proprietária. Requer conta de usuário e fila assíncrona.
2. **Lista de compras** com otimização de cesta ("onde a lista toda sai mais barato").
3. **Alerta de preço** por produto favorito.
4. **iOS**.
5. Expansão de UFs — um adapter por vez, com o teste de contrato como portão.

## Resumo de custos (mensal, MVP)

| Item | Custo |
|---|---|
| Fly.io (API + Postgres + Redis) | US$ 15–25 |
| Google Play (taxa única) | US$ 25 |
| Sentry (free tier) | US$ 0 |
| Domínio | ~R$ 40/ano |
| **Total recorrente** | **≈ US$ 20/mês** |

## Riscos do cronograma

| Risco | Sinal de alerta | Resposta |
|---|---|---|
| Sprint 0 falha | cobertura < 60% | pivotar para base colaborativa; +4 semanas |
| Fonte bloqueia o IP do servidor | 429/403 em massa | reduzir TTL de cache para +24 h, negociar acesso oficial com a SEFAZ |
| Play Store recusa por política de dados | rejeição | política de privacidade e declaração de coleta prontas **antes** da Sprint 6 |

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
| S0-6 | Testar `BarcodeDetector` + polyfill num protótipo mínimo (1 página HTML) em Chrome Android e Safari iOS | nota de compatibilidade real, não só a doc de terceiros |

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

## Sprint 2 — Backend: preço justo, busca por nome, cache e localidades

- `services/preco_justo.py` conforme doc [06](06-algoritmo-preco-justo.md) + testes unitários.
- `GET /v1/precos?termo=...` e `GET /v1/produtos/busca` (autocomplete via `pg_trgm`) —
  US-06 nasce no backend junto com o scan, não depois.
- Cache Redis (L2) e fallback para o Postgres quando a fonte falha.
- `/v1/localidades/*` (proxy IBGE) e filtro por `codigo_ibge`.
- Rate limiting, CORS, envelope de erro, Sentry, `/v1/health`.
- **DoD:** teste de contrato rodando na CI; p95 < 800 ms com cache quente; autocomplete
  < 300 ms.

## Sprint 3 — PWA: scanner, busca por nome e resultado

- Projeto Vite + React + TypeScript + `vite-plugin-pwa` (manifest + service worker
  do app shell desde o commit inicial — não deixar para o fim).
- Scanner via `getUserMedia` + `BarcodeDetector`/polyfill, com permissão de câmera
  tratada (estado de recusa com instrução de como reativar no navegador).
- Campo de busca por nome com autocomplete, lado a lado com o scanner na tela
  inicial — não escondido em outra aba.
- Tela de resultado: componente `FaixaPrecoJusto` (o semáforo de 4 faixas do print) +
  lista de lojas com preço, distância e data. Mesma tela para os dois caminhos de
  entrada (scan e nome).
- Estados de erro/vazio/carregando desenhados — não improvise depois.
- **DoD:** escanear um produto de verdade **e** buscar pelo nome, num mercado de
  verdade, e ver o preço nos dois casos — em Chrome Android e em Safari iOS.

## Sprint 4 — PWA: filtros, histórico e polimento

- Seletor de raio (1/5/10/20 km) e seletor manual UF → município.
- Geolocalização via `navigator.geolocation`, com fallback ao seletor manual quando
  negada.
- Histórico local (IndexedDB) dos últimos 20 scans/buscas.
- Ícones/splash do manifest, textos, tela "de onde vêm os preços" (transparência),
  prompt de instalação (`beforeinstallprompt`) discreto após o primeiro resultado útil.
- Auditoria **Lighthouse PWA** (Chrome DevTools): instalável, performance, acessibilidade.
- **DoD:** navegação completa sem travar; funciona com GPS negado; Lighthouse PWA ≥ 90.

## Sprint 5 — Beta fechado

- Deploy no homelab: `docker compose up -d --build` em `~/apps/compara-precos/`,
  Proxy Host novo no NPM, nova porta do Tailscale Funnel — tudo conforme doc
  [03](03-arquitetura.md), seção 9. Nada novo para contratar.
- Link compartilhado com 10–15 pessoas reais (amigos) por 1 semana (WhatsApp).
- Instrumentar as métricas do doc [01](01-produto-e-escopo.md), seção 6, separando
  taxa de sucesso por caminho de entrada (scan vs. nome) e por navegador.
- Corrigir os 5 problemas mais citados.
- **DoD:** taxa de busca (scan + nome) com resultado útil ≥ 60% medida em campo.

## Sprint 6 — Compartilhar com todo mundo

Não é um lançamento comercial — é abrir o link para o grupo maior de amigos que vai
usar de verdade.

- Revisar a tela "de onde vêm os preços" e a política de privacidade simples — doc
  [08](08-legal-lgpd-e-riscos.md). Mesmo sem intenção comercial, é barato e correto
  ser transparente com quem vai usar.
- Ícone/Open Graph revisados, para o link ficar apresentável quando compartilhado no
  WhatsApp.
- Publicar o link (Funnel, ou domínio próprio se tiver comprado um) no grupo.
- Monitoramento: alerta de Sentry + Netdata (já rodando no homelab) para saber que o
  serviço caiu antes que alguém precise avisar.
- (Opcional, sem pressa) empacotar como TWA para aparecer como ícone "de verdade" no
  Android de quem usar bastante — dificilmente vale a taxa da App Store da Apple para
  esse uso.

## Depois do MVP (ordem sugerida)

1. **Envio de NFC-e por QR Code** (a ideia original) — resolve os estados sem API e
   cria base proprietária. Requer conta de usuário e fila assíncrona.
2. **Lista de compras** com otimização de cesta ("onde a lista toda sai mais barato").
3. **Alerta de preço** por produto favorito.
4. **TWA na Play Store** (opcional) — mesma PWA, empacotada, só se quiser o ícone
   "oficial" no celular de quem usa.
5. Expansão de UFs — um adapter por vez, com o teste de contrato como portão.

## Resumo de custos (mensal, MVP)

| Item | Custo |
|---|---|
| Homelab (servidor, energia, internet) | R$ 0 adicional — infraestrutura já existente e já paga |
| Tailscale Funnel (TLS/ingress) | R$ 0 — coberto pelo free tier do Tailscale |
| Sentry (free tier) | R$ 0 |
| Domínio próprio (opcional) | ~R$ 40/ano — só se quiser trocar o link `.ts.net` |
| **Total recorrente** | **R$ 0/mês** |

Sem custo de nuvem e sem taxa de loja de apps — o único gasto possível é um domínio,
e é opcional.

## Riscos do cronograma

| Risco | Sinal de alerta | Resposta |
|---|---|---|
| Sprint 0 falha | cobertura < 60% | pivotar para base colaborativa; +4 semanas |
| Fonte bloqueia o IP do servidor | 429/403 em massa | reduzir TTL de cache para +24 h, negociar acesso oficial com a SEFAZ |
| `BarcodeDetector`/polyfill instável em iOS | scan falha/trava em teste real (S0-6) | busca por nome vira caminho padrão em iOS, não só alternativa |
| Baixa instalação da PWA | usuários usam só via navegador, sem "instalar" | não é bloqueante — o app funciona igual sem instalar; revisar prompt de instalação na Sprint 4 |
| Queda de energia/internet residencial | app fora do ar sem aviso | aceito conscientemente para uso não comercial (doc 03, seção 9.6); Netdata + alerta simples avisam rápido |
| Disco do servidor cheio/corrompido sem backup externo | perda da base de preços coletada | backup diário fora do disco físico (doc 03, seção 9.4) — configurar já na Sprint 1, não depois |

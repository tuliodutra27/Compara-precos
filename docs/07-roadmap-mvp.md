# 07 — Roadmap do MVP

Premissa: **1 desenvolvedor, meio período** (~15–20 h/semana). Com dedicação integral,
divida os prazos por dois. Sprints de 1 semana.

## Sprint 0 — Validação técnica — *status: parcialmente concluída, pausada*

O objetivo era responder: **existe dado utilizável?** Resposta: existe, mas atrás de
uma porta trancada que não é nossa para abrir sozinhos.

| # | Tarefa | Status | Resultado |
|---|---|---|---|
| S0-1 | Achar e mapear o endpoint real da Menor Preço Brasil | ✅ Feito (08/2026) | Endpoint encontrado via análise estática do APK (o app é Angular/Ionic via Capacitor, código legível). Ver doc [02](02-fontes-de-dados.md), seção 2.1. **Bloqueado** por exigir OAuth2 gov.br + autorização específica da Procergs para o nosso client_id (seção 2.2) — confirmado com teste real (`401` em 3 GTINs, sem token). |
| S0-1' (novo) | Pedido informal de acesso enviado à Procergs | 📤 Redigido, aguardando envio/resposta | Doc [adr/001](adr/001-pedido-procergs.md). **Decisão do autor: aguardar resposta antes de escrever código contra essa fonte** — sem prazo definido, pode nunca vir. |
| S0-4 | Ler termos de uso da Menor Preço Brasil / Procergs | Descontinuado por ora | Ficou irrelevante — bloqueado por autenticação antes de chegar a ser questão de ToS. |
| S0-5 | Medir cobertura: 30 GTINs × 3 regiões do RJ | ❌ Não realizável | Não dá para medir cobertura de uma API que devolve 401 para tudo. |
| S0-6 | Testar `BarcodeDetector` + polyfill (Chrome Android / Safari iOS) | ⏳ Pendente | Segue válido e independente do bloqueio acima — pode ser feito a qualquer momento. |

**O que isso muda no roadmap:** a base colaborativa via NFC-e (doc
[02](02-fontes-de-dados.md), seção 5) deixa de ser "fase 2/plano B" e vira **a fonte
de dados real do MVP** — não depende de autorização de ninguém, só de você e seus
amigos escaneando as próprias notas. Enquanto a Procergs não responde, **as sprints
de backend abaixo pausam a parte de "adapter de fonte estadual"** e o próximo passo
de validação técnica real é mapear o layout da página de consulta de NFC-e do RJ
(equivalente ao antigo S0-1, mas para a fonte que efetivamente vamos usar).

**UF:** RJ, mantido — a decisão não dependia da Menor Preço Brasil especificamente.

## Sprint 1 — Backend: núcleo + ingestão de NFC-e (pivô pós Sprint 0)

**Antes de codar:** validar o layout real da página de consulta pública de NFC-e do
RJ (URL do QR Code, se tem CAPTCHA, formato do HTML) — é o novo passo de validação
técnica que faltou no Sprint 0, adiado porque o esforço foi todo para a Menor Preço
Brasil primeiro.

- Projeto FastAPI + docker-compose (Postgres/PostGIS + Redis).
- Migrations do doc [04](04-modelo-de-dados.md).
- `POST /v1/notas` — recebe a URL/chave do QR Code de uma NFC-e do RJ, busca a página
  pública da SEFAZ-RJ, faz o parsing (descrição, GTIN, quantidade, valor unitário por
  item; CNPJ/endereço do estabelecimento; data/hora) e grava como `oferta` com
  `fonte = 'nota_usuario'` (doc 04).
- A interface `FontePrecos` (doc [03](03-arquitetura.md), seção 4) continua no design
  — é onde a Menor Preço Brasil entraria como adapter **se/quando** a Procergs
  autorizar — mas **não é implementada neste sprint**, porque não há fonte externa
  liberada para adaptar ainda.
- `GET /v1/precos` lendo as ofertas gravadas no banco por `lat`/`lon`/`raio_km`.
- **DoD:** enviar uma nota fiscal real sua (RJ), ver os itens aparecerem na tabela
  `oferta`, e `GET /v1/precos?gtin=...` devolver aquele preço.

## Sprint 2 — Backend: preço justo, busca por nome, cache e localidades

- `services/preco_justo.py` conforme doc [06](06-algoritmo-preco-justo.md) + testes unitários.
- `GET /v1/precos?termo=...` e `GET /v1/produtos/busca` (autocomplete via `pg_trgm`) —
  US-06 nasce no backend junto com o scan, não depois.
- Cache Redis (L2) sobre as consultas ao banco.
- `/v1/localidades/*` (proxy IBGE) e filtro por `codigo_ibge`.
- Rate limiting, CORS, envelope de erro, Sentry, `/v1/health`.
- **DoD:** p95 < 800 ms com cache quente; autocomplete < 300 ms.

## Sprint 3 — PWA: scanner de produto, busca por nome, envio de nota e resultado

- Projeto Vite + React + TypeScript + `vite-plugin-pwa` (manifest + service worker
  do app shell desde o commit inicial — não deixar para o fim).
- Scanner de **código de barras do produto** via `getUserMedia` + `BarcodeDetector`/
  polyfill, com permissão de câmera tratada (estado de recusa com instrução de como
  reativar no navegador).
- **Tela de "enviar nota"**: scanner do **QR Code da NFC-e** (mesma câmera, alvo
  diferente) → chama `POST /v1/notas` → mostra os itens reconhecidos para
  confirmação. É essa tela que alimenta a base de dados real do MVP (doc
  [02](02-fontes-de-dados.md), seção 5) — precisa estar visível e fácil de achar, não
  escondida em configurações.
- Campo de busca por nome com autocomplete, lado a lado com o scanner na tela
  inicial — não escondido em outra aba.
- Tela de resultado: componente `FaixaPrecoJusto` (o semáforo de 4 faixas do print) +
  lista de lojas com preço, distância e data. Mesma tela para os dois caminhos de
  entrada de busca (scan de produto e nome).
- Estados de erro/vazio/carregando desenhados — não improvise depois.
- **DoD:** enviar uma nota fiscal real (RJ) pelo app **e** depois escanear/buscar um
  dos produtos dela e ver o preço aparecer — em Chrome Android e em Safari iOS.

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

1. **Adapter Menor Preço Brasil**, se/quando a Procergs autorizar (doc
   [adr/001](adr/001-pedido-procergs.md)) — a interface `FontePrecos` já está
   desenhada para isso (doc 03), é plugar sem redesenhar nada.
2. **Lista de compras** com otimização de cesta ("onde a lista toda sai mais barato").
3. **Alerta de preço** por produto favorito.
4. **TWA na Play Store** (opcional) — mesma PWA, empacotada, só se quiser o ícone
   "oficial" no celular de quem usa.
5. Expansão de UFs — parser de NFC-e por estado (a base colaborativa é o padrão
   agora, não só fallback) ou adapter de API estadual, dependendo do que cada UF
   tiver disponível.

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
| ~~Sprint 0 falha~~ | cobertura < 60% | **Materializado, mas de um jeito diferente do previsto**: não foi falta de cobertura, foi bloqueio de acesso (login). Resposta aplicada: pivô para base colaborativa (seção acima), já em vigor. |
| Página de consulta de NFC-e do RJ tem CAPTCHA ou layout hostil a parsing | parser trava/quebra na validação técnica pendente | avaliar OCR ou preenchimento manual como fallback pontual; caso extremo, pausar o projeto até achar outra fonte |
| Procergs nunca responde ao pedido informal | sem novidade após semanas | sem ação — o MVP não depende dessa resposta para funcionar, só fica sem essa fonte extra indefinidamente |
| `BarcodeDetector`/polyfill instável em iOS | scan falha/trava em teste real (S0-6) | busca por nome vira caminho padrão em iOS, não só alternativa |
| Baixa instalação da PWA | usuários usam só via navegador, sem "instalar" | não é bloqueante — o app funciona igual sem instalar; revisar prompt de instalação na Sprint 4 |
| Queda de energia/internet residencial | app fora do ar sem aviso | aceito conscientemente para uso não comercial (doc 03, seção 9.6); Netdata + alerta simples avisam rápido |
| Disco do servidor cheio/corrompido sem backup externo | perda da base de preços coletada | backup diário fora do disco físico (doc 03, seção 9.4) — configurar já na Sprint 1, não depois |

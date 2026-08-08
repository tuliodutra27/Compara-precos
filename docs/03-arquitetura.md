# 03 — Arquitetura

## 1. Princípio central

**O front-end nunca fala direto com o governo.** Todas as consultas passam por um
backend próprio (BFF — Backend For Frontend). Isso garante:

- **cache** — a mesma consulta de GTIN+região não bate no portal estadual duas vezes;
- **normalização** — o cliente recebe um formato único, independente do estado;
- **resiliência** — se um portal cair ou mudar, corrigimos no servidor sem precisar
  que o usuário atualize nada (nem loja de apps, nem versão fixada — é web);
- **rate limiting sob controle** — um único IP de saída, com backoff, em vez de
  milhares de navegadores martelando o portal;
- **base própria** — cada consulta alimenta o nosso histórico, que vira o ativo real
  do produto.

```
┌──────────────────────┐     HTTPS/JSON      ┌──────────────────────────┐
│  PWA (React + Vite)   │ ──────────────────► │  BFF — FastAPI           │
│  câmera + GPS + SW    │ ◄────────────────── │  /v1/precos, /v1/produtos│
└──────────────────────┘                      └───────┬──────────────────┘
                                                      │
                              ┌───────────────────────┼───────────────────────┐
                              ▼                       ▼                       ▼
                      ┌──────────────┐       ┌─────────────────┐    ┌───────────────┐
                      │ Redis        │       │ Postgres +      │    │ Camada de     │
                      │ cache quente │       │ PostGIS         │    │ adapters (UF) │
                      │ TTL 6h       │       │ histórico       │    └───────┬───────┘
                      └──────────────┘       └─────────────────┘            │
                                                                 ┌──────────┼──────────┐
                                                                 ▼          ▼          ▼
                                                           MenorPreco   PrecoHora   NotaParana
                                                             Brasil        BA          PR
```

> Esse diagrama é a arquitetura lógica. Onde cada caixa efetivamente roda (o homelab
> do autor) está na seção 9.

## 2. Por que PWA e não app nativo

| | PWA | App nativo (React Native/Flutter) |
|---|---|---|
| Distribuição | link direto, deploy até estar no ar | build assinado, revisão de loja (dias), 2 lojas |
| Atualização | imediata para todo mundo (com ressalvas do SW, seção 6) | depende do usuário atualizar |
| Custo de entrada | R$ 0 | US$ 25 (Google) + US$ 99/ano (Apple) |
| Câmera/GPS | via API do navegador, funciona bem em Android; iOS com ressalvas (seção 5) | acesso nativo, sem ressalvas |
| Descoberta | SEO, link, WhatsApp | busca na loja de apps |
| "Instalar" | Adicionar à tela inicial, sem instalação de fato | instalação real |

Para o MVP, a velocidade de iterar (corrigir um adapter quebrado e o app já está no
ar para todo mundo, sem esperar revisão de loja) pesa mais do que a última milha de
polimento nativo. Se o produto validar, nada impede empacotar a mesma PWA como TWA
(Trusted Web Activity) e publicar na Play Store depois — ver doc
[08](08-legal-lgpd-e-riscos.md), seção 3.

## 3. Stack recomendada

| Camada | Escolha | Por quê |
|---|---|---|
| Frontend | **React + Vite + TypeScript**, `vite-plugin-pwa` (gera manifest + service worker) | build rápido, ecossistema maduro, PWA "de fábrica" via plugin |
| Leitura de código de barras | `BarcodeDetector` nativo quando disponível, com **polyfill** (`barcode-detector` no npm, que cai para um decoder WASM/ZXing quando o navegador não suporta a API nativa) | cobre Chrome/Edge/Android nativamente e Safari/Firefox via polyfill — ver seção 5 |
| Câmera | `getUserMedia` (`facingMode: "environment"`) | API padrão do navegador, sem dependência nativa |
| Geolocalização | `navigator.geolocation` | API padrão do navegador |
| Backend | **Python 3.12 + FastAPI** | async nativo (essencial para chamar 2–3 fontes em paralelo), tipagem com Pydantic, ecossistema de scraping/parsing maduro para a fase 2 |
| Banco | **Postgres 16 + PostGIS** | busca por raio geográfico é uma query, não um algoritmo |
| Cache | **Redis** | TTL por chave GTIN+geohash |
| Fila (fase 2) | **Celery + Redis** ou `arq` | processamento assíncrono de notas fiscais enviadas |
| Hospedagem | **Self-hosted no homelab do autor** (Ubuntu Server + Docker Compose), atrás do Nginx Proxy Manager já existente, publicado via Tailscale Funnel | R$ 0/mês — infra já paga; TLS automático via Funnel; deploy = `docker compose up -d --build`. Ver seção 9 |
| Observabilidade | **Sentry** (frontend + API, free tier) + **Netdata** (já rodando no homelab, cobre os containers novos sem configuração extra) | sem isso, um adapter quebrado só é descoberto por reclamação de usuário |

**Alternativa se preferir um stack só JS:** Node + Fastify + Prisma no backend
funciona bem e reduz para uma linguagem só. A escolha por Python é pela fase 2
(parsing de NFC-e). Nada mais no plano depende dessa decisão.

## 4. Camada de adapters (o coração do backend)

Sem mudança em relação a um app nativo — o backend é o mesmo dos dois lados. Cada
fonte estadual vira uma classe com a mesma interface; adicionar um estado = escrever
um arquivo, sem tocar em mais nada.

```python
# app/adapters/base.py
from abc import ABC, abstractmethod
from datetime import datetime
from pydantic import BaseModel

class OfertaBruta(BaseModel):
    gtin: str
    descricao: str
    preco: float
    cnpj: str
    nome_estabelecimento: str
    endereco: str | None
    latitude: float | None
    longitude: float | None
    vendido_em: datetime

class FontePrecos(ABC):
    uf: str            # "BA", "PR", ...
    nome: str

    @abstractmethod
    async def buscar_por_gtin(
        self, gtin: str, lat: float, lon: float, raio_km: int
    ) -> list[OfertaBruta]: ...

    @abstractmethod
    async def buscar_por_termo(
        self, termo: str, lat: float, lon: float, raio_km: int
    ) -> list[OfertaBruta]: ...
```

Um `registry` resolve a fonte pela UF da consulta:

```python
FONTES: dict[str, type[FontePrecos]] = {
    "BA": PrecoDaHoraBA,
    "PR": MenorPrecoParana,
    # default: MenorPrecoBrasil (cobre a maioria dos estados)
}
def fonte_para(uf: str) -> FontePrecos:
    return FONTES.get(uf, MenorPrecoBrasil)(uf=uf)
```

Regras obrigatórias para todo adapter:

1. **Timeout de 8 s** por requisição externa, com 1 retry e backoff exponencial.
2. **Nunca deixar exceção vazar**: falha vira lista vazia + log + métrica.
3. **Teste de contrato** (`tests/contracts/test_<uf>.py`) que roda diariamente na CI
   contra o endpoint real e falha ruidosamente quando o formato muda.
4. Sempre gravar o payload bruto (comprimido) por 7 dias — depurar mudança de formato
   sem isso é adivinhação.

## 5. Leitura de código de barras no navegador (o ponto mais delicado da PWA)

A `BarcodeDetector` (Shape Detection API) só está disponível nativamente em
**Chrome/Edge/Opera desktop e Android**. **Não** está em Firefox nem em Safari/iOS —
e boa parte dos usuários no Brasil está no iPhone.

Mitigação em duas camadas:

1. **Usar um pacote que já resolve isso** (`barcode-detector` no npm — um "ponyfill"
   que expõe a mesma interface `BarcodeDetector`, usando a API nativa quando existe e
   caindo para um decoder em WASM/JS baseado em ZXing quando não existe). O código do
   app chama uma interface só; a escolha de motor fica encapsulada.
2. **A busca por nome nunca é uma alternativa de segunda classe** (doc
   [01](01-produto-e-escopo.md), US-06). Em navegadores onde o scan por câmera for
   lento ou instável, a UI pode sugerir a busca por nome sem que isso pareça uma
   solução de contorno malfeita — é um caminho igualmente completo, com a mesma tela
   de resultado.

Notas técnicas adicionais:

- Câmera exige **contexto seguro (HTTPS)** — não funciona em `http://`, exceto
  `localhost` em dev.
- iOS Safari (14.3+) suporta `getUserMedia` mesmo em PWA instalada (modo standalone),
  mas vale testar a cada major release do iOS — histórico de regressões nesse ponto.
- Testar em pelo menos: Chrome Android, Safari iOS, Chrome desktop. Esses três cobrem
  a esmagadora maioria dos acessos esperados.

## 6. PWA: manifest e service worker

- **`manifest.json`**: nome, ícones (192/512px + maskable), `display: "standalone"`,
  `theme_color`, `start_url`. Gerado pelo `vite-plugin-pwa` a partir de um único SVG
  fonte.
- **Service worker**: cacheia o *app shell* (JS/CSS/HTML) para carregamento instantâneo
  em visitas repetidas e funcionamento básico offline (tela abre, mostra o histórico
  local, avisa "sem internet" na busca). **Não** cacheia respostas de `/v1/precos` —
  preço é informação que expira, cache deve vir do backend (Redis/Postgres), não do
  dispositivo.
- **Estratégia de atualização**: `registerType: "autoUpdate"` com um toast simples
  "Nova versão disponível, toque para atualizar" quando o SW novo estiver pronto —
  sem isso, um usuário pode ficar dias numa versão antiga com uma aba sempre aberta.
- **Prompt de instalação**: capturar o evento `beforeinstallprompt` e mostrar um
  convite discreto após o primeiro resultado útil (não no primeiro segundo de uso).

## 7. Estratégia de cache de dados (define o custo e a velocidade)

Três camadas:

| Camada | Chave | TTL | Objetivo |
|---|---|---|---|
| L1 — navegador | GTIN/termo + geohash5, em memória (estado do app) | sessão | voltar para um resultado já visto não gera requisição |
| L2 — Redis | `precos:{gtin\|termo}:{geohash5}:{raio}` | 6 h | requisições de usuários diferentes na mesma região |
| L3 — Postgres | tabela `oferta` | permanente | histórico, base para preço justo, resposta quando a fonte cai |

**Geohash de 5 caracteres** ≈ célula de 5 km — granularidade certa para agrupar
usuários da mesma região sem estourar a cardinalidade do cache.

Fluxo de uma consulta:

```
scan/busca → L1? → devolve
           → L2? → devolve
           → consulta fonte estadual (async)
                  ├─ sucesso → normaliza → grava Postgres + Redis → devolve
                  └─ falha   → busca no Postgres (últimos 30 dias, dentro do raio)
                               → devolve com flag "dados em cache, pode estar desatualizado"
```

Esse último ramo é o que impede a tela em branco quando a SEFAZ está fora do ar.

## 8. Estrutura de pastas proposta

```
compara-precos/
├── web/                          # PWA
│   ├── src/
│   │   ├── routes/                # Home/scanner, Resultado, Ajustes
│   │   ├── components/            # FaixaPrecoJusto, ListaLojas, BuscaPorNome, ...
│   │   ├── hooks/                 # useScanner, useLocalizacao, useAutocomplete
│   │   ├── api/                   # cliente do BFF
│   │   └── storage/                # histórico local (IndexedDB/localStorage)
│   ├── public/
│   │   ├── manifest.json
│   │   └── icons/
│   ├── vite.config.ts
│   └── package.json
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/v1/                # rotas
│   │   ├── adapters/              # uma classe por fonte
│   │   ├── core/                  # config, cache, geo
│   │   ├── models/                # SQLAlchemy
│   │   ├── schemas/               # Pydantic
│   │   └── services/              # preco_justo.py, agregacao.py
│   ├── alembic/
│   ├── tests/
│   └── pyproject.toml
├── docs/
└── spec/openapi.yaml
```

## 9. Deploy no homelab (produção)

> **Decisão:** o homelab pessoal do autor é a hospedagem **definitiva**, não um
> estágio temporário de beta. É um projeto não comercial, para uso do autor e de
> amigos convidados — não um produto público, então a barra de disponibilidade é
> "funciona bem para um grupo pequeno", não "SLA de produto comercial". Essa troca
> consciente está detalhada no fim desta seção.

### 9.1 O servidor

Ubuntu Server 24.04 num notebook (Intel i5-7200U, 4 vCPU, 7,7 GB RAM, disco de 290 GB —
93 GB livres hoje). Já roda, sem folga preocupante, Nextcloud, MySQL, Home Assistant,
Nginx Proxy Manager, AdGuard Home, Netdata e mais duas apps próprias (`uniasselvi-sjb`,
`climatempo-sjb`). O padrão do homelab é **um projeto Docker Compose por app**, em
`~/apps/<nome>/docker-compose.yml`, cada um na sua própria rede Docker isolada — o
Nginx Proxy Manager **não** compartilha rede com os apps; ele alcança cada um pela
porta publicada no host, via o gateway `172.18.0.1`. `compara-precos` segue o mesmo
padrão, sem inventar nada novo.

**Orçamento de recursos** (estimativa): Postgres+PostGIS ~300–500 MB, Redis
~50–100 MB, backend (FastAPI) ~150–250 MB, frontend (nginx estático) ~20 MB → total
~600 MB–900 MB. Folga confortável mesmo com tudo mais já rodando.

### 9.2 Ingress público: Tailscale Funnel + Nginx Proxy Manager

O Funnel **já está ativo** e público de verdade (sem exigir Tailscale instalado em
quem acessa): `https://<seu-host>.<sua-tailnet>.ts.net`. Hoje a raiz (`/`) desse
hostname aponta **direto** para `127.0.0.1:5000` (o `uniasselvi-sjb`) — **isso não
muda**. O plano para o compara-precos:

1. **Nova porta de Funnel**, `8443` (dentro do limite de 3 portas simultâneas do
   Funnel — 443/8443/10000 — sem tocar na porta 443 já em uso), apontando para o
   Nginx Proxy Manager (porta 80 do host) em vez de para um container direto. Isso
   centraliza no NPM o roteamento de tudo que vier depois — a próxima app não precisa
   de mais uma porta de Funnel, só de mais um Proxy Host.
2. **Novo Proxy Host no NPM**, domínio `<seu-host>.<sua-tailnet>.ts.net` (SSL
   desligado no NPM — o Funnel já termina TLS antes de chegar até ele), com duas
   Custom Locations:
   - `/` → `172.18.0.1:8090` (container `web`, nginx estático servindo a PWA)
   - `/api` → `172.18.0.1:8091` (container `backend`, FastAPI/uvicorn)
3. Isso muda a base URL do doc [05](05-contrato-api.md) de
   `https://api.comparaprecos.app/v1` para
   `https://<seu-host>.<sua-tailnet>.ts.net:8443/api/v1` — ajustar quando a Sprint 1
   sair do papel.

> A validar na hora (não é garantido sem testar): o `Host` header que chega no NPM
> pode incluir a porta (`<seu-host>.<sua-tailnet>.ts.net:8443`) dependendo do
> cliente. Se o Proxy Host cair em 404, é o primeiro lugar a olhar — ajustar o campo
> "Domain Names".

**HTTPS de graça:** o Funnel emite certificado Let's Encrypt automaticamente para o
hostname — resolve sozinho o pré-requisito de contexto seguro da PWA (câmera,
geolocalização, service worker) sem nenhuma configuração extra de TLS.

**DNS interno (opcional):** registrar `comparaprecos.homelab` no AdGuard (rewrite
para o IP Tailscale do servidor) + outro Proxy Host no NPM, no mesmo padrão de
`casa.homelab`/`nextcloud.homelab` já existentes — dá acesso direto de dentro da
tailnet/LAN sem depender do Funnel, útil para o próprio autor testar.

### 9.3 docker-compose de referência

Ilustrativo — nasce de fato na Sprint 1, não existe ainda:

```yaml
# ~/apps/compara-precos/docker-compose.yml
services:
  db:
    image: postgis/postgis:16-3.4
    restart: unless-stopped
    environment:
      - POSTGRES_DB=comparaprecos
      - POSTGRES_USER=comparaprecos
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - ./data/db:/var/lib/postgresql/data
    # sem porta publicada: só o backend precisa enxergar

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    # sem porta publicada

  backend:
    build: ./backend
    restart: unless-stopped
    ports:
      - "8091:8000"           # NPM alcança via 172.18.0.1:8091
    environment:
      - DATABASE_URL=postgresql://comparaprecos:${DB_PASSWORD}@db/comparaprecos
      - REDIS_URL=redis://redis:6379/0
    depends_on: [db, redis]

  web:
    build: ./web
    restart: unless-stopped
    ports:
      - "8090:80"              # NPM alcança via 172.18.0.1:8090
```

### 9.4 Backup — o item que não pode ficar para depois

A base de preços coletada **é o ativo real do produto** (doc
[04](04-modelo-de-dados.md)). Rodar num único disco, num único notebook, sem cópia
fora dele, significa que uma falha de disco apaga meses de coleta de uma vez.
Configurar já na Sprint 1, não depois de acumular dado que dói perder:

- `pg_dump` diário via cron (ex. `0 3 * * *`), comprimido.
- Cópia para **fora do disco físico do servidor** — o Nextcloud que já roda ali
  ajuda, mas é o mesmo disco; melhor complementar com um destino realmente externo
  (conta gratuita de object storage, ou até um `git push` do dump comprimido para um
  repositório privado).

### 9.5 Monitoramento

O Netdata já roda no host e cobre os containers novos automaticamente — sem
ferramenta nova, só vale configurar um alerta (Netdata já suporta) para container
parado ou uso anômalo de CPU/RAM.

### 9.6 A troca consciente

Hospedar num notebook doméstico atrás de internet residencial, sem redundância de
hardware nem energia, significa: se a energia cair, o ISP tiver uma instabilidade, ou
o notebook travar, o app fica fora do ar sem aviso, até alguém notar. Para um produto
comercial isso seria inaceitável — é por isso que o doc original recomendava
Fly.io/Railway. Para uso pessoal entre amigos, é uma troca razoável e consciente:
zero custo mensal, controle total, e o pior cenário é "manda mensagem no grupo que o
site caiu" — não perda de receita ou reputação.

## 10. Perguntas em aberto

- Vale comprar um domínio próprio (ex. ~R$ 40/ano) para não depender do link feio do
  `.ts.net`? Não é bloqueante — dá para trocar depois sem tocar em nada do backend,
  só reapontando Funnel/NPM.
- Vale medir, na Sprint 3, a taxa real de sucesso do polyfill de barcode em iOS antes
  de investir tempo em polimento de UI de scanner? Recomendação: **sim** — se a taxa
  for ruim, a busca por nome sobe de "alternativa" para "padrão" em iOS.

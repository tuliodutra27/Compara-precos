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
| Hospedagem frontend | **Cloudflare Pages** ou **Vercel** (free tier) | CDN global, HTTPS grátis por padrão — pré-requisito da PWA (seção 6) |
| Hospedagem backend | **Fly.io / Railway**, container Docker | custo baixo (~US$ 15–25/mês), deploy simples, migração fácil para AWS depois |
| Observabilidade | **Sentry** (frontend + API) + logs estruturados | sem isso, um adapter quebrado só é descoberto por reclamação de usuário |

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

## 9. Ambientes

- `dev` — docker-compose local (Postgres+PostGIS, Redis) + `vite dev` com proxy para
  `localhost:8000`.
- `prod` — front-end na CDN (Cloudflare Pages/Vercel), backend no Fly.io, Postgres
  gerenciado com backup diário.
- Sem `staging` no MVP: não compensa. Feature flags resolvem.

## 10. Perguntas em aberto

- Cloudflare Pages ou Vercel para o front? Ambos têm free tier suficiente para o MVP —
  não é decisão estruturante, escolha pelo que você já conhece.
- Vale medir, na Sprint 3, a taxa real de sucesso do polyfill de barcode em iOS antes
  de investir tempo em polimento de UI de scanner? Recomendação: **sim** — se a taxa
  for ruim, a busca por nome sobe de "alternativa" para "padrão" em iOS.

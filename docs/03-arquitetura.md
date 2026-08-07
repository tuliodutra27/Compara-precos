# 03 — Arquitetura

## 1. Princípio central

**O app nunca fala direto com o governo.** Todas as consultas passam por um backend
próprio (BFF — Backend For Frontend). Isso garante:

- **cache** — a mesma consulta de GTIN+região não bate no portal estadual duas vezes;
- **normalização** — o app recebe um formato único, independente do estado;
- **resiliência** — se um portal cair ou mudar, corrigimos no servidor sem publicar
  nova versão nas lojas de apps (o que leva dias);
- **rate limiting sob controle** — um único IP de saída, com backoff, em vez de milhares
  de celulares martelando o portal;
- **base própria** — cada consulta alimenta o nosso histórico, que vira o ativo real
  do produto.

```
┌─────────────────┐        HTTPS/JSON        ┌──────────────────────────┐
│  App (Expo RN)  │ ───────────────────────► │  BFF — FastAPI           │
│  câmera + GPS   │ ◄─────────────────────── │  /v1/precos, /v1/produtos│
└─────────────────┘                          └───────┬──────────────────┘
                                                     │
                              ┌──────────────────────┼──────────────────────┐
                              ▼                      ▼                      ▼
                      ┌──────────────┐      ┌─────────────────┐    ┌───────────────┐
                      │ Redis        │      │ Postgres +      │    │ Camada de     │
                      │ cache quente │      │ PostGIS         │    │ adapters (UF) │
                      │ TTL 6h       │      │ histórico       │    └───────┬───────┘
                      └──────────────┘      └─────────────────┘            │
                                                                ┌──────────┼──────────┐
                                                                ▼          ▼          ▼
                                                          MenorPreco   PrecoHora   NotaParana
                                                            Brasil        BA          PR
```

## 2. Stack recomendada

| Camada | Escolha | Por quê |
|---|---|---|
| Mobile | **React Native + Expo** (SDK atual, `expo-camera` + `expo-location`) | um código para Android/iOS, câmera e GPS resolvidos, build na nuvem via EAS, OTA update sem passar pela loja |
| Backend | **Python 3.12 + FastAPI** | async nativo (essencial para chamar 2-3 fontes em paralelo), tipagem com Pydantic, ecossistema de scraping/parsing maduro para a fase 2 |
| Banco | **Postgres 16 + PostGIS** | busca por raio geográfico é uma query, não um algoritmo |
| Cache | **Redis** | TTL por chave GTIN+geohash |
| Fila (fase 2) | **Celery + Redis** ou `arq` | processamento assíncrono de notas fiscais enviadas |
| Infra | **Fly.io / Railway** no MVP; container Docker | custo baixo (~US$ 15–25/mês), deploy simples, migração fácil para AWS depois |
| Observabilidade | **Sentry** (app + API) + logs estruturados | sem isso, um adapter quebrado só é descoberto por review de 1 estrela |

**Alternativa se preferir um stack só JS:** Node + Fastify + Prisma funciona bem e
reduz o número de linguagens para uma. A escolha por Python é pelo parsing de NFC-e da
fase 2. Nada mais no plano depende dessa decisão.

## 3. Camada de adapters (o coração do backend)

Cada fonte estadual vira uma classe com a mesma interface. Adicionar um estado =
escrever um arquivo, sem tocar em mais nada.

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

## 4. Estratégia de cache (define o custo e a velocidade)

Três camadas:

| Camada | Chave | TTL | Objetivo |
|---|---|---|---|
| L1 — app | GTIN + geohash5 | 1 h | scan repetido no corredor do mercado não gera requisição |
| L2 — Redis | `precos:{gtin}:{geohash5}:{raio}` | 6 h | requisições de usuários diferentes na mesma região |
| L3 — Postgres | tabela `oferta` | permanente | histórico, base para preço justo, resposta quando a fonte cai |

**Geohash de 5 caracteres** ≈ célula de 5 km — granularidade certa para agrupar
usuários da mesma região sem estourar a cardinalidade do cache.

Fluxo de uma consulta:

```
scan → L1? → devolve
     → L2? → devolve e grava L1
     → consulta fonte estadual (async)
            ├─ sucesso → normaliza → grava Postgres + Redis → devolve
            └─ falha   → busca no Postgres (últimos 30 dias, dentro do raio)
                         → devolve com flag "dados em cache, pode estar desatualizado"
```

Esse último ramo é o que impede a tela branca quando a SEFAZ está fora do ar.

## 5. Estrutura de pastas proposta

```
compara-precos/
├── mobile/                      # Expo
│   ├── app/                     # rotas (expo-router)
│   │   ├── index.tsx            # home / scanner
│   │   ├── resultado/[gtin].tsx
│   │   └── ajustes.tsx          # raio, cidade, permissões
│   ├── src/
│   │   ├── api/                 # cliente do BFF
│   │   ├── components/          # FaixaPrecoJusto, ListaLojas, ...
│   │   ├── hooks/               # useScanner, useLocalizacao
│   │   └── storage/             # histórico local
│   └── app.json
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/v1/              # rotas
│   │   ├── adapters/            # uma classe por fonte
│   │   ├── core/                # config, cache, geo
│   │   ├── models/              # SQLAlchemy
│   │   ├── schemas/             # Pydantic
│   │   └── services/            # preco_justo.py, agregacao.py
│   ├── alembic/
│   ├── tests/
│   └── pyproject.toml
├── docs/
└── spec/openapi.yaml
```

## 6. Ambientes

- `dev` — docker-compose local (Postgres+PostGIS, Redis), app apontando para `localhost`.
- `prod` — Fly.io, com Postgres gerenciado e backup diário.
- Sem `staging` no MVP: não compensa. Feature flags resolvem.

## 7. Perguntas em aberto

- Fly.io ou Railway? Decidir pelo que você já conhece — não é decisão estruturante.
- Publicar iOS no MVP? Custa US$ 99/ano e a revisão é mais lenta. Recomendação:
  **Android primeiro**, iOS na v1.1.

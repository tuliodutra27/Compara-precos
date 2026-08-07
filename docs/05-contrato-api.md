# 05 — Contrato da API (BFF v1)

Base URL: `https://api.comparaprecos.app/v1` — contrato formal em
[`spec/openapi.yaml`](../spec/openapi.yaml).

Regras gerais:

- Todas as respostas em JSON, `Content-Type: application/json; charset=utf-8`.
- Erros seguem o mesmo envelope (seção 5).
- O app envia `X-App-Version` e um `X-Device-Id` anônimo (UUID gerado no primeiro
  uso, **sem** relação com identidade — ver doc [08](08-legal-lgpd-e-riscos.md)).
- Sem autenticação de usuário no MVP; rate limit por `X-Device-Id` + IP.

## 1. `GET /v1/precos`

O endpoint principal: dado um GTIN e uma localização, devolve faixas + lista de lojas.

### Parâmetros

| Nome | Tipo | Obrigatório | Padrão | Descrição |
|---|---|---|---|---|
| `gtin` | string(8–14) | sim* | — | código de barras escaneado |
| `termo` | string | sim* | — | busca por nome (alternativa ao `gtin`) |
| `lat` / `lon` | float | não** | — | posição do usuário |
| `codigo_ibge` | string(7) | não** | — | município escolhido manualmente |
| `raio_km` | int | não | `10` | 1, 5, 10 ou 20 |
| `dias` | int | não | `30` | janela temporal (máx. 90) |

\* exatamente um de `gtin` ou `termo`.
\** exatamente um entre (`lat`+`lon`) e `codigo_ibge`.

### Resposta 200

```json
{
  "produto": {
    "gtin": "7896484410687",
    "descricao": "BEBIDA LACTEA ENERGIA 1L CHOCOLATE",
    "marca": null,
    "imagem_url": null
  },
  "faixas": {
    "preco_justo": 4.49,
    "limite_barato": 4.48,
    "limite_razoavel": 4.49,
    "limite_toleravel": 4.50,
    "minimo": 4.29,
    "maximo": 5.99,
    "amostra_suficiente": true
  },
  "amostra": {
    "n_precos": 60,
    "n_lojas": 5,
    "periodo_inicio": "2026-06-21",
    "periodo_fim": "2026-07-18",
    "abrangencia": "raio_10km"
  },
  "ofertas": [
    {
      "preco": 4.29,
      "faixa": "barato",
      "vendido_em": "2026-07-18T19:04:00-03:00",
      "desatualizado": false,
      "estabelecimento": {
        "cnpj": "00000000000191",
        "nome": "SUPERMERCADO EXEMPLO",
        "endereco": "R. das Flores, 100 - Centro",
        "municipio": "Curitiba",
        "uf": "PR",
        "latitude": -25.4284,
        "longitude": -49.2733,
        "distancia_km": 1.4
      }
    }
  ],
  "meta": {
    "fonte": "menor_preco_brasil",
    "cache": "hit",
    "gerado_em": "2026-08-07T10:00:00-03:00",
    "aviso": null
  }
}
```

Campos que a UI **precisa** usar:

- `faixas.amostra_suficiente = false` → esconder o semáforo, mostrar só `ofertas`.
- `ofertas[].desatualizado = true` (venda com mais de 30 dias) → badge de alerta.
- `meta.aviso` — preenchido quando servimos dados de fallback do banco porque a fonte
  estadual falhou. A UI mostra a mensagem literalmente.
- `amostra.abrangencia` — `raio_10km`, `municipio` ou `uf`, para avisar quando houve
  expansão automática da área (doc 06, seção 6).

### Resposta 200 com resultado vazio

Produto não encontrado **não é erro**:

```json
{
  "produto": { "gtin": "7896484410687", "descricao": null },
  "faixas": null,
  "amostra": { "n_precos": 0, "n_lojas": 0 },
  "ofertas": [],
  "meta": { "aviso": "Nenhum preço registrado para este produto nesta região." }
}
```

## 2. `GET /v1/produtos/{gtin}`

Metadados do produto sem os preços — usado pelo histórico local do app, que guarda só
os GTINs.

## 3. `GET /v1/localidades/estados` e `GET /v1/localidades/estados/{uf}/municipios`

Proxy cacheado (TTL 30 dias) da API do IBGE, para o seletor manual de cidade. Existe
para o app não depender diretamente de terceiros e para respostas menores.

```json
[{ "codigo_ibge": "4106902", "nome": "Curitiba", "uf": "PR" }]
```

## 4. `GET /v1/health`

`{"status": "ok", "fontes": {"menor_preco_brasil": "up", "preco_hora_ba": "degraded"}}`

Alimentado pelos testes de contrato — permite avisar dentro do app quando uma fonte
está fora do ar.

## 5. Envelope de erro

```json
{
  "erro": {
    "codigo": "PARAMETRO_INVALIDO",
    "mensagem": "Informe lat/lon ou codigo_ibge.",
    "detalhes": { "campo": "lat" }
  }
}
```

| HTTP | `codigo` | Quando |
|---|---|---|
| 400 | `PARAMETRO_INVALIDO` | validação de entrada |
| 404 | `NAO_ENCONTRADO` | rota/recurso inexistente |
| 429 | `LIMITE_EXCEDIDO` | rate limit (inclui `Retry-After`) |
| 502 | `FONTE_INDISPONIVEL` | fonte caiu **e** não há fallback no banco |
| 500 | `ERRO_INTERNO` | bug — sempre com `trace_id` no Sentry |

## 6. Rate limiting

| Escopo | Limite |
|---|---|
| por `X-Device-Id` | 60 req/min, 1.000 req/dia |
| por IP | 300 req/min |

## 7. Versionamento

Prefixo `/v1` no path. Mudança quebra-contrato só em `/v2`, com `/v1` mantido por 90
dias — apps antigos continuam nas lojas por muito tempo.

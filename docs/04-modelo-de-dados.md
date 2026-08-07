# 04 — Modelo de dados

Postgres 16 + PostGIS. O esquema abaixo é o DDL inicial do MVP (vira a primeira
migration do Alembic).

## 1. Visão geral

```
produto (GTIN)  1 ──── N  oferta  N ──── 1  estabelecimento (CNPJ)
                            │
                            └── fonte: 'api_estadual' | 'nota_usuario'
```

Uma **oferta** é um fato imutável: "este GTIN custou R$ X, neste CNPJ, nesta data".
Nada é atualizado — só inserido. Isso dá histórico de preço de graça.

## 2. DDL

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pg_trgm;   -- busca por nome do produto

-- ---------------------------------------------------------------- produto
CREATE TABLE produto (
    gtin              VARCHAR(14) PRIMARY KEY,
    descricao         TEXT        NOT NULL,   -- descrição mais frequente/normalizada
    marca             TEXT,
    ncm               VARCHAR(8),
    unidade           VARCHAR(10),            -- UN, KG, L...
    imagem_url        TEXT,
    criado_em         TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_produto_descricao_trgm ON produto USING gin (descricao gin_trgm_ops);

-- -------------------------------------------------------- estabelecimento
CREATE TABLE estabelecimento (
    cnpj              VARCHAR(14) PRIMARY KEY,
    razao_social      TEXT        NOT NULL,
    nome_fantasia     TEXT,
    logradouro        TEXT,
    numero            VARCHAR(20),
    bairro            TEXT,
    municipio         TEXT        NOT NULL,
    codigo_ibge       VARCHAR(7),             -- município IBGE, para filtro por cidade
    uf                CHAR(2)     NOT NULL,
    cep               VARCHAR(8),
    geom              GEOGRAPHY(POINT, 4326), -- NULL quando a fonte não devolve coords
    criado_em         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_estab_geom  ON estabelecimento USING gist (geom);
CREATE INDEX idx_estab_ibge  ON estabelecimento (codigo_ibge);

-- ----------------------------------------------------------------- oferta
CREATE TYPE origem_dado AS ENUM ('api_estadual', 'nota_usuario');

CREATE TABLE oferta (
    id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    gtin              VARCHAR(14) NOT NULL REFERENCES produto(gtin),
    cnpj              VARCHAR(14) NOT NULL REFERENCES estabelecimento(cnpj),
    preco             NUMERIC(10,2) NOT NULL CHECK (preco > 0),
    descricao_origem  TEXT,                   -- descrição crua do lojista
    vendido_em        TIMESTAMPTZ NOT NULL,
    fonte             origem_dado NOT NULL,
    fonte_detalhe     TEXT,                   -- 'menor_preco_brasil', 'preco_hora_ba'...
    chave_nfe         VARCHAR(44),            -- só quando fonte = nota_usuario
    coletado_em       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- evita duplicar o mesmo preço vindo da mesma fonte
CREATE UNIQUE INDEX uq_oferta_dedup
    ON oferta (gtin, cnpj, preco, vendido_em, fonte);

-- índice que serve a consulta quente: preços de um GTIN nos últimos 30 dias
CREATE INDEX idx_oferta_gtin_data ON oferta (gtin, vendido_em DESC);
CREATE INDEX idx_oferta_cnpj_data ON oferta (cnpj, vendido_em DESC);
```

## 3. Consulta principal (preços por GTIN dentro de um raio)

```sql
SELECT  o.preco,
        o.vendido_em,
        e.cnpj,
        COALESCE(e.nome_fantasia, e.razao_social) AS loja,
        e.logradouro, e.bairro, e.municipio, e.uf,
        ST_Distance(e.geom, ST_MakePoint(:lon, :lat)::geography) / 1000.0 AS distancia_km
FROM    oferta o
JOIN    estabelecimento e ON e.cnpj = o.cnpj
WHERE   o.gtin = :gtin
  AND   o.vendido_em >= now() - INTERVAL '30 days'
  AND   ST_DWithin(e.geom, ST_MakePoint(:lon, :lat)::geography, :raio_metros)
ORDER BY o.preco ASC, o.vendido_em DESC;
```

Variante por cidade (quando o usuário escolhe manualmente, sem GPS): trocar o
`ST_DWithin` por `e.codigo_ibge = :codigo_ibge`.

## 4. Última oferta por loja

A lista de resultados deve mostrar **uma linha por loja** (a venda mais recente),
não todas as vendas:

```sql
SELECT DISTINCT ON (o.cnpj)
       o.cnpj, o.preco, o.vendido_em
FROM   oferta o
WHERE  o.gtin = :gtin
  AND  o.vendido_em >= now() - INTERVAL '30 days'
ORDER  BY o.cnpj, o.vendido_em DESC;
```

## 5. Retenção e volume

- Ofertas com mais de **12 meses** vão para tabela de agregado mensal
  (`oferta_mensal`: gtin, codigo_ibge, mês, min, p25, mediana, p75, max, n) e são
  apagadas da tabela quente. Só é necessário quando passar de ~50 M linhas.
- Estimativa de MVP (1 estado, 5 mil usuários, 4 scans/semana): ~1 M ofertas/mês.
  Postgres nem sente.

## 6. Perguntas em aberto

- Guardar o payload bruto das fontes no banco (`jsonb`) ou em object storage?
  Recomendação: **object storage** (S3/R2), 7 dias de retenção, fora do Postgres.
- Precisamos de tabela de usuários no MVP? Não — só na fase 2, com o envio de notas.

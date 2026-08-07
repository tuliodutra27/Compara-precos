# 06 — Algoritmo de "preço justo"

Este é o diferencial de UX do app de referência (o print que originou o projeto):
em vez de despejar uma lista de preços, ele responde *"o preço que você está vendo é
caro?"* com um semáforo de quatro faixas.

```
Barato        Razoável      Tolerável     Caro
Abaixo de     Abaixo de     Abaixo de     Acima de
R$ 4,48       R$ 4,48       R$ 4,49       R$ 4,50
                    ● preço justo: R$ 4,49
"Calculado a partir de 60 pesquisas de preço em 5 lojas, entre 21/06 e 18/07."
```

## 1. Definição

Dado o conjunto de preços `P` do GTIN, dentro da região e da janela temporal
escolhidas, com **uma amostra por loja** (a venda mais recente de cada CNPJ — sem isso
uma rede grande com muitas vendas distorce tudo):

| Grandeza | Cálculo |
|---|---|
| `preco_justo` | **mediana** de `P` |
| Limite Barato / Razoável | percentil 25 |
| Limite Razoável / Tolerável | mediana (p50) |
| Limite Tolerável / Caro | percentil 75 |

Faixas resultantes:

- **Barato** — `preco ≤ p25`
- **Razoável** — `p25 < preco ≤ p50`
- **Tolerável** — `p50 < preco ≤ p75`
- **Caro** — `preco > p75`

Mediana e quartis (em vez de média) porque preços de varejo têm outliers grosseiros:
erro de digitação do lojista, produto em promoção de encerramento, embalagem diferente
com o mesmo GTIN.

## 2. Limpeza obrigatória antes de calcular

Sem isso o número mente:

1. **Uma amostra por CNPJ** (a mais recente dentro da janela).
2. **Janela temporal:** 30 dias. Preço de 3 meses atrás não descreve o mercado de hoje.
3. **Remoção de outliers por IQR:** descartar `preco < p25 − 1.5·IQR` ou
   `preco > p75 + 1.5·IQR`, onde `IQR = p75 − p25`.
4. **Piso de sanidade:** descartar preços ≤ R$ 0,01 ou > 100× a mediana bruta.
5. **Amostra mínima:** com `n < 5` lojas, **não exibir faixas** — mostrar apenas a lista
   de preços encontrados e o aviso "amostra pequena". Um semáforo calculado sobre 2
   preços é pior que semáforo nenhum.

## 3. Implementação de referência

```python
# backend/app/services/preco_justo.py
from dataclasses import dataclass
from statistics import median
from datetime import datetime

MIN_AMOSTRAS = 5

@dataclass
class FaixasPreco:
    preco_justo: float      # mediana
    limite_barato: float    # p25
    limite_razoavel: float  # p50
    limite_toleravel: float # p75
    minimo: float
    maximo: float
    n_precos: int
    n_lojas: int
    periodo_inicio: datetime
    periodo_fim: datetime

def percentil(valores: list[float], p: float) -> float:
    """Percentil por interpolação linear (mesmo método do numpy.percentile)."""
    if not valores:
        raise ValueError("lista vazia")
    ordenados = sorted(valores)
    if len(ordenados) == 1:
        return ordenados[0]
    k = (len(ordenados) - 1) * p
    f, c = int(k), min(int(k) + 1, len(ordenados) - 1)
    return ordenados[f] + (ordenados[c] - ordenados[f]) * (k - f)

def remover_outliers(precos: list[float]) -> list[float]:
    if len(precos) < 4:
        return precos
    q1, q3 = percentil(precos, 0.25), percentil(precos, 0.75)
    iqr = q3 - q1
    lo, hi = q1 - 1.5 * iqr, q3 + 1.5 * iqr
    limpos = [p for p in precos if lo <= p <= hi]
    return limpos or precos          # nunca devolve vazio

def calcular_faixas(ofertas: list["Oferta"]) -> FaixasPreco | None:
    """`ofertas` já deve vir com no máximo uma linha por CNPJ."""
    if len({o.cnpj for o in ofertas}) < MIN_AMOSTRAS:
        return None                  # amostra insuficiente → UI mostra só a lista

    precos = remover_outliers([float(o.preco) for o in ofertas])
    datas = [o.vendido_em for o in ofertas]

    return FaixasPreco(
        preco_justo     = round(median(precos), 2),
        limite_barato   = round(percentil(precos, 0.25), 2),
        limite_razoavel = round(percentil(precos, 0.50), 2),
        limite_toleravel= round(percentil(precos, 0.75), 2),
        minimo          = round(min(precos), 2),
        maximo          = round(max(precos), 2),
        n_precos        = len(precos),
        n_lojas         = len({o.cnpj for o in ofertas}),
        periodo_inicio  = min(datas),
        periodo_fim     = max(datas),
    )
```

## 4. Testes que precisam existir

```
- lista vazia                        → None
- 4 lojas                            → None (abaixo do mínimo)
- 5 preços iguais                    → todas as faixas iguais, sem divisão por zero
- 1 preço absurdo (R$ 4.499,00)      → descartado pelo IQR, mediana não se move
- número par de amostras             → mediana interpolada corretamente
- mesma loja com 10 vendas           → conta como 1 amostra
```

## 5. Rodapé de transparência (obrigatório na UI)

Copiar o padrão do app de referência, porque é o que sustenta a confiança:

> "Os preços foram calculados a partir de **{n_precos}** pesquisas de preço em
> **{n_lojas}** lojas, entre **{periodo_inicio}** e **{periodo_fim}**."

Se `n_lojas < 5`: substituir a faixa por *"Poucos dados nesta região — mostrando os
preços encontrados."*

## 6. Perguntas em aberto

- A faixa deve considerar a **região do usuário** ou o **estado inteiro**? O app de
  referência parece usar a região. Recomendação: calcular sobre o raio escolhido e,
  se `n < 5`, expandir automaticamente para o município, depois para a UF, sinalizando
  a expansão na UI ("dados de todo o município").
- Vale ponderar por recência (preço de ontem pesa mais que o de 29 dias atrás)?
  Fica para a v2 — no MVP, simplicidade é mais importante que precisão marginal.

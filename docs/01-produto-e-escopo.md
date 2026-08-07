# 01 — Produto e escopo do MVP

## 1. Problema

O consumidor brasileiro não sabe, no momento da compra, se o preço na gôndola está
caro ou barato. Os dados existem — toda NFC-e emitida no país registra GTIN, preço,
CNPJ e data —, mas estão espalhados em portais estaduais com usabilidade ruim e sem
comparação regional consolidada.

## 2. Proposta de valor

Escanear o código de barras e receber, em menos de 5 segundos:

- o **menor preço** encontrado num raio configurável (padrão 10 km);
- a **faixa de preço justo** (Barato / Razoável / Tolerável / Caro) — o mesmo conceito
  do app de referência do print, detalhado em [06](06-algoritmo-preco-justo.md);
- a **lista de estabelecimentos** ordenada por preço, com distância e data da última
  venda registrada;
- a **transparência da amostra**: "calculado a partir de N preços em M lojas, entre
  DD/MM e DD/MM" — sem isso o número não é confiável.

## 3. Personas

| Persona | Necessidade | O que o MVP entrega |
|---|---|---|
| **Ana, compra do mês** | Não pagar 30% a mais no mesmo item | Scan + comparação por raio |
| **Carlos, compra rápida** | Decidir na gôndola em segundos | Faixa de preço justo com semáforo |
| **Dona Marta, orçamento apertado** | Saber em qual mercado do bairro ir | Ranking de lojas por preço |

## 4. Escopo do MVP

### Dentro (must have)

1. **Scanner de código de barras** (EAN-13, EAN-8, UPC-A, ITF-14) com a câmera.
2. **Busca de preços por GTIN** na fonte de dados do estado ativo.
3. **Filtro geográfico**: por GPS (raio de 1 / 5 / 10 / 20 km) **ou** por cidade/UF
   escolhida manualmente (via API de localidades do IBGE), para quem nega a permissão
   de localização ou quer pesquisar antes de sair de casa.
4. **Tela de resultado**: preço justo, faixas, menor/maior preço, lista de lojas
   ordenada por preço com distância, e o rodapé de transparência da amostra.
5. **Busca por nome** do produto como fallback quando o GTIN não retorna nada.
6. **Histórico local** dos últimos scans (offline, no dispositivo).
7. **Backend próprio (BFF)** com cache — o app nunca fala direto com o governo.

### Fora (v2+)

- Envio de notas fiscais pelo usuário (QR Code da NFC-e) — **especificado**, mas
  implementado só na fase 2. É o plano B para estados sem API.
- Lista de compras e otimização de cesta ("onde comprar a lista toda mais barato").
- Alerta de queda de preço / histórico de longo prazo por produto.
- Contas de usuário, login social, sincronização entre dispositivos.
- Gamificação, cashback, cupons.
- Cobertura nacional simultânea.

### Não-objetivos explícitos

- Não é marketplace: não vendemos nem intermediamos compra.
- Não prometemos preço em tempo real da gôndola — mostramos o **último preço
  registrado em nota fiscal**, sempre com a data, e isso precisa estar visível na UI.

## 5. User stories principais

```
US-01  Como consumidor, quero escanear o código de barras de um produto
       para ver quanto ele custa nas lojas perto de mim.
       DoD: scan → resultado em < 5s (p95) com ≥ 1 loja, ou mensagem clara de "sem dados".

US-02  Como consumidor sem GPS ativo, quero escolher minha cidade
       para ver preços daquela região.
       DoD: seletor UF → município alimentado pela API do IBGE, persistido no app.

US-03  Como consumidor, quero saber se o preço que estou vendo é justo
       para decidir sem comparar loja a loja.
       DoD: faixa calculada conforme doc 06, com N ≥ 5 amostras; abaixo disso, a UI
       mostra "amostra insuficiente" em vez de um número frágil.

US-04  Como consumidor, quero ver a distância até cada loja
       para não atravessar a cidade por R$ 0,20.
       DoD: distância em km calculada a partir da posição do usuário, ordenável.

US-05  Como consumidor, quero ver quando aquele preço foi registrado
       para saber se está desatualizado.
       DoD: data da venda por item; itens com mais de 30 dias sinalizados.
```

## 6. Métricas de sucesso do MVP

| Métrica | Meta na 4ª semana pós-lançamento |
|---|---|
| Taxa de scan com resultado útil (≥ 3 lojas) | ≥ 60% |
| Tempo p95 do scan até o resultado | ≤ 5 s |
| Erro de API para o usuário | ≤ 2% das buscas |
| Retenção D7 | ≥ 20% |
| Scans por usuário ativo/semana | ≥ 4 |

A **primeira** métrica é a que decide o produto: se a cobertura de dados for baixa,
nenhuma melhoria de UI salva o app — o caminho passa a ser a base colaborativa (fase 2).

## 7. Perguntas em aberto

- Qual estado é o de lançamento? Depende do teste de campo da doc 02 (qualidade da
  API + volume de NFC-e + a região onde você mora, para conseguir testar na rua).
- O app precisa de conta de usuário já no MVP? Recomendação: **não** — só é
  necessário quando entrar o envio de notas.

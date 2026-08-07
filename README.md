# Compara Preços

App mobile para comparar preços de supermercado no Brasil: o usuário abre a câmera,
escaneia o código de barras (GTIN/EAN) do produto e vê onde ele está mais barato
**perto dele**, com base em preços reais extraídos de notas fiscais eletrônicas
(NFC-e/NF-e) publicados pelas Secretarias de Fazenda estaduais.

> Status: **planejamento do MVP**. Este repositório contém, hoje, a especificação
> completa do produto, da arquitetura e do roadmap. Nenhum código de produção foi
> escrito ainda — o próximo passo é a Sprint 0 descrita em
> [`docs/07-roadmap-mvp.md`](docs/07-roadmap-mvp.md).

## A ideia em uma frase

> "Vale a pena comprar aqui?" respondido em menos de 5 segundos, com preço mínimo,
> mediano e a faixa de preço justo dos últimos 30 dias, num raio configurável.

## Índice da documentação

| Documento | O que responde |
|---|---|
| [01 — Produto e escopo](docs/01-produto-e-escopo.md) | O que entra e o que **não** entra no MVP, personas, user stories, métricas de sucesso |
| [02 — Fontes de dados](docs/02-fontes-de-dados.md) | Quais APIs do governo existem por estado, o que cada uma entrega, riscos e plano B |
| [03 — Arquitetura](docs/03-arquitetura.md) | Stack recomendada, diagrama de componentes, padrão de adapters por UF, cache |
| [04 — Modelo de dados](docs/04-modelo-de-dados.md) | Esquema Postgres/PostGIS com DDL pronto |
| [05 — Contrato da API](docs/05-contrato-api.md) | Endpoints do nosso backend (BFF) com exemplos de request/response |
| [06 — Algoritmo de preço justo](docs/06-algoritmo-preco-justo.md) | Como calcular as faixas Barato / Razoável / Tolerável / Caro do print de referência |
| [07 — Roadmap do MVP](docs/07-roadmap-mvp.md) | 8 semanas divididas em sprints, com Definition of Done por entrega |
| [08 — Jurídico, LGPD e riscos](docs/08-legal-lgpd-e-riscos.md) | Uso de dados públicos, geolocalização, termos de uso, riscos técnicos |
| [spec/openapi.yaml](spec/openapi.yaml) | Contrato formal da API v1 |

## Decisões já tomadas (premissas do plano)

Estas premissas guiam toda a documentação. Se alguma mudar, os documentos afetados
estão indicados entre parênteses.

1. **Mobile:** React Native + Expo (app único Android/iOS, câmera e GPS prontos). — *(03)*
2. **Backend:** Python + FastAPI, com Postgres 16 + PostGIS e Redis. — *(03, 04)*
3. **Fonte primária:** APIs públicas estaduais de preços (Menor Preço Brasil,
   Preço da Hora BA, Menor Preço Nota Paraná). — *(02)*
4. **Fonte secundária (fase 2):** notas fiscais enviadas pelos próprios usuários via
   QR Code da NFC-e — a ideia original, que vira o plano B quando o estado não tem API. — *(02, 07)*
5. **MVP geográfico:** lançar em **1 estado** (o de maior qualidade de API), não no Brasil todo. — *(01, 07)*

## Como contribuir com o planejamento

Abra uma issue ou um PR contra a branch de trabalho. Cada documento tem uma seção
"Perguntas em aberto" no final — é lá que ficam as decisões que ainda precisam de
validação (principalmente as de **02 — Fontes de dados**, que exigem teste de campo
contra os endpoints reais).

## Licença

[MIT](LICENSE)

# Compara Preços

PWA (aplicativo web instalável, sem loja de apps obrigatória) para comparar preços
de supermercado no Brasil: o usuário abre o site no navegador, escaneia o código de
barras (GTIN/EAN) do produto pela câmera **ou digita o nome do produto** (como
aparece na nota fiscal) e vê onde ele está mais barato **perto dele**, com base em
preços reais extraídos de notas fiscais eletrônicas (NFC-e/NF-e) publicados pelas
Secretarias de Fazenda estaduais.

> Status (08/2026): **Sprint 0 parcialmente concluída, pivô de fonte de dados feito.**
> A fonte estadual planejada originalmente (Menor Preço Brasil) foi testada ao vivo e
> está bloqueada por exigir autorização específica da Procergs (pedido enviado,
> aguardando resposta — [`docs/adr/001`](docs/adr/001-pedido-procergs.md)). O caminho
> real do MVP passou a ser a base colaborativa via NFC-e (a ideia original do projeto),
> detalhada em [`docs/02`](docs/02-fontes-de-dados.md), seção 5. Nenhum código de
> produção foi escrito ainda — próximo passo real: validar o layout da página de
> consulta de NFC-e do RJ, depois Sprint 1 em [`docs/07-roadmap-mvp.md`](docs/07-roadmap-mvp.md).

## A ideia em uma frase

> "Vale a pena comprar aqui?" respondido em menos de 5 segundos — via scan ou por
> nome digitado — com preço mínimo, mediano e a faixa de preço justo dos últimos
> 30 dias, num raio configurável.

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
| [docs/adr/001 — Pedido à Procergs](docs/adr/001-pedido-procergs.md) | Rascunho do pedido de autorização de acesso à API da Menor Preço Brasil |

## Decisões já tomadas (premissas do plano)

Estas premissas guiam toda a documentação. Se alguma mudar, os documentos afetados
estão indicados entre parênteses.

1. **Frontend:** PWA — React + Vite, instalável direto do navegador. Scanner via
   câmera do navegador (`getUserMedia`), busca por nome sempre disponível como
   caminho alternativo/primário. — *(03)*
2. **Backend:** Python + FastAPI, com Postgres 16 + PostGIS e Redis. — *(03, 04)*
3. **Fonte de dados do MVP:** base colaborativa via QR Code da NFC-e (a ideia
   original) — cada usuário escaneia as próprias notas. APIs estaduais prontas
   (Menor Preço Brasil, Preço da Hora BA, Menor Preço Nota Paraná) ficam para quando/
   se forem liberadas (a do RJ está bloqueada — ver status acima). — *(02, 07)*
5. **MVP geográfico:** lançar em **1 estado** (o de maior qualidade de API), não no Brasil todo. — *(01, 07)*
6. **Hospedagem:** self-hosted no homelab pessoal do autor (Tailscale Funnel + Nginx
   Proxy Manager), uso **não comercial**, restrito ao autor e a amigos convidados —
   não um produto público. — *(03, 07, 08)*

## Como contribuir com o planejamento

Abra uma issue ou um PR contra a branch de trabalho. Cada documento tem uma seção
"Perguntas em aberto" no final — é lá que ficam as decisões que ainda precisam de
validação (principalmente as de **02 — Fontes de dados**, que exigem teste de campo
contra os endpoints reais).

## Licença

[MIT](LICENSE)

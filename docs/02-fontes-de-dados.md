# 02 — Fontes de dados de preço no Brasil

> **Aviso importante:** os endpoints abaixo foram levantados por pesquisa e por
> engenharia reversa pública (repositórios de terceiros e os próprios apps oficiais).
> **Nenhum deles foi testado ao vivo neste levantamento** — a rede do ambiente onde
> este documento foi escrito bloqueia domínios `.gov.br`. A **Sprint 0** do roadmap
> existe exatamente para validar cada um deles na sua máquina antes de escrever
> qualquer linha do app. Trate esta página como hipótese a confirmar, não como verdade.

## 1. O fato central

**Não existe uma API nacional única de preços ao consumidor.** O dado nasce na NFC-e,
que é autorizada pela SEFAZ de cada estado — logo, a base é estadual. O que existe é
uma **plataforma compartilhada** que a maioria dos estados adotou.

## 2. Fonte primária: Menor Preço Brasil

- **O que é:** app oficial de consulta de preços desenvolvido pela Procergs/Sefaz-RS
  em parceria com o ENCAT, lançado nacionalmente pelo CONFAZ. Nasceu como "Menor Preço
  Nota Gaúcha" (RS, 2019) e virou plataforma compartilhada.
- **Cobertura:** mais de 15 estados + DF. Estados citados pelas fontes oficiais como
  aderentes: AC, AL, AP, AM, BA, CE, ES, MA, MT, MG, PA, PE, PI, RJ, RN, RO, RR, SC,
  SE, TO, DF e RS. **Confirmar a lista atual na Sprint 0** — houve migração recente
  (o ES, por exemplo, adotou o Menor Preço Brasil como app único, aposentando o seu).
- **Como funciona:** consulta por descrição do produto **ou por código de barras**,
  com geolocalização e raio — exatamente o fluxo que queremos.
- **Como integrar:** não há documentação pública de API. O caminho é inspecionar o
  tráfego do app oficial (`br.gov.rs.procergs.mpbr`) ou do site, com proxy HTTPS
  (mitmproxy/Charles), e mapear os endpoints, headers e formato de resposta.
  → tarefa **S0-1** do roadmap.

## 3. Fontes estaduais próprias (fora da plataforma compartilhada)

### 3.1 Preço da Hora — Bahia

- Portal: `precodahora.ba.gov.br`
- Backend consultável por **GTIN + latitude/longitude**, com parâmetros observados em
  implementações públicas de terceiros:

| Parâmetro | Exemplo | Significado |
|---|---|---|
| `gtin` | `7896484410687` | código de barras |
| `latitude` / `longitude` | `-12.97`, `-38.50` | centro da busca |
| `raio` | `15` | raio em km |
| `horas` | `72` | janela temporal das vendas |
| `ordenar` | `preco.asc` | ordenação |
| `pagina` | `1` | paginação |

- Resposta em JSON com dois blocos: `produto` (preço, GTIN, descrição, NCM, imagem) e
  `estabelecimento` (CNPJ, nome, endereço, distância).
- **Pegadinha conhecida:** exige cookie de sessão + token CSRF obtidos numa requisição
  inicial ao portal. Precisa de um "handshake" antes da busca.
- Referência de implementação: <https://github.com/igorpereirag/precodahora_api>

### 3.2 Menor Preço — Nota Paraná

- Portal: `menorpreco.notaparana.pr.gov.br`
- Endpoint observado (formato REST simples, resposta JSON):

```
GET /api/v1/produtos?local=<lat>,<lon>&termo=<GTIN ou texto>&raio=<km>&offset=<n>
```

- Raio máximo divulgado pelo app oficial: **20 km**. Busca aceita código de barras
  **ou** nome do produto. Base atualizada em tempo real a cada NFC-e emitida.

### 3.3 Espírito Santo, Pernambuco, Rio Grande do Sul

Tinham portais próprios (`internet.sefaz.es.gov.br/informacoes/menorpreco`,
`nfg.sefaz.rs.gov.br`) que estão sendo consolidados no Menor Preço Brasil. Servem como
**fonte de referência/validação cruzada**, não como integração principal.

### 3.4 São Paulo

Maior mercado do país e **sem app estadual de preços** equivalente. Se SP for
prioridade comercial, o caminho é a base colaborativa (seção 5) — o que muda o roadmap.
Por isso a recomendação é **não lançar em SP no MVP**.

## 4. Metadados do produto (nome, marca, imagem)

As APIs de preço às vezes devolvem a descrição digitada pelo lojista — que é suja
("BEB LACT ENERG 1L CHOC"). Para exibir um nome decente:

| Fonte | O que dá | Custo / atrito |
|---|---|---|
| **CCG/SVRS** — Cadastro Centralizado de GTIN (NT 2022.001) | descrição oficial do dono da marca, NCM, CEST | Web Service exige **certificado digital A1/A3**; há consulta manual em `dfe-portal.svrs.rs.gov.br/NFE/Gtin` |
| **GS1 Brasil (Verified by GS1)** | dado canônico + imagem | API com cadastro/contrato, limites de consulta |
| **Base própria** | normalização + curadoria | de graça, cresce com o uso |

**Decisão para o MVP:** usar a descrição da própria API de preço, normalizada
(uppercase, remoção de abreviações comuns), e guardar em tabela `produto` a **descrição
mais frequente** entre as ofertas daquele GTIN — é boa o suficiente e custa zero.
Integração com CCG/GS1 fica para a v2.

## 5. Plano B (e complemento): base colaborativa via NFC-e

A sua ideia original — usuários enviando notas — continua valendo, como **fase 2** e
como cobertura para estados sem API:

1. O usuário escaneia o **QR Code da NFC-e** (todo cupom fiscal tem).
2. O QR contém a URL de consulta pública da SEFAZ daquele estado, com a chave de acesso.
3. O backend busca essa URL e faz o parsing da página → lista de itens com **descrição,
   GTIN, quantidade, valor unitário**, além de CNPJ, endereço e data/hora.
4. Uma nota = dezenas de preços. Poucos usuários ativos geram base rápido.

Por que não no MVP: cada estado tem um layout de página diferente (parser por UF),
há CAPTCHA em alguns portais, e sem usuários não há base — é um problema clássico de
partida a frio, que as APIs estaduais resolvem de graça.

**Mas projete o banco desde já para as duas origens** — é por isso que a tabela
`oferta` tem a coluna `fonte` no doc [04](04-modelo-de-dados.md).

## 6. Geografia: IBGE

Única API oficial, documentada, aberta e estável do plano:

```
GET https://servicodados.ibge.gov.br/api/v1/localidades/estados
GET https://servicodados.ibge.gov.br/api/v1/localidades/estados/{UF}/municipios
```

Usada no seletor "cidade/região" do filtro manual. Carregar uma vez e cachear no app.

## 7. Riscos das fontes e mitigação

| Risco | Impacto | Mitigação |
|---|---|---|
| Endpoint não documentado muda sem aviso | app quebra | adapter isolado por UF + testes de contrato diários em CI + alerta |
| Rate limiting / bloqueio de IP | indisponibilidade | cache agressivo (doc 03), 1 IP de saída fixo, backoff, `User-Agent` identificável e honesto |
| Termos de uso proíbem uso automatizado | risco jurídico | ler o ToS de cada portal na Sprint 0; ver doc [08](08-legal-lgpd-e-riscos.md) |
| Cobertura baixa de GTIN em lojas pequenas | resultado vazio | fallback para busca por nome + mensagem honesta de "sem dados" |
| Preço antigo apresentado como atual | perda de confiança | data sempre visível; item com > 30 dias sinalizado |

## 8. Perguntas em aberto

- Menor Preço Brasil: qual o endpoint real, e ele aceita GTIN direto? **(S0-1)**
- Preço da Hora BA: o handshake de CSRF ainda é necessário? **(S0-2)**
- Nota Paraná: o endpoint `/api/v1/produtos` continua ativo e sem autenticação? **(S0-3)**
- Algum portal expõe termos de uso que proíbem consumo programático? **(S0-4)**

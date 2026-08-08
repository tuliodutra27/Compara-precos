# 02 — Fontes de dados de preço no Brasil

> **Status (08/2026): Sprint 0 parcialmente concluída.** A seção 2 (Menor Preço
> Brasil) foi validada ao vivo — endpoint real encontrado e testado, mas **bloqueado**
> por exigência de autorização da Procergs (seção 2.2). As seções 3–4 continuam sendo
> hipótese de pesquisa, não testadas. O caminho de implementação atual do MVP é a
> seção 5 (base colaborativa via NFC-e), ainda não validada para o layout do RJ
> especificamente.

## 1. O fato central

**Não existe uma API nacional única de preços ao consumidor.** O dado nasce na NFC-e,
que é autorizada pela SEFAZ de cada estado — logo, a base é estadual. O que existe é
uma **plataforma compartilhada** que a maioria dos estados adotou.

**Estado de lançamento decidido: Rio de Janeiro.** O RJ **não tem** portal próprio de
preços (diferente de BA/PR) — sua única fonte de API seria a plataforma compartilhada
da seção 2, que está bloqueada (seção 2.2). Por isso o caminho real do MVP é a base
colaborativa (seção 5): você e seus amigos alimentando a base com as próprias notas
fiscais. As seções 3.1–3.3 (BA, PR, ES/PE/RS) ficam como referência para expansão de
UF **se/quando** a Menor Preço Brasil for desbloqueada, não como parte do MVP atual.

## 2. Fonte primária (e única, para o RJ): Menor Preço Brasil — **BLOQUEADA**

> **Atualização pós Sprint 0 (validado ao vivo, não é mais hipótese):** o endpoint foi
> encontrado e confirmado, mas exige login gov.br **e** autorização específica da
> Procergs que não está disponível por autosserviço. Ver seção 2.1 e 2.2 abaixo.
> Enquanto não houver resposta da Procergs (doc [adr/001](adr/001-pedido-procergs.md)),
> esta fonte está **fora do caminho de implementação** — o projeto segue pela base
> colaborativa (seção 5), que virou a fonte real do MVP, não mais plano B.

- **O que é:** app oficial de consulta de preços desenvolvido pela Procergs/Sefaz-RS
  em parceria com o ENCAT, lançado nacionalmente pelo CONFAZ. Nasceu como "Menor Preço
  Nota Gaúcha" (RS, 2019) e virou plataforma compartilhada.
- **Cobertura:** mais de 15 estados + DF. Estados citados pelas fontes oficiais como
  aderentes: AC, AL, AP, AM, BA, CE, ES, MA, MT, MG, PA, PE, PI, RJ, RN, RO, RR, SC,
  SE, TO, DF e RS. **Confirmar a lista atual na Sprint 0** — houve migração recente
  (o ES, por exemplo, adotou o Menor Preço Brasil como app único, aposentando o seu).
- **Como funciona:** consulta por descrição do produto **ou por código de barras**,
  com geolocalização e raio — exatamente o fluxo que queremos.
### 2.1 O que foi confirmado (S0-1, feito em 08/2026)

A captura de tráfego por proxy não funcionou de forma prática (o app se recusou a
abrir com um proxy configurado — provavelmente checagem de rede além de cert pinning
simples). O caminho que funcionou foi **análise estática do APK**: o app não é
nativo, é **Angular/Ionic empacotado com Capacitor** — uma WebView carregando
JavaScript, então o código roda praticamente em texto puro dentro do próprio `.apk`
(`adb pull` do pacote instalado + `unzip` já é suficiente para ler `assets/public/*.js`
sem precisar descompilar bytecode).

Isso revelou, com certeza (não é mais suposição):

| O quê | Valor |
|---|---|
| API base | `https://mprs.sefaz.rs.gov.br/API/ConsultaMenorPrecoBrasil/api/v1/` |
| Busca por GTIN | `GET Item/PorGtin?pesquisa.gtin=<n>&pesquisa.latitude=<lat>&pesquisa.longitude=<lon>&pesquisa.nroDiaPrz=30&pesquisa.nroKmDistancia=<km>` |
| Busca por descrição | `GET Item/PorDescricao?pesquisa.descricao=<texto>&...` |
| Busca por NCM | `GET Item/PorNcm?pesquisa.ncms=<ncm>&...` |
| Tamanhos de GTIN aceitos | 8, 12, 13 ou 14 dígitos (o app decide GTIN vs. NCM pelo tamanho do texto digitado) |
| Raio | padrão 5 km, máximo 30 km (confirmado no FAQ oficial) |
| Janela temporal | 30 dias (`nroDiaPrz: 30`) — bate com o que o doc [06](06-algoritmo-preco-justo.md) já assumia |

### 2.2 O que bloqueia o uso (o motivo real desta fonte estar suspensa)

Testado ao vivo (`curl`, sem token, 3 GTINs reais, coordenadas do Rio de Janeiro):
**todas as chamadas devolvem `401 Authorization has been denied`.** A pesquisa de
preço — que deveria ser um dado público — exige login.

O login é **OAuth2 via gov.br** (Login Único, escopo
`openid+email+profile+govbr_confiabilidades`), não um login do próprio app. E aqui
está o bloqueio real, confirmado por pesquisa na documentação oficial do gov.br:

> **Login Único só prova quem é o usuário. Ele não dá acesso a sistemas de terceiros.**
> Mesmo que cada usuário do compara-precos logasse com a própria conta gov.br, a API
> da Procergs (`mprs.sefaz.rs.gov.br`) verifica o `client_id` (campo `aud` do token) —
> ela só aceita tokens emitidos *para o client_id do app oficial da Menor Preço
> Brasil*. Nosso app precisaria que a **Procergs autorizasse especificamente o nosso
> client_id** a chamar a API deles — uma autorização discricionária, pedida
> diretamente a eles, não um cadastro de autosserviço.
>
> O processo formal de integração (Conecta gov.br) exige entidade registrada com
> diretor de TI — não se aplica a projeto pessoal. O único canal viável encontrado é
> um "Fale Conosco" de atendimento ao cidadão, sem garantia de resposta.

**Decisão registrada:** foi redigido um pedido informal (doc
[adr/001](adr/001-pedido-procergs.md)) e o projeto **aguarda resposta antes de
prosseguir com código** contra esta fonte — decisão explícita do autor, não uma
suposição minha. Enquanto isso, a base colaborativa (seção 5) é o caminho de
implementação real do MVP no RJ.

## 3. Fontes estaduais próprias (fora da plataforma compartilhada, referência para expansão futura de UF)

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

Essa normalização importa mais agora que a busca por nome (US-06) é caminho
primário, não só fallback: é sobre essa descrição normalizada que o autocomplete
(`GET /v1/produtos/busca`, doc [05](05-contrato-api.md)) roda a busca por similaridade
de texto (`pg_trgm`). Descrição suja = autocomplete ruim = usuário não encontra o
produto digitando — o mesmo problema de qualidade, só que na porta de entrada por
texto em vez da porta de entrada por GTIN.

## 5. A ideia original vira o caminho real do MVP: base colaborativa via NFC-e

> **Estava desenhada como "fase 2"/plano B; com a Menor Preço Brasil bloqueada
> (seção 2.2), esta seção é o que efetivamente se implementa primeiro no RJ.** A
> mecânica não muda, só a ordem no roadmap — ver doc [07](07-roadmap-mvp.md).

A sua ideia original — usuários enviando notas — é a base de dados real deste MVP:

1. O usuário escaneia o **QR Code da NFC-e** (todo cupom fiscal tem).
2. O QR contém a URL de consulta pública da SEFAZ daquele estado, com a chave de acesso.
3. O backend busca essa URL e faz o parsing da página → lista de itens com **descrição,
   GTIN, quantidade, valor unitário**, além de CNPJ, endereço e data/hora.
4. Uma nota = dezenas de preços. Poucos usuários ativos geram base rápido.

**Por que era considerada "não no MVP" originalmente:** cada estado tem um layout de
página diferente (parser por UF — mas agora só precisa o do RJ), há CAPTCHA em alguns
portais, e sem usuários não há base — problema clássico de partida a frio. No cenário
atual (uso pessoal, você + amigos, base pequena e controlada), a partida a frio não é
o obstáculo que seria para um produto público — vocês mesmos alimentam a base com as
compras reais que já fariam de qualquer forma.

**Falta validar (vira o novo S0-1', ver doc 07):** o layout da página de consulta de
NFC-e do **RJ especificamente** (`www.fazenda.rj.gov.br` ou equivalente) — path do QR
Code, se tem CAPTCHA, formato do HTML a parsear. Isso não foi verificado ainda nesta
rodada.

**O banco já está pronto para as duas origens** — é por isso que a tabela `oferta`
tem a coluna `fonte` no doc [04](04-modelo-de-dados.md). Se a Procergs um dia
autorizar (doc [adr/001](adr/001-pedido-procergs.md)), a Menor Preço Brasil entra
como fonte adicional sem redesenhar nada.

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

- ~~Menor Preço Brasil: qual o endpoint real, ele aceita GTIN direto?~~ **Respondido,
  seção 2.1.**
- ~~O app usa certificate pinning?~~ **Não chegou a ser necessário confirmar** — a
  captura por proxy falhou por outro motivo (o app não abriu com proxy configurado),
  mas a análise estática do APK resolveu sem precisar dessa resposta.
- A Procergs vai responder ao pedido informal (doc [adr/001](adr/001-pedido-procergs.md))?
  Se sim, o que dizem? **Bloqueante para reativar a seção 2.**
- Qual o layout da página de consulta pública de NFC-e do **RJ** (para o parser da
  seção 5)? Ainda não verificado — próximo passo real de validação técnica.
- A Menor Preço Brasil expõe termos de uso que proíbem consumo programático? Ficou
  irrelevante por ora (bloqueada por autenticação antes mesmo de chegar no ToS).
- *(Backlog de expansão de UF, fora do MVP do RJ)* Preço da Hora BA: o handshake de
  CSRF ainda é necessário? Nota Paraná: o endpoint `/api/v1/produtos` continua ativo e
  sem autenticação?

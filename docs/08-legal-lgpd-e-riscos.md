# 08 — Jurídico, LGPD e riscos

> Este documento organiza os pontos de atenção e propõe encaminhamentos. **Não é
> parecer jurídico.** Antes de qualquer intenção comercial futura, vale uma consulta
> rápida com advogado de direito digital — algumas horas resolvem tudo o que está aqui.

**Contexto:** o Compara Preços, aqui, é um projeto pessoal e **não comercial** —
hospedado no homelab do autor (doc [03](03-arquitetura.md), seção 9) e usado por ele
e por amigos convidados, não distribuído ao público em geral. Isso reduz a régua de
vários itens abaixo (não há relação de consumo, não há processador de dados
terceirizado — o autor é controlador e operador ao mesmo tempo, tudo dentro da
própria infraestrutura). Duas coisas, porém, continuam valendo do mesmo jeito
independentemente de escala: os termos de uso das fontes de dado da SEFAZ (seção 1) e
as boas práticas de privacidade com quem usa o app (seção 2) — tratar bem o dado de
quem confia em você não é uma obrigação que só nasce em produto comercial.

## 1. Uso dos dados das SEFAZ

> **Atualizado após teste real (08/2026):** a premissa original desta seção — "portais
> abertos ao cidadão, sem login" — **era verdadeira para a ideia em geral, mas não
> para a Menor Preço Brasil especificamente**. Testado ao vivo: a API exige OAuth2
> via gov.br, e mesmo com login a Procergs precisa autorizar nosso `client_id`
> explicitamente (ver doc [02](02-fontes-de-dados.md), seção 2.2, e o pedido em
> [adr/001](adr/001-pedido-procergs.md)). O que segue abaixo passa a valer para a
> **página pública de consulta de NFC-e por chave de acesso** (a que o QR Code do
> cupom fiscal aponta) — essa sim é, por desenho, aberta a qualquer cidadão sem login,
> em todos os estados, porque é para isso que ela existe.

**A favor:**

- As páginas de consulta de NFC-e por chave de acesso são publicadas pelas próprias
  SEFAZ estaduais, **abertas ao cidadão, sem login** — qualquer pessoa com o QR Code
  de uma nota (a sua, a de um amigo que compartilhou) pode consultar.
- Preço praticado por CNPJ não é dado pessoal — é informação comercial de pessoa
  jurídica, fora do escopo da LGPD.
- Há amparo na Lei de Acesso à Informação (Lei 12.527/2011) e na política de dados
  abertos (Decreto 8.777/2016) para o acesso a dados públicos.
- Diferente de bater numa API de terceiro repetidamente, aqui **quem decide fazer a
  consulta é o próprio usuário, um QR Code de cada vez** — o padrão de uso é
  fundamentalmente mais parecido com "abrir a página no navegador" do que com
  scraping em massa.

**Contra / a verificar (agora específico da consulta de NFC-e do RJ):**

- Layout da página, presença de CAPTCHA e eventual rate limiting — nada disso foi
  verificado ainda para o RJ (é o próximo passo real de validação técnica).
- Volume alto de requisições pode ser tratado como abuso, mesmo sendo dado público —
  mas aqui o volume é limitado pelo número de notas que os próprios usuários enviam,
  não por um scraper varrendo GTINs.

**Encaminhamentos:**

1. `User-Agent` honesto e identificável (`ComparaPrecos/1.0 (+https://site; contato@…)`)
   — nada de fingir ser navegador.
2. Uma requisição por nota enviada — não há necessidade de cache agressivo aqui, o
   padrão de uso já é naturalmente de baixo volume.
3. Respeitar `robots.txt` e `Retry-After`; backoff exponencial em 429/503.
4. **Sobre a Menor Preço Brasil especificamente:** o pedido informal já foi redigido
   e a decisão é aguardar resposta antes de investir mais tempo nela (doc
   [adr/001](adr/001-pedido-procergs.md)) — não é mais um "seria bom fazer", é o
   estado real do projeto.
5. Sempre **creditar a fonte** na UI ("Dados: notas fiscais enviadas pelos usuários,
   consultadas na SEFAZ/RJ") e
   nunca sugerir vínculo oficial com o governo.

## 2. LGPD — dados dos nossos usuários

O MVP foi desenhado para coletar o mínimo possível:

| Dado | Finalidade | Base legal | Retenção |
|---|---|---|---|
| Localização aproximada (lat/lon) | filtrar preços por região | **consentimento** (permissão do SO) | não persistida; usada em memória, cacheada só como geohash de 5 chars |
| `X-Device-Id` (UUID aleatório) | rate limiting e antifraude | legítimo interesse | 90 dias |
| GTINs escaneados | métricas de produto | legítimo interesse (agregado) | agregado; sem vínculo com o device após 30 dias |
| Histórico de scans | conveniência | — | **só no dispositivo**, nunca sai do celular |

Regras que o código precisa respeitar:

1. **Não armazenar coordenada exata do usuário** em banco ou log. Truncar para geohash
   de 5 caracteres (~5 km) antes de qualquer persistência.
2. **Não pedir login** no MVP — sem cadastro, não há titular identificado.
3. **Explicar antes de pedir**: tela de contexto antes do prompt de permissão de
   localização do navegador, e o app precisa funcionar sem GPS (seletor manual de
   cidade) — o navegador só mostra o prompt uma vez e negar costuma ser definitivo
   até o usuário mexer nas configurações do site.
4. **Política de privacidade** publicada em URL própria, linkada no rodapé da PWA —
   fica pronta para quando/se o app for empacotado numa loja (seção 3).
5. Canal de contato do encarregado (pode ser o seu e-mail) na política.
6. **HTTPS obrigatório**: câmera (`getUserMedia`), geolocalização e service worker só
   funcionam em contexto seguro. Não é só boa prática — é pré-requisito técnico.

Na **fase 2** (envio de notas fiscais) o cenário muda: a NFC-e pode conter CPF do
consumidor. Regra desde já: **descartar o CPF no momento do parsing**, antes de
qualquer gravação. Nunca persistir CPF.

## 3. Play Store / App Store (opcional, e ainda menos relevante para uso não comercial)

Como PWA, o lançamento **não passa por loja de apps nem por processo de revisão** —
é publicar no link do Funnel/domínio e pronto (doc [03](03-arquitetura.md), seção 9).
Dado que o uso é pessoal, entre amigos, a Play Store deixa de ser sequer um objetivo
natural — só entraria em cena se, um dia, o escopo mudar para algo mais amplo. Se
fizer sentido nesse cenário futuro (mais descoberta, ícone "oficial" no launcher), o
caminho de menor esforço é empacotar a mesma PWA como **TWA (Trusted Web Activity)**,
sem reescrever nada. Pontos a atender nesse momento futuro:

| Exigência | Como atender |
|---|---|
| Formulário "Segurança de dados" | declarar localização (não armazenada) e ID de dispositivo |
| Justificativa do uso de localização | descrição clara na ficha + tela de contexto in-app |
| Política de privacidade acessível | URL pública, também linkada dentro do app |
| Não simular app oficial de governo | nome, ícone e textos sem brasão, sem "SEFAZ", sem "gov" |

O último item é o de maior risco de reprovação, TWA ou não. Evite qualquer elemento
visual que sugira origem governamental — vale também para a PWA em si.

## 4. Isenção de responsabilidade (dentro do app)

Texto sugerido para a tela "De onde vêm os preços":

> Os preços exibidos vêm de notas fiscais eletrônicas autorizadas pelas Secretarias
> de Fazenda estaduais e se referem a vendas **já realizadas**. O valor cobrado hoje
> na loja pode ser diferente. Este aplicativo não tem vínculo com órgãos públicos nem
> com os estabelecimentos listados.

## 5. Riscos técnicos e de produto

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Layout/CAPTCHA da página de NFC-e do RJ muda ou bloqueia parsing | média | alto | parser isolado por UF + teste de contrato + payload bruto guardado 7 dias (mesmo padrão já previsto para adapters de API) |
| Cobertura de dados baixa (poucas notas enviadas = poucas lojas cobertas) | alta | alto | é o risco esperado de partir de uma base colaborativa pequena; aceito para uso entre amigos, sem correção automática possível |
| GTIN ausente na NFC-e de lojas pequenas | alta | médio | busca por nome como caminho equivalente, não escondido |
| Navegador sem suporte a leitura de código de barras (Safari/Firefox sem `BarcodeDetector`) | média | médio | polyfill JS (doc 03) + busca por nome sempre visível como alternativa igual, não degradada |
| Mesmo GTIN com embalagens diferentes | baixa | médio | outliers por IQR + exibir descrição da loja |
| Reclamação de estabelecimento sobre preço exibido | baixa | médio | canal de contato + processo de correção documentado; o dado é fiscal e verificável |
| Servidor caseiro fora do ar (energia/internet residencial) | média | médio | aceito conscientemente para uso não comercial (doc 03, seção 9.6); sem SLA a cumprir com ninguém |

## 6. Perguntas em aberto

- Registrar marca no INPI? Só relevante se um dia o escopo mudar para algo público/
  comercial — para uso pessoal entre amigos, dispensável por ora.
- Modelo de receita? **Não se aplica** dado o uso não comercial — sem anúncios, sem
  cobrança. Revisitar só se o escopo do projeto mudar.

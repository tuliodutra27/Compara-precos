# Rascunho — pedido de acesso à API da Menor Preço Brasil

**Canal:** Fale Conosco da SEFAZ sobre o Menor Preço Brasil —
<https://s1-internet.sefaz.es.gov.br/faleconosco/categoria/Assunto/481/Cidad%C3%A3o/21>
(canal encontrado via SEFAZ-ES; como o backend é centralizado na Procergs/RS, vale
tentar também um canal equivalente do RS se houver — a confirmar).

**Expectativa realista:** este é um canal de atendimento ao cidadão, não um canal de
relacionamento com desenvolvedores. O processo formal de integração (Conecta gov.br)
exige entidade registrada com diretor de TI — não se aplica a projeto pessoal. Trate
esta mensagem como uma tentativa de baixo custo, não como um caminho garantido.

---

**Assunto:** Solicitação de informações sobre acesso programático à API do Menor Preço Brasil

Olá,

Estou desenvolvendo, como projeto pessoal e sem fins comerciais, um aplicativo web
(PWA) para consulta de preços de supermercado no Rio de Janeiro, com o objetivo de
uso próprio e de um pequeno grupo de amigos.

Ao investigar as fontes de dados disponíveis, identifiquei que o Menor Preço Brasil
consulta a API `https://mprs.sefaz.rs.gov.br/API/ConsultaMenorPrecoBrasil/api/v1/`,
protegida por autenticação via Login Único gov.br (OAuth2), e que o acesso exige um
token emitido para o client_id específico do aplicativo oficial.

Gostaria de saber:

1. Existe algum caminho para uma aplicação de terceiros (ainda que pessoal/não
   comercial) obter autorização para consultar essa API, com cada usuário autenticando
   com sua própria conta gov.br?
2. Existe alguma API pública de preços, sem essa exigência de autenticação, para
   consulta de dados já públicos (os preços vêm de notas fiscais eletrônicas, que são
   informação de caráter público)?
3. Há algum contato de relacionamento com desenvolvedores/parcerias técnicas da
   Procergs para esse tipo de pergunta, caso este não seja o canal certo?

Fico à disposição para mais detalhes sobre o projeto, se for útil.

Obrigado,
[seu nome]

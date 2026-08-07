# 08 — Jurídico, LGPD e riscos

> Este documento organiza os pontos de atenção e propõe encaminhamentos. **Não é
> parecer jurídico.** Antes do lançamento público, vale uma consulta rápida com
> advogado de direito digital — algumas horas resolvem tudo o que está aqui.

## 1. Uso dos dados das SEFAZ

**A favor:**

- Os preços vêm de documentos fiscais eletrônicos e são publicados pelos próprios
  estados em portais e apps **abertos ao cidadão, sem login**.
- Preço praticado por CNPJ não é dado pessoal — é informação comercial de pessoa
  jurídica, fora do escopo da LGPD.
- Há amparo na Lei de Acesso à Informação (Lei 12.527/2011) e na política de dados
  abertos (Decreto 8.777/2016) para o acesso a dados públicos.

**Contra / a verificar:**

- Os endpoints são **backends de apps oficiais, não APIs públicas documentadas**. Uso
  automatizado pode violar os termos de uso do portal — **ler cada ToS é a tarefa S0-4**.
- Volume alto de requisições pode ser tratado como abuso, independentemente do direito
  de acesso ao dado.

**Encaminhamentos:**

1. `User-Agent` honesto e identificável (`ComparaPrecos/1.0 (+https://site; contato@…)`)
   — nada de fingir ser navegador.
2. Cache agressivo: meta de ≤ 1 requisição por GTIN+região a cada 6 h.
3. Respeitar `robots.txt` e `Retry-After`; backoff exponencial em 429/503.
4. **Ofício para a SEFAZ do estado de lançamento** pedindo acesso oficial ou
   confirmação de que o uso é permitido. Custa um e-mail e transforma risco em
   parceria — vários estados respondem bem a apps que ampliam o alcance da política
   de transparência. Fazer isso ainda na Sprint 1, sem esperar resposta para seguir.
5. Sempre **creditar a fonte** na UI ("Dados: SEFAZ/PR — Menor Preço Nota Paraná") e
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
   localização, e o app precisa funcionar sem GPS (seletor manual de cidade).
4. **Política de privacidade** publicada em URL própria — obrigatória para a Play Store
   e para o formulário de Segurança de Dados.
5. Canal de contato do encarregado (pode ser o seu e-mail) na política.

Na **fase 2** (envio de notas fiscais) o cenário muda: a NFC-e pode conter CPF do
consumidor. Regra desde já: **descartar o CPF no momento do parsing**, antes de
qualquer gravação. Nunca persistir CPF.

## 3. Play Store / App Store

| Exigência | Como atender |
|---|---|
| Formulário "Segurança de dados" | declarar localização (não armazenada) e ID de dispositivo |
| Justificativa do uso de localização | descrição clara na ficha + tela de contexto in-app |
| Política de privacidade acessível | URL pública, também linkada dentro do app |
| Não simular app oficial de governo | nome, ícone e textos sem brasão, sem "SEFAZ", sem "gov" |

O item 4 é o de maior risco de reprovação. Evite qualquer elemento visual que sugira
origem governamental.

## 4. Isenção de responsabilidade (dentro do app)

Texto sugerido para a tela "De onde vêm os preços":

> Os preços exibidos vêm de notas fiscais eletrônicas autorizadas pelas Secretarias
> de Fazenda estaduais e se referem a vendas **já realizadas**. O valor cobrado hoje
> na loja pode ser diferente. Este aplicativo não tem vínculo com órgãos públicos nem
> com os estabelecimentos listados.

## 5. Riscos técnicos e de produto

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Endpoint não documentado muda de formato | alta | alto | adapter isolado + teste de contrato diário na CI + payload bruto guardado 7 dias |
| Bloqueio de IP pela SEFAZ | média | alto | cache, backoff, UA honesto, ofício oficial (seção 1.4) |
| Cobertura de dados baixa na região do usuário | média | alto | expansão automática de raio → município → UF, com aviso; base colaborativa na fase 2 |
| GTIN ausente na NFC-e de lojas pequenas | alta | médio | fallback para busca por nome |
| Mesmo GTIN com embalagens diferentes | baixa | médio | outliers por IQR + exibir descrição da loja |
| Reclamação de estabelecimento sobre preço exibido | baixa | médio | canal de contato + processo de correção documentado; o dado é fiscal e verificável |
| Concorrência com o app oficial gratuito | alta | médio | diferencial é UX + comparação multi-fonte + lista de compras (v2) |

## 6. Perguntas em aberto

- Registrar marca no INPI? Barato (~R$ 400) e evita dor de cabeça se o app crescer.
- Modelo de receita: anúncios discretos, versão pro sem anúncios, ou nada no MVP?
  Recomendação: **nada no MVP** — monetização antes de retenção mata produto novo.

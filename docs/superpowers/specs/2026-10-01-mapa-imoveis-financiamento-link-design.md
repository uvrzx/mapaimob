# Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7)

## Contexto

Usuário pediu pra testar "financiamento linkado com condomínio" — não
existia. Confirmado por grep: `financiamento.html` é carregado sem
parâmetro nenhum (`index.html` — `$('financingFrame').src =
'financiamento.html'`), sem `postMessage`, sem query string. A
calculadora (`mapa-imoveis/financiamento.html`) é 100% standalone —
estado só vive em memória do próprio iframe, inputs preenchidos à mão.

## O que entra

Botão "💰" em cada unidade da seção "Unidades agenciadas" do cartão de
prédio (popup de condomínio, fase 6). Clicar nele:

1. Troca pra aba Financiamento (mesmo efeito visual de clicar na aba).
2. Recarrega `financingFrame` com `?vi=<preço da unidade>` na URL.
3. A calculadora lê esse parâmetro no carregamento e já preenche o campo
   "Valor do imóvel (VI)" — o corretor não digita de novo um valor que
   já está no sistema.

**Preço usado**: `rentPrice` se a unidade for só aluguel, senão
`salePrice` (mesma regra de `priceLabelFor`, já usada em todo o app).

## O que não entra (decisão deliberada, não esquecimento)

- **Não** linka pelo condomínio como um todo — um condomínio tem várias
  unidades com preços diferentes, "financiamento do condomínio" não
  significa nada sozinho. O link é sempre por **unidade**.
- **Não** adiciona o botão na página de detalhe (`#detailView`) nem no
  popup de unidade avulsa no modo Mapa — pedido foi especificamente
  sobre o cartão de condomínio. Extensão natural se o cliente pedir
  depois (mesmo botão, mesma função `simulateFinancing`, só chamado de
  outro lugar).
- **Não** passa nome do cliente nem endereço pro simulador — o campo
  "Cliente" da calculadora é pro nome da PESSOA (comprador), não tem
  campo de endereço/imóvel nenhum; inventar um campo novo lá é escopo
  que ninguém pediu.
- **Não** preserva uma simulação em andamento — clicar "💰" numa unidade
  SEMPRE recarrega o iframe do zero com o novo VI. Como o simulador
  nunca salvou nada (sem IndexedDB, sem localStorage), isso não perde
  dado real — só reseta campos que o corretor talvez tivesse acabado de
  digitar pra outra proposta. Aceitável: é uma ação explícita do
  usuário ("simular ESSA unidade agora"), não algo que acontece sozinho.

## Como funciona

`index.html` ganha `showFinancingTab()` (extraído do handler de clique
da aba Financiamento, pra reusar sem duplicar a troca de view) e
`simulateFinancing(property)` / `simulateFinancingFor(id)` (monta a URL
com `vi`, seta no iframe, chama `showFinancingTab()`).

`financiamento.html` lê `location.search` uma vez, no carregamento: se
tiver `vi` numérico válido (`>0`), seta `state.VI` e o `value` do input
`[data-field="VI"]` (formatado igual ao blur de formatação que já
existe), antes da primeira pintura (`render()`).

## Segurança

`vi` é número (`encodeURIComponent` de um valor numérico, `parseFloat`
na leitura) — sem interpolação de texto livre em HTML, não precisa de
`escapeHtml`.

## Teste manual

- No popup de um condomínio com unidades agenciadas: clicar "💰" numa
  unidade com `salePrice` preenchido. Confirma: aba vira Financiamento,
  campo "Valor do imóvel" mostra o preço certo formatado
  (`1.234,56` etc), resto dos campos vazio/zerado.
- Unidade com `dealType: 'aluguel'` (preço em `rentPrice`): confirma que
  usa `rentPrice`, não `salePrice` (que pode estar 0).
- Clicar "💰" não também aciona `openPropertyDetail` da linha (o click
  tem que ficar só no botão, não propagar pro `onclick` da linha toda).
- Abrir a aba Financiamento manualmente (sem vir de um "💰") continua
  carregando sem nenhum parâmetro, campo VI vazio — comportamento atual
  preservado.
- Simular unidade A, depois simular unidade B sem fechar nada no meio:
  campo VI reflete o preço de B (recarregou do zero).
- Acessar `financiamento.html` direto (sem `index.html`, sem `?vi=`):
  continua funcionando exatamente como hoje — `vi` ausente não quebra
  nada.
- `financiamento.html?vi=abc` (não numérico) ou `?vi=-100` (negativo):
  campo VI fica vazio/zerado, sem erro no console.

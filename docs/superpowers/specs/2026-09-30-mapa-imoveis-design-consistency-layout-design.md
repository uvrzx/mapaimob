# Mapa de Imóveis — Consistência de Design e Layout 3 Colunas

## Contexto

Mini-fase fora da sequência numerada (fases 1-4 do redesign, já mergeadas).
Usuário reclamou de 3 problemas acumulados ao longo das fases anteriores:

1. O layout do mapa nunca foi reestruturado pra bater com a imagem de
   referência original da fase 1 (app estilo "uphome": filtro | lista de
   cards vertical rolável | mapa, em 3 colunas). Hoje a lista de imóveis é
   uma faixa horizontal rolável **abaixo** do mapa, não uma coluna ao lado.
2. "Fontes e botões feios e discrepantes" espalhados pelo app — toda fase
   anterior teve pelo menos 1 achado de review sobre botão sem o estilo
   certo. Falta um documento de referência único pro design system atual.
3. `mapa-imoveis/financiamento.html` (aba Financiamento, carregada via
   `<iframe>`) nunca foi retemado — continua 100% no tema escuro antigo
   (fundo quase preto, amarelo `#facc15` de destaque) que `index.html`
   tinha antes da fase 1 virar claro/indigo.

## `design.md` — documento de referência do design system

Novo arquivo na raiz do repo (`design.md`), fonte única de verdade pra
fonte, cores, radius/shadow e a regra de skin de botão. Conteúdo vem
direto do `:root` e das regras de botão de `mapa-imoveis/index.html` (sem
inventar valor — copiado do CSS real). Documenta:

- Fonte: Inter (Google Fonts, pesos 400/500/600/700/800), fallback
  `system-ui`.
- Tokens de cor do `:root` atual (`--bg`, `--panel`, `--panel-2`, `--line`,
  `--text`, `--muted`, `--accent`, `--accent-tint`, `--ok-*`, `--warn-*`).
- `--radius: 16px`, `--shadow-soft`.
- Regra de skin de botão — **primário**: `background:var(--accent);
  color:#fff`; **ghost**: `background:transparent; color:var(--muted);
  border:1px solid var(--line)` — com a lista de seletores que já seguem
  essa regra hoje (`header button`/`.ghost`, `#propertyForm button`/
  `.ghost`, `.property-popup-actions button`/`.ghost`, `.dash-main
  button`/`.dash-side button` e suas variantes `.ghost`), e a exceção
  conhecida (`.content-head button`, que só existe na variante ghost, sem
  par primário).

Não é código executável — é markdown. Mesmo assim vira task formal do
plano, com "teste" = grep confirmando que os valores documentados batem
com o CSS real de `index.html` (evita doc desatualizado no dia em que
nascer, mesmo problema que motivou pedir o doc).

## `financiamento.html` retemado pro claro/indigo

`financiamento.html` tem os MESMOS nomes de variável que `index.html`
tinha antes da fase 1 (`--bg`, `--panel`, `--panel-2`, `--line`, `--text`,
`--muted`, `--accent`, `--accent-2`, `--ok-*`, `--warn-*`, `--radius`,
`--mono`, `--sans`) — só os valores estão desatualizados. Retema trocando
pelos valores atuais de `index.html`.

**Decisão sobre `--accent-2`**: o palette atual de `index.html` colapsou o
antigo esquema de 2 acentos (azul + amarelo) num único `--accent` indigo —
não existe token equivalente pra `--accent-2`. Em vez de manter um alias
morto, os 4 usos de `var(--accent-2)` em `financiamento.html` (botão do
header, valor em destaque do painel vivo, valor em destaque do modo
apresentação ×2) passam a usar `var(--accent)` diretamente, e a declaração
`--accent-2` some do `:root`.

**Hardcoded fora do token**: o botão do header usa `color: #161300` (texto
quase-preto, hardcoded, pensado pra contrastar com o amarelo antigo) — a
mesma classe de bug que a fase 1 teve que corrigir em `index.html` (cor
hardcoded em vez de token). Vira `color: #fff`, seguindo a regra de botão
primário do `design.md`.

**Fonte**: `financiamento.html` hoje não carrega fonte externa nenhuma
(usa só `system-ui`). Como o objetivo desta task é consistência com o
resto do app, adiciona os mesmos `<link>` de Google Fonts do `index.html`
e prefixa `--sans` com `'Inter'`.

**Fora do escopo desta retemagem**: o bloco `@media print` (linhas ~132-
138) usa preto/branco/cinza hardcoded de propósito — é papel impresso, não
a UI em tela, não segue o tema claro/indigo nem precisa seguir. Não mexer
nisso.

Qualquer valor usado tem que bater EXATAMENTE com o token equivalente de
`index.html` — sem aproximar.

## Layout do mapa em 3 colunas

Estrutura atual (dentro de `#contentPanel`, que já é flex-column):
`.content-head` (topo) → `#mapWrap` (mapa + `#mapFilterCard` flutuante) →
`#listPanel` (`#listItems` em `display:flex` horizontal, `.list-item` com
`width:220px` fixo, scroll-snap `x`) — a lista é uma faixa embaixo do
mapa.

Vira: `.content-head` continua igual no topo. `#mapWrap` e `#listPanel`
passam a ficar dentro de uma nova linha flex (`.content-row`, única classe
nova) — `#listPanel` à ESQUERDA como coluna vertical rolável de largura
fixa (320px), `#mapWrap` à DIREITA com `flex:1` preenchendo o resto.
`#listItems` muda de `flex` horizontal pra `flex-direction:column`
vertical; `.list-item` muda de `width:220px` fixo pra `width:100%`
(preenche a coluna). Scroll-snap horizontal (fazia sentido pra faixa,
não pra coluna) é removido, não substituído por snap vertical — não foi
pedido.

Pra não arriscar a estrutura grande do `#mapFilterCard` (filtros dentro de
`#mapWrap`, ~40 linhas de HTML com Alpine), a ORDEM no HTML não muda —
`#mapWrap` continua antes de `#listPanel` no source. A posição visual
(lista à esquerda) vem só de `order: -1` no CSS de `#listPanel` dentro do
novo flex row. Zero risco de quebrar o filtro ao mover HTML.

IDs/classes existentes não mudam de nome — só de CSS/posição. `.hidden`
em `#listPanel` (toggle do botão "Ocultar lista") continua funcionando
sem tocar no JS: escondendo a coluna, `#mapWrap` com `flex:1` toma o
espaço todo, do mesmo jeito que já acontecia antes com o mapa tomando a
largura toda quando a faixa sumia.

## Controles do mapa nos cantos

Bate com a imagem de referência: ícone de camadas/satélite no canto
SUPERIOR direito (já é o padrão do Leaflet pro `L.control.layers`
existente — **nada muda aqui**), zoom no canto INFERIOR direito (hoje é o
padrão do Leaflet, `topleft` — precisa mover).

Em `initMap()`: `L.map('map', { zoomControl: false })` desliga o controle
padrão, e `L.control.zoom({ position: 'bottomright' }).addTo(map)` adiciona
o controle na posição nova. Mudança isolada de 2 linhas, sem efeito em
mais nada da função.

## Fora de escopo

- Qualquer campo novo no imóvel ou na calculadora de financiamento.
- Corrigir o `innerHTML` de endereço/apelido pré-existente no popup/lista/
  detalhe (dívida técnica de fases anteriores, não desta).
- Mexer no bloco `@media print` de `financiamento.html` (preto/branco de
  papel, não é tema de tela).
- Scroll-snap vertical na nova coluna de lista (não foi pedido).
- Mover a posição do `L.control.layers` (satélite) — já está no canto
  certo (`topright`, padrão do Leaflet).
- Qualquer arquivo além de `design.md` (novo), `mapa-imoveis/index.html` e
  `mapa-imoveis/financiamento.html`.

## Teste manual

- `design.md`: grep dos valores documentados (`:root`, regras de botão)
  contra `mapa-imoveis/index.html` — tudo bate.
- Abrir o app (`preview_start({name: "mapa-imoveis"})`): lista de imóveis
  aparece como coluna vertical à ESQUERDA do mapa, mapa ocupa o resto à
  direita. Cards da lista preenchem a largura da coluna (não mais 220px
  fixo lado a lado).
- Clicar "Ocultar lista": coluna some, mapa expande pra largura toda
  (`map.invalidateSize()` já disparado pelo handler existente). Clicar de
  novo: coluna volta.
- Zoom do Leaflet (`+`/`-`) aparece no canto INFERIOR direito do mapa.
  Controle de camadas (ícone de satélite) continua no canto SUPERIOR
  direito, sem mudança.
- Abrir `http://localhost:<porta>/financiamento.html` direto (mesma porta
  do `preview_start({name: "mapa-imoveis"})`, já que é servido da mesma
  pasta) e também pela aba "Financiamento" dentro do `index.html`: fundo
  claro, texto escuro, azul indigo (`#4b5fee`) como cor de destaque em vez
  do amarelo antigo, fonte Inter. Nenhum elemento com fundo/texto do tema
  escuro antigo sobrando. Botão "Apresentar ▸" com fundo indigo e texto
  branco.
- Calculadora de financiamento continua funcionando (preencher campos,
  ver painel vivo atualizar, abrir modo apresentação, fechar) — retema é
  só CSS, lógica de cálculo intocada.

# Design System — Mapa de Imóveis

Fonte única de verdade pro design system do app `mapa-imoveis`. Valores
copiados direto do `:root` e das regras de botão de
`mapa-imoveis/index.html` (tema claro, azul marinho + azul Mediterrâneo).
Qualquer outro arquivo do app (`financiamento.html` incluso) deve usar
esses mesmos valores — nunca aproximar.

## Fonte

Inter, via Google Fonts:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
```

Pesos usados em todo o app, sem exceção: 400 (texto), 600 (labels/
botões ghost), 700 (botões primários, títulos pequenos, destaques de
apresentação). `financiamento.html` usava 500 e 800 numa exceção não
documentada aqui — normalizado pra 600/700 e o `<link>` enxugado (não
carrega mais pesos que nada usa). Qualquer peso novo introduzido depois
tem que ser um desses dois, nunca um terceiro.

`--sans: 'Inter', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;`

## Cores (`:root`)

```css
--navy: #0b2545;
--bg: #f2f6fa;
--panel: #ffffff;
--panel-2: #e9f0f6;
--line: #d5e1ea;
--text: var(--navy);
--muted: #4a5f78;
--accent: #17708f;
--accent-tint: rgba(23,112,143,.12);
--ok-bg: #e7f9ee; --ok-fg: #15803d; --ok-line: #b7ecc8;
--warn-bg: #fdeaea; --warn-fg: #b91c1c; --warn-line: #f5c2c2;
--mid-fg: #b45309;
```

Paleta: marinho + branco, com azul Mediterrâneo (`--accent`) como único
acento de marca — ações primárias, seleção/estado ativo, links.
`--navy` é o texto e o fundo do CTA "+ Adicionar imóvel". Não existe
segundo acento de marca.

Cores semânticas de status (`--ok-*` verde, `--warn-*` vermelho,
`--mid-fg` âmbar escuro, usado no tier médio do badge de preenchimento do
condomínio) não são de marca e não mudam com o tema. As paletas de
`STATUS_COLORS`/`STATUS_TEXT_COLORS` e de categoria de POI (JS) também são
semânticas próprias e ficam fora dos tokens.

Contraste WCAG AA (≥ 4.5:1) conferido nos pares reais de texto/fundo:
navy/panel 15.4, navy/bg 14.2, navy/panel-2 13.4, muted/panel 6.6,
muted/bg 6.0, muted/panel-2 5.7, branco/accent 5.6, accent/panel 5.6,
accent/bg 5.2, accent/panel-2 4.9, accent sobre accent-tint (chip de tipo
ativo) 4.7, branco/navy 15.4, branco/ok-fg 5.0, branco/mid-fg 5.0,
branco/warn-fg 6.5. Recalcular se mexer em qualquer token (`--accent`
mais claro que `#17708f` ou `--muted` mais claro que `#4a5f78` reprovam).

## Radius / Shadow

```css
--radius: 16px;
--shadow-soft: 0 8px 24px rgba(20,24,50,.06);
```

## Monoespaçada (valores numéricos/técnicos)

```css
--mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
```

## Regra de skin de botão

**Primário** — ação principal (salvar, ver detalhes, apresentar, 💰):

```css
background: var(--accent);
color: #fff;
border: 0;
font-weight: 700;
cursor: pointer;
```

**CTA adicionar** — só `#btnAddProperty` ("+ Adicionar imóvel" no header):
`background: var(--navy); color: #fff;`. Estado `.active` (modo "marcar no
mapa") usa `var(--ok-fg)` + branco, igual ao `header button.active`.

**Ghost** — ação secundária (cancelar, ocultar, excluir):

```css
background: transparent;
color: var(--muted);
border: 1px solid var(--line);
font-weight: 600;
```

`padding`, `border-radius` e `font-size` variam por contexto (botão de
header é maior/mais arredondado que botão de popup) — só o par
cor-de-fundo/cor-de-texto/peso é fixo.

Seletores que já seguem essa regra hoje em `mapa-imoveis/index.html`:

- `header button` / `header button.ghost`
- `#propertyForm button` / `#propertyForm button.ghost`
- `.property-popup-actions button` / `.property-popup-actions button.ghost`
- `.dash-main button, .dash-side button` / `.dash-main button.ghost,
  .dash-side button.ghost`
- `#detailView > button, .detail-main button` / `#detailView > button.ghost,
  .detail-main button.ghost`

**Exceção conhecida**: `.content-head button` (botão "Ocultar/Mostrar
lista") só existe na variante ghost — não tem par primário, porque não é
uma ação "principal" de tela. Não é bug, é intencional.

Qualquer botão novo do app deve usar um desses padrões (ou a exceção
ghost-only, se fizer sentido pro contexto) — nunca cor hardcoded.

## Estado ativo/toggle (chips, abas, segmented controls)

Controles de seleção (não são "botões de ação", são toggle) usam fundo
sólido `var(--accent)` + texto branco quando ativos: `.chip.active`,
`.seg-group button.active`, `.view-tab.active` (`.type-btn.active` usa
`--accent-tint` + texto/ícone `--accent`). Exceção:
`header button.active` (usado só no botão "+ Adicionar imóvel" em modo
de marcar no mapa) usa `var(--ok-fg)` (verde) em vez de `var(--accent)` —
de propósito, sinaliza "ação em andamento", não seleção normal.

## Breakpoints e layout responsivo

Um único breakpoint estrutural por faixa; sem `@media` por componente
além destes (e do `@media (max-width: 1000px)` próprio de
`financiamento.html`):

- **≥ 1200px** — sidebar de filtros 320px + lista 320px + mapa.
- **900–1199px** — sidebar 280px; a lista (`#listPanel`) começa oculta
  (decidido por JS no load; "Mostrar lista" em `#btnToggleList`).
- **< 900px** — sidebar vira gaveta fixa à esquerda
  (`min(320px, 88vw)`, `translateX`, z-index 1200) com backdrop clicável
  (`#drawerBackdrop`), aberta por "Filtrar" no HUD do mapa; Esc e o "×"
  do cabeçalho de filtros também fecham. A lista vira painel sobreposto
  (z-index 1100) aberto por `#btnToggleList` e fecha ao escolher um item.
  O mapa sempre ocupa a largura inteira da área de conteúdo.
  `.dash-grid`/`.detail-grid` empilham em 1 coluna.
- **< 600px** — header compacto: sem título, abas ocupam a linha toda,
  Exportar/Importar/Adicionar viram ícones ao lado da busca. Header cabe
  em 2 linhas de 360px a 1199px.

**Header** (grid, 4 zonas no desktop ≥ 1200px): marca · abas · busca
centralizada (máx. 640px) · ações à direita (Exportar, Importar e, no
extremo direito, o CTA "+ Adicionar imóvel"). Abaixo de 1200px vira 2
linhas: marca + abas, depois busca + ações. Ações sempre alinhadas à
direita; nada de itens soltos à esquerda com vazio no resto da barra.

Regras fixas: sem scroll horizontal da página de 360px a 1920px; HUD
(`#mapHud`) e legenda de amenidades ficam dentro de `#mapWrap`; abaixo
de 900px a legenda começa recolhida atrás do chip "Amenidades".

## Filtros

`#filterPanel` mostra por padrão Negócio, Tipo de imóvel (grade de 3
colunas), Status, Preço e Quartos mín. O restante (banheiros, vagas,
suítes, área, mobiliado, pet, bairro, corretor, amenidades do condomínio)
fica num `<details>` "Mais filtros", fechado por padrão, com badge no
`<summary>` contando filtros avançados ativos
(`advancedFiltersActiveCount`). Cabeçalho "Filtros / Limpar" é sticky no
topo da sidebar.

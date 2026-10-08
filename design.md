# Design System — Mapa de Imóveis

Fonte única de verdade pro design system do app `mapa-imoveis`. Valores
copiados direto do `:root` e das regras de botão de
`mapa-imoveis/index.html` (tema midnight + dourado, mapa claro; linguagem de alto padrão).
Qualquer outro arquivo do app (`financiamento.html` incluso) deve usar
esses mesmos valores — nunca aproximar.

## Fonte

Geist (UI) + Cormorant Garamond (títulos), via Google Fonts (Inter banido):

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@500;600&family=Geist:wght@400;600;700&display=swap" rel="stylesheet">
```

Pesos usados em todo o app, sem exceção: 400 (texto), 600 (labels/
botões ghost), 700 (botões primários, títulos pequenos, destaques de
apresentação). `financiamento.html` usava 500 e 800 numa exceção não
documentada aqui — normalizado pra 600/700 e o `<link>` enxugado (não
carrega mais pesos que nada usa). Qualquer peso novo introduzido depois
tem que ser um desses dois, nunca um terceiro.

`--sans: 'Geist', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;`

## Cores (`:root`)

```css
color-scheme: dark;
--navy: #0C1527;
--bg: #060D1C; --panel: #0E1830; --panel-2: #1B2A4A; --line: #26324D;
--text: #F4F0E9; --muted: #A3AEC4;
--accent: #C2A375; --on-accent: #0A1020; --accent-text: #D4BC92; --accent-tint: rgba(194,163,117,.14);
--ok-bg: #0e3a2a; --ok-fg: #15803d; --ok-text: #86efac; --ok-line: #1f6b4a;
--warn-bg: #3d1521; --warn-fg: #b91c1c; --warn-text: #fca5a5; --warn-line: #7a2a3a;
--mid-fg: #b45309;
```

Referência visual: site de imobiliária de alto padrão (midnight quase preto,
ouro/areia, serifa nos títulos, botões em pílula dourada, cartões escuros com
borda fina). Ouro é o **único acento** e vale no app inteiro. Fundo da
página: `--bg` com dois brilhos radiais discretos nos cantos inferiores (ouro
à esquerda, azul à direita) em `.shell`.

**Preenchimento dourado usa `--on-accent` (texto escuro), nunca `#fff`.**
`--accent-text` (ouro claro) é a versão pra texto/ícone/traço sobre escuro.

**Mapa em tema claro (única exceção):** `#mapWrap` redefine os tokens
(`--bg #F1EDE6; --panel #FBF9F5; --panel-2 #ECE5D8; --line #E0D8CA; --text
#0C1527; --muted #55607A; --accent #0C1527; --on-accent #F4F0E9;
--accent-text #765B2B`) e tudo dentro (HUD, legenda, controles Leaflet,
popups, tooltip) herda. Tiles OSM sem filtro. Popup usa
`STATUS_TEXT_COLORS_MAP` (texto escuro nas pílulas). Popup/legenda seguem claros, mas todos os **botões do mapa** (HUD, zoom, camadas, "Amenidades", ações do popup) são midnight/dourado: re-declaram os tokens escuros no próprio elemento, sem branco.

HUD (`#mapHud`): pílula clara com borda `--line`; seletor
Unidade/Condomínio sem trilho (ativo = navy, inativo = `--muted`), "Filtrar"
em contorno. <600px alinha à esquerda pra não colidir com o controle de
camadas.

**Tema escuro (app todo, exceto o mapa):** midnight com texto bege-claro, pra reduzir luminosidade. Tipografia: títulos de tela/cartão em Cormorant Garamond 600 (`--serif`, numerais `lining-nums`), todo o resto em Geist; rótulos de campo em caixa alta com tracking (nos filtros, em Cormorant sem caixa alta). Exceções pedidas: rótulos e controles dos **filtros** e os **botões do mapa** também usam Cormorant (15px, 600, `lining-nums`). Tabelas e demais botões ficam em Geist.

Semânticas de status: `--ok-fg`/`--warn-fg`/`--mid-fg` são **fills sólidos**
(badge, botão ativo, barra) com texto branco; `--ok-text`/`--warn-text` são
as versões claras pra **texto** sobre `--ok-bg`/`--warn-bg`. As paletas de
`STATUS_COLORS`/`STATUS_TEXT_COLORS` (texto claro nas pílulas) e de
categoria de POI (JS) são semânticas próprias, fora dos tokens.

Contraste WCAG AA (≥ 4.5:1) medido: texto/panel 16.1, texto/panel-2 14.6, muted/panel 8.2, muted/panel-2 7.4, on-accent/ouro 7.9, accent-text/panel 9.9; mapa: texto/panel 17.3, muted/panel-2 5.4, accent-text/panel-2 5.5, ok-text/ok-bg 9.0.

## Radius / Shadow

```css
--radius: 16px;
--shadow-soft: 0 12px 32px rgba(0,0,0,.45); /* no mapa: 0 1px 2px + 0 6px 20px navy a .06/.10 */
```

## Números

Valores do financiamento usam Geist com `tabular-nums` (mesma fonte do mapa), não monoespaçada.

## Monoespaçada (legado, sem uso)

```css
--mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
```

## Regra de skin de botão

**Primário** — ação principal (salvar, ver detalhes, apresentar, 💰):

```css
background: var(--accent);
color: var(--on-accent);
border: 0;
font-weight: 700;
cursor: pointer;
```

**CTA adicionar**: `#btnAddProperty` segue o primário (pílula dourada). `.active` (modo "marcar no mapa") usa `var(--ok-fg)` + branco.

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
sólido `var(--accent)` + `var(--on-accent)` quando ativos: `.chip.active`,
`.seg-group button.active`, `.view-tab.active` (`.type-btn.active` também é dourado sólido). Exceção:
`header button.active` (usado só no botão "+ Adicionar imóvel" em modo
de marcar no mapa) usa `var(--ok-fg)` (verde) em vez de `var(--accent)` —
de propósito, sinaliza "ação em andamento", não seleção normal.

## Breakpoints e layout responsivo

Um único breakpoint estrutural por faixa; sem `@media` por componente
além destes (e do `@media (max-width: 1000px)` próprio de
`financiamento.html`):

- **≥ 1200px** — sidebar de filtros 320px + mapa; a lista de imóveis
  (`#listPanel`, 320px) **começa oculta em todas as larguras** (JS no load;
  "Mostrar lista" em `#btnToggleList`).
- **900–1199px** — sidebar 280px, mesma regra da lista.

**Navegação:** o app abre na aba **Mapa**; ordem das abas: Mapa · Painel de
vendas (antigo "Dashboard") · Financiamento.
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

## Forma e microcopy

Raio: **interativos em pílula** (`999px`: botões, chips, abas, busca, seletor
do HUD), **cartões 16px**, **inputs de formulário 12px**. Sem emoji na UI
(ícones de ação usam texto, ex. "R$" no botão de simular). Sem travessão (—)
em texto visível; vazio de dado usa "-".

## Superfícies sólidas, sem contorno

Botão, chip, campo e cartão **não usam borda** pra se definir. A hierarquia é
tonal: `--bg` (página) < `--panel` (sidebar, header, cartões, formulário) <
`--panel-2` (botão secundário, chip, campo, aba). O contraste vem do texto
(`--text` sobre `--panel-2`), nunca de uma linha. Botão secundário ("ghost")
é preenchido com `--panel-2`, não transparente com contorno. Linhas só em
tabela e lista de dados (`border-bottom` de linha). Foco de teclado segue
com `outline` dourado.

"Limpar" (filtros) é pílula sólida e só aparece quando há filtro ativo
(`hasFilters()`); ao clicar zera os filtros e some.

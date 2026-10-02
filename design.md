# Design System — Mapa de Imóveis

Fonte única de verdade pro design system do app `mapa-imoveis`. Valores
copiados direto do `:root` e das regras de botão de
`mapa-imoveis/index.html` (tema escuro, azul marinho + azul Mediterrâneo).
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
color-scheme: dark;
--navy: #0b2545;
--bg: #071a33; --panel: #0b2545; --panel-2: #123560; --line: #27507f;
--text: #f4f8fc; --muted: #a9bfd8;
--accent: #1b7fa3; --accent-text: #5bc0de; --accent-tint: rgba(91,192,222,.16);
--ok-bg: #0e3a2a; --ok-fg: #15803d; --ok-text: #86efac; --ok-line: #1f6b4a;
--warn-bg: #3d1521; --warn-fg: #b91c1c; --warn-text: #fca5a5; --warn-line: #7a2a3a;
--mid-fg: #b45309;
```

**Tema escuro (único tema):** fundo azul marinho, texto branco, pra reduzir
o impacto de luminosidade. O azul Mediterrâneo é o único acento de marca,
em duas versões: `--accent` (**preenchimento**, sempre com texto branco —
botões primários, aba/chip ativo) e `--accent-text` (**texto/ícone/traço**
sobre fundo escuro — links, estados hover, ícones de amenidade). Nunca use
`--accent` como cor de texto sobre o painel (3.4:1, reprova).

Semânticas de status: `--ok-fg`/`--warn-fg`/`--mid-fg` são **fills sólidos**
(badge, botão ativo, barra) com texto branco; `--ok-text`/`--warn-text` são
as versões claras pra **texto** sobre `--ok-bg`/`--warn-bg`. As paletas de
`STATUS_COLORS`/`STATUS_TEXT_COLORS` (texto claro nas pílulas) e de
categoria de POI (JS) são semânticas próprias, fora dos tokens.

Mapa: tiles do OpenStreetMap escurecidos por CSS (`.osm-tiles`, filtro
invert + hue-rotate) só na camada base — a camada de satélite não é afetada.
Controles/tooltip/atribuição do Leaflet seguem os tokens.

Contraste WCAG AA (≥ 4.5:1) medido nos pares reais: texto/panel 14.4,
muted/panel-2 6.5, branco/accent (aba ativa) 4.55, navy/accent-text (CTA)
7.4, muted/panel 11.1, accent-text/panel-2 5.1, branco/ok-fg 5.0,
branco/warn-fg 6.5, ok-text/ok-bg 9.0, warn-text/warn-bg 8.3. `--accent`
mais claro que `#1b7fa3` reprova com texto branco; recalcular ao mexer.

## Radius / Shadow

```css
--radius: 16px;
--shadow-soft: 0 8px 24px rgba(0,0,0,.4);
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
`background: var(--accent-text); color: var(--navy);` (destaca sobre o header
navy). Estado `.active` (modo "marcar no
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

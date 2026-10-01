# Design System — Mapa de Imóveis

Fonte única de verdade pro design system do app `mapa-imoveis`. Valores
copiados direto do `:root` e das regras de botão de
`mapa-imoveis/index.html` (fase 2 do redesign, tema claro + azul + amarelo).
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
--bg: #f5f6fa;
--panel: #ffffff;
--panel-2: #f0f1f6;
--line: #e6e8f0;
--text: #14161f;
--muted: #4b5563;
--accent: #4b5fee;
--accent-tint: rgba(75,95,238,.12);
--accent-2: #facc15;
--ok-bg: #e7f9ee; --ok-fg: #15803d; --ok-line: #b7ecc8;
--warn-bg: #fdeaea; --warn-fg: #b91c1c; --warn-line: #f5c2c2;
```

Dois acentos: `--accent` (azul, indigo) é o principal — ações, seleção,
estado ativo. `--accent-2` (amarelo) é secundário — só pra destaque
pontual (hoje: fundo do botão "+ Adicionar imóvel" no header). Nunca usar
`--accent-2` pra texto sobre fundo claro (baixo contraste); sempre como
fundo sólido com `var(--text)` por cima, ou como acento de borda/ícone.

`--muted`, `--ok-fg` e `--warn-fg` foram escurecidos na fase 2 (eram
`#8a8f9c`/`#1aa34a`/`#e5484d`) — as versões antigas ficavam abaixo de
4.5:1 de contraste (WCAG AA) contra `--panel`/`--panel-2`/`--bg`. Não
reverter pros valores antigos sem recalcular contraste (`--muted` em
especial precisa passar em 4.5:1 mesmo contra `--panel-2`, que é mais
escuro que `--panel`, não só contra branco puro).

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

**Primário** — ação principal (salvar, ver detalhes, apresentar):

```css
background: var(--accent);
color: #fff;
border: 0;
font-weight: 700;
cursor: pointer;
```

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

**Exceção conhecida 2**: `#btnAddProperty` ("+ Adicionar imóvel" no
header) usa `var(--accent-2)` (amarelo) + `var(--text)` em vez do par
primário azul/branco — é o único CTA de destaque amarelo do app, de
propósito (ver seção Cores). O estado `.active` (modo "marcar no mapa")
continua usando `var(--ok-fg)` (verde) + branco, igual ao
`header button.active` genérico.

Qualquer botão novo do app deve usar um desses 2 padrões (ou a exceção
ghost-only, se fizer sentido pro contexto) — nunca cor hardcoded.

## Estado ativo/toggle (chips, abas, segmented controls)

Controles de seleção (não são "botões de ação", são toggle) usam fundo
sólido `var(--accent)` + texto branco quando ativos: `.chip.active`,
`.seg-group button.active`, `.view-tab.active`, `.type-btn.active`. Exceção:
`header button.active` (usado só no botão "+ Adicionar imóvel" em modo
de marcar no mapa) usa `var(--ok-fg)` (verde) em vez de `var(--accent)` —
de propósito, sinaliza "ação em andamento", não seleção normal.

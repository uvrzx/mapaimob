# Design System — Mapa de Imóveis

Fonte única de verdade pro design system do app `mapa-imoveis`. Valores
copiados direto do `:root` e das regras de botão de
`mapa-imoveis/index.html` (fase 1 do redesign, tema claro/indigo).
Qualquer outro arquivo do app (`financiamento.html` incluso) deve usar
esses mesmos valores — nunca aproximar.

## Fonte

Inter, via Google Fonts:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

Pesos usados: 400 (texto), 600 (labels/botões ghost), 700 (botões
primários, títulos pequenos). 500 e 800 são carregados mas não usados
hoje em `index.html` — disponíveis pra destaques futuros.

`--sans: 'Inter', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;`

## Cores (`:root`)

```css
--bg: #f5f6fa;
--panel: #ffffff;
--panel-2: #f0f1f6;
--line: #e6e8f0;
--text: #14161f;
--muted: #8a8f9c;
--accent: #4b5fee;
--accent-tint: rgba(75,95,238,.12);
--ok-bg: #e7f9ee; --ok-fg: #1aa34a; --ok-line: #b7ecc8;
--warn-bg: #fdeaea; --warn-fg: #e5484d; --warn-line: #f5c2c2;
```

Um único acento (`--accent`, indigo) — não existe `--accent-2` no tema
atual. Se algum arquivo antigo ainda referenciar um segundo acento
(amarelo/dourado), é tema pré-fase-1 e precisa ser retemado, não copiado.

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

Qualquer botão novo do app deve usar um desses 2 padrões (ou a exceção
ghost-only, se fizer sentido pro contexto) — nunca cor hardcoded.

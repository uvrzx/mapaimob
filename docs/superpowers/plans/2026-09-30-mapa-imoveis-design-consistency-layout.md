# Mapa de Imóveis — Consistência de Design e Layout 3 Colunas — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Documentar o design system atual num `design.md` de referência,
retemar `financiamento.html` pro claro/indigo (hoje ainda no tema escuro
antigo pré-fase-1), mover o controle de zoom do Leaflet pro canto inferior
direito, e reestruturar o layout do mapa em 3 colunas (filtro embutido no
mapa | lista vertical rolável | mapa) batendo com a imagem de referência
original da fase 1.

**Architecture:** 4 tasks independentes entre si (nenhuma depende do HTML/
CSS que outra produz) — só compartilham os 2 arquivos-fonte. Task 1 cria
`design.md` (arquivo novo, doc apenas). Task 2 mexe só em
`financiamento.html`. Tasks 3 e 4 mexem em `mapa-imoveis/index.html`, em
regiões diferentes (JS de `initMap()` vs. HTML/CSS de
`#contentPanel`/`#mapWrap`/`#listPanel`) — sem sobreposição de linhas.

**Tech Stack:** Mesmo stack — HTML/CSS/JS vanilla + Alpine.js + Leaflet,
sem framework de teste, sem dependência nova. Verificação via Browser pane
(`preview_start({name: "mapa-imoveis"})`) + leitura visual/`read_page`.

## Global Constraints

- Escopo de arquivo por task: Task 1 só cria `design.md` (raiz do repo).
  Task 2 só edita `mapa-imoveis/financiamento.html`. Tasks 3 e 4 só editam
  `mapa-imoveis/index.html`. Nenhuma task cria arquivo além do que está
  explicitamente listado nela. Sem dependência nova.
- IDs e classes HTML **existentes** não podem mudar de nome — só de CSS
  ou posição no layout. Classes novas (ex.: `.content-row` na Task 4) são
  permitidas.
- Qualquer valor de cor/fonte usado em `financiamento.html` (Task 2) tem
  que bater EXATAMENTE com o token equivalente de `mapa-imoveis/
  index.html` — não aproximar, copiar o valor real.
- **Fique ESTRITAMENTE dentro do que cada task pede.** Aprendizado das 4
  fases anteriores: subagente que mexe fora do escopo do brief (refatora
  algo "já que estava ali", ajusta um estilo não pedido, etc.) causa
  retrabalho de review. Se notar algo fora de escopo que parece errado,
  **não mexa** — comente no relatório da task e segue.
- Sem tocar no bloco `@media print` de `financiamento.html` (Task 2) —
  cores de papel impresso, não fazem parte do tema de tela.
- Sem mexer no `L.control.layers` (satélite) do Leaflet (Task 3) — já
  está no canto certo por padrão do Leaflet (`topright`).

---

### Task 1: `design.md` — documento de referência do design system

**Files:**
- Create: `design.md` (raiz do repo, `C:\Users\lyncy\Claude\Projects\Projetos Imob\design.md`)

**Interfaces:**
- Produces: `design.md` (documento markdown, sem interface de código —
  referência lida por humanos/agentes em fases futuras).
- Consumes: `:root` e regras de botão (`header button`, `#propertyForm
  button`, `.property-popup-actions button`, `.dash-main button`,
  `.dash-side button`, e as variantes `.ghost` de cada uma) de
  `mapa-imoveis/index.html` — só leitura, nenhuma linha de `index.html`
  muda nesta task.

- [ ] **Step 1: Criar `design.md` com o conteúdo completo abaixo**

Arquivo novo — sem "Old" (não existe ainda). Conteúdo completo:

```markdown
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

Pesos usados: 400 (texto), 500, 600 (labels/botões ghost), 700 (botões
primários, títulos pequenos), 800 (não usado hoje em `index.html`, mas
carregado — disponível pra destaques futuros).

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

**Exceção conhecida**: `.content-head button` (botão "Ocultar/Mostrar
lista") só existe na variante ghost — não tem par primário, porque não é
uma ação "principal" de tela. Não é bug, é intencional.

Qualquer botão novo do app deve usar um desses 2 padrões (ou a exceção
ghost-only, se fizer sentido pro contexto) — nunca cor hardcoded.
```

- [ ] **Step 2: Teste — grep confirmando que o doc bate com o CSS real**

Via Bash, na raiz do repo:

```bash
grep -A12 ':root {' mapa-imoveis/index.html
grep -E "header button|#propertyForm button|\.property-popup-actions button|\.dash-main button|\.dash-side button" mapa-imoveis/index.html
```

Confirmar visualmente que cada valor de cor/radius/shadow e cada regra de
botão do `grep` bate exatamente com o que foi escrito em `design.md` no
Step 1 (mesmos hex, mesmas regras `background`/`color`/`border`/
`font-weight` pros pares primário/ghost). Qualquer divergência = corrigir
`design.md`, não o CSS (esta task não toca em `index.html`).

- [ ] **Step 3: Commit**

```bash
git add design.md
git commit -m "docs: design system de referencia do mapa-imoveis (fontes, cores, regra de botao)"
```

---

### Task 2: Retemar `financiamento.html` pro claro/indigo

**Files:**
- Modify: `mapa-imoveis/financiamento.html` (`<head>`: link de fonte;
  CSS: `:root` e os 4 usos de `var(--accent-2)`)

**Interfaces:**
- Consumes: `design.md` (Task 1) como referência dos valores corretos —
  não precisa esperar a Task 1 terminar pra rodar (os valores já estão
  documentados abaixo, copiados do mesmo `index.html`), só usa o mesmo
  vocabulário de tokens.
- Produces: nenhuma interface nova — é retemagem visual pura, a engine de
  cálculo (`compute`, `render`, `state`) não muda.

- [ ] **Step 1: `<head>` — adicionar fonte Inter antes do `<style>`**

Old:
```html
<title>Calculadora de Proposta — Corretor GV</title>
<style>
```
New:
```html
<title>Calculadora de Proposta — Corretor GV</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
<style>
```

- [ ] **Step 2: `:root` — trocar pros valores atuais de `index.html`, remover `--accent-2`**

Old:
```css
  /* Paleta alinhada ao mapa-imoveis/index.html (preto/azul/amarelo, dark). */
  :root {
    --bg: #050609;
    --panel: #12141c;
    --panel-2: #1b1e29;
    --line: #2a2e3d;
    --text: #eef0f5;
    --muted: #8b92a5;
    --accent: #2f6fed;
    --accent-2: #facc15;
    --ok-bg: #10361f; --ok-fg: #4ade80; --ok-line:#1f6b3a;
    --warn-bg: #3a1717; --warn-fg: #ff6b6b; --warn-line:#7a2a2a;
    --radius: 10px;
    --mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  }
```
New:
```css
  /* Paleta alinhada ao mapa-imoveis/index.html (claro/indigo — fase 1 do redesign). */
  :root {
    --bg: #f5f6fa;
    --panel: #ffffff;
    --panel-2: #f0f1f6;
    --line: #e6e8f0;
    --text: #14161f;
    --muted: #8a8f9c;
    --accent: #4b5fee;
    --ok-bg: #e7f9ee; --ok-fg: #1aa34a; --ok-line: #b7ecc8;
    --warn-bg: #fdeaea; --warn-fg: #e5484d; --warn-line: #f5c2c2;
    --radius: 16px;
    --mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
    --sans: 'Inter', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  }
```

Nota: `--accent-2` não existe mais — o tema atual usa um único acento. Os
4 usos de `var(--accent-2)` nos próximos steps passam a usar
`var(--accent)`.

- [ ] **Step 3: Botão do header — cor hardcoded `#161300` vira skin de botão primário**

Old:
```css
  header button {
    background: var(--accent-2); color: #161300; border: 0; padding: 9px 16px;
    border-radius: 20px; font-weight: 700; cursor: pointer; font-size: 14px;
  }
```
New:
```css
  header button {
    background: var(--accent); color: #fff; border: 0; padding: 9px 16px;
    border-radius: 20px; font-weight: 700; cursor: pointer; font-size: 14px;
  }
```

- [ ] **Step 4: Valor em destaque do painel vivo**

Old:
```css
  .live .row.big .v { font-size: 20px; color: var(--accent-2); }
```
New:
```css
  .live .row.big .v { font-size: 20px; color: var(--accent); }
```

- [ ] **Step 5: Valor em destaque do topo do modo apresentação**

Old:
```css
  .present-head .vi { font-size: clamp(40px, 6vw, 72px); font-weight: 800; color: var(--accent-2); font-family: var(--mono); line-height: 1.1; }
```
New:
```css
  .present-head .vi { font-size: clamp(40px, 6vw, 72px); font-weight: 800; color: var(--accent); font-family: var(--mono); line-height: 1.1; }
```

- [ ] **Step 6: Valor em destaque do rodapé do modo apresentação**

Old:
```css
  .present-foot .v { font-size: clamp(28px, 4vw, 40px); font-weight: 800; font-family: var(--mono); color: var(--accent-2); }
```
New:
```css
  .present-foot .v { font-size: clamp(28px, 4vw, 40px); font-weight: 800; font-family: var(--mono); color: var(--accent); }
```

- [ ] **Step 7: Teste manual — tema claro aplicado, cálculo intocado**

Via Browser pane (`preview_start({name: "mapa-imoveis"})`, depois
`navigate` pra `http://localhost:<porta>/financiamento.html` direto — sem
precisar passar pelo `index.html`, já que é a mesma pasta/servidor):
confirmar fundo claro (`--bg: #f5f6fa`), texto escuro, botão "Apresentar
▸" com fundo indigo (`#4b5fee`) e texto branco (não mais amarelo/preto).
Grep rápido pra confirmar que não sobrou `--accent-2` nem `#161300` nem
`#050609`/`#facc15` no arquivo (`grep -n "accent-2\|#161300\|#050609\|#facc15" mapa-imoveis/financiamento.html`
deve retornar vazio, exceto se aparecerem dentro do bloco `@media print`
— não deveriam, já que esses valores não existiam lá). Preencher os
campos de "Valor do imóvel" e "Financiamento aprovado", confirmar que o
painel vivo à direita atualiza os valores em tempo real (engine de
cálculo não foi tocada, só CSS). Abrir modo apresentação (botão
"Apresentar ▸"), confirmar que os cards também estão no tema claro, e
fechar (Esc ou botão "✕ Fechar"). Repetir a checagem visual abrindo a aba
"Financiamento" de dentro do `index.html` (`preview_start({name:
"mapa-imoveis"})` → clicar na aba "Financiamento") pra confirmar que o
iframe também renderiza no tema novo.

- [ ] **Step 8: Commit**

```bash
git add mapa-imoveis/financiamento.html
git commit -m "fix: retema financiamento.html do escuro antigo para o claro/indigo da fase 1"
```

---

### Task 3: Controle de zoom do Leaflet no canto inferior direito

**Files:**
- Modify: `mapa-imoveis/index.html` (JS: `initMap()`)

**Interfaces:**
- Consumes: `map` (instância global do Leaflet, já existente), `MAP_CENTER`,
  `MAP_ZOOM` (já existentes).
- Produces: nenhuma função nova — só muda a configuração do controle de
  zoom padrão do Leaflet dentro de `initMap()`.

- [ ] **Step 1: `L.map()` desliga o zoom control padrão, novo `L.control.zoom` no canto inferior direito**

Old:
```js
function initMap() {
  map = L.map('map').setView(MAP_CENTER, MAP_ZOOM);
```
New:
```js
function initMap() {
  map = L.map('map', { zoomControl: false }).setView(MAP_CENTER, MAP_ZOOM);
  L.control.zoom({ position: 'bottomright' }).addTo(map);
```

- [ ] **Step 2: Teste manual — zoom no canto certo, camadas sem mudança**

Via Browser pane (`preview_start({name: "mapa-imoveis"})`): no mapa,
confirmar visualmente (screenshot ou `read_page`) que os botões `+`/`-`
de zoom do Leaflet aparecem no canto INFERIOR direito do `#mapWrap`, não
mais no superior esquerdo. Confirmar que o ícone de camadas (toggle
"Satélite (2016)", `L.control.layers`) continua no canto SUPERIOR
direito, sem mudança de posição. Clicar `+`/`-` e confirmar que o zoom
ainda funciona normalmente. Arrastar o mapa e reabrir o painel de filtros
(`#mapFilterCard`, canto superior esquerdo) — confirmar que nada
sobrepõe o novo controle de zoom.

- [ ] **Step 3: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "fix: move controle de zoom do Leaflet para o canto inferior direito"
```

---

### Task 4: Layout do mapa em 3 colunas (lista vertical à esquerda)

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML de `#contentPanel`; CSS de
  `#contentPanel`/`#mapWrap`/`#listPanel`/`#listItems`/`.list-item`)

**Interfaces:**
- Consumes: `#mapWrap`, `#listPanel`, `#listItems`, `.list-item` (IDs/
  classes já existentes, sem mudar de nome). Handler de `#btnToggleList`
  (já existente, alterna `.hidden` em `#listPanel` + `map.invalidateSize()`)
  — não precisa mudar, continua funcionando com o novo layout.
- Produces: `.content-row` (classe CSS nova, wrapper flex que organiza
  `#mapWrap` e `#listPanel` lado a lado).

- [ ] **Step 1: HTML — abrir `.content-row` logo após `.content-head`, antes de `#mapWrap`**

Old:
```html
    </div>
    <div id="mapWrap">
```
New:
```html
    </div>
    <div class="content-row">
    <div id="mapWrap">
```

- [ ] **Step 2: HTML — fechar `.content-row` depois de `#listPanel`**

Old:
```html
    <div id="listPanel"><div id="listItems"></div></div>
  </main>
```
New:
```html
    <div id="listPanel"><div id="listItems"></div></div>
    </div>
  </main>
```

Nota: a ORDEM no HTML não muda — `#mapWrap` continua antes de
`#listPanel` no source (evita reescrever o bloco grande de
`#mapFilterCard`/Alpine que fica dentro de `#mapWrap`). A posição visual
(lista à esquerda) vem só do CSS no Step 5 (`order: -1`).

- [ ] **Step 3: CSS — `.content-row` como linha flex, logo após `.content-head button:hover`**

Old:
```css
  .content-head button:hover { border-color: var(--accent); color: var(--text); }
  #listPanel.hidden { display: none; }
```
New:
```css
  .content-head button:hover { border-color: var(--accent); color: var(--text); }
  .content-row { flex: 1; min-height: 0; display: flex; }
  #listPanel.hidden { display: none; }
```

- [ ] **Step 4: CSS — `#mapWrap` perde a margem esquerda (a coluna da lista assume esse espaço)**

Old:
```css
  #mapWrap {
    flex: 1; min-height: 0; position: relative; isolation: isolate;
    margin: 14px 20px; border-radius: 16px; overflow: hidden; border: 1px solid var(--line);
  }
```
New:
```css
  #mapWrap {
    flex: 1; min-height: 0; position: relative; isolation: isolate;
    margin: 14px 20px 14px 14px; border-radius: 16px; overflow: hidden; border: 1px solid var(--line);
  }
```

- [ ] **Step 5: CSS — `#listPanel`/`#listItems`/`.list-item` de faixa horizontal pra coluna vertical**

Old:
```css
  /* lista de imóveis (strip horizontal de cards abaixo do mapa) */
  #listPanel { flex-shrink: 0; padding: 0 20px 18px; }
  #listItems { display: flex; gap: 14px; overflow-x: auto; padding-bottom: 4px; scroll-snap-type: x proximity; scroll-behavior: smooth; }
  .list-item {
    display: flex; flex-direction: column; width: 220px; flex-shrink: 0; scroll-snap-align: start;
    border-radius: 14px; overflow: hidden; background: var(--panel-2); border: 1px solid var(--line);
    cursor: pointer; transition: transform .15s ease, border-color .15s ease;
  }
  .list-item:hover { transform: translateY(-3px); border-color: var(--accent); }
  .list-item:active { transform: translateY(-1px) scale(.97); }
  .list-item-photo { width: 100%; height: 120px; object-fit: cover; flex-shrink: 0; background: var(--panel); }
```
New:
```css
  /* lista de imóveis (coluna vertical rolável à esquerda do mapa) */
  #listPanel { flex-shrink: 0; width: 320px; order: -1; padding: 14px 0 18px 20px; overflow-y: auto; }
  #listItems { display: flex; flex-direction: column; gap: 14px; }
  .list-item {
    display: flex; flex-direction: column; width: 100%;
    border-radius: 14px; overflow: hidden; background: var(--panel-2); border: 1px solid var(--line);
    cursor: pointer; transition: transform .15s ease, border-color .15s ease;
  }
  .list-item:hover { transform: translateY(-3px); border-color: var(--accent); }
  .list-item:active { transform: translateY(-1px) scale(.97); }
  .list-item-photo { width: 100%; height: 120px; object-fit: cover; flex-shrink: 0; background: var(--panel); }
```

- [ ] **Step 6: Teste manual — 3 colunas, cards preenchendo a largura, toggle de lista intacto**

Via Browser pane (`preview_start({name: "mapa-imoveis"})`): confirmar que
`#listPanel` aparece como coluna vertical à ESQUERDA do `#mapWrap`
(`read_page` ou screenshot — `#listPanel` e `#mapWrap` lado a lado, não
mais empilhados). Rolar a lista verticalmente dentro da coluna (não mais
scroll horizontal). Confirmar que cada `.list-item` preenche a largura
da coluna (320px menos padding), não mais cards de 220px lado a lado.
Clicar "Ocultar lista" (`#btnToggleList`): coluna some
(`getComputedStyle($('listPanel')).display === 'none'`), `#mapWrap` se
expande pra preencher o espaço todo, `map.invalidateSize()` já dispara
(handler existente, não mexido nesta task). Clicar de novo: coluna volta
na posição esquerda. Clicar num `.list-item` e confirmar que ainda abre o
popup/detalhe do imóvel certo no mapa (comportamento de clique não foi
tocado, só CSS).

- [ ] **Step 7: Rodar self-check no console**

`read_console_messages()` — sem `ASSERT FAIL` (self-checks existentes do
app não dependem de layout/CSS, mas confirma que a página carregou sem
erro de JS).

- [ ] **Step 8: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: layout do mapa em 3 colunas, lista de imoveis como coluna vertical a esquerda"
```

---

## Self-Review

**Cobertura do spec:**
- `design.md` de referência (fonte, cores, radius/shadow, regra de botão
  + seletores que já seguem) → Task 1. ✅
- Retema `financiamento.html` pro claro/indigo, incluindo o hardcoded
  `#161300` e o `--accent-2` sem equivalente → Task 2. ✅
- Fonte Inter em `financiamento.html` pra bater com o resto do app →
  Task 2 Step 1. ✅
- Controle de zoom do Leaflet no canto inferior direito → Task 3. ✅
- Layout 3 colunas (lista vertical rolável à esquerda, mapa à direita) →
  Task 4. ✅
- `#listPanel.hidden` continua funcionando sem tocar no JS do botão →
  confirmado no Self-Review da Task 4 (Step 3 do plano: `order:-1` +
  `flex:1` em `#mapWrap` cobrem o caso, handler de `#btnToggleList` não
  foi listado em nenhum "Files" de nenhuma task). ✅
- Fora de escopo (campo novo, `innerHTML` pré-existente, `@media print`,
  scroll-snap vertical, mover `L.control.layers`) → nenhuma task toca
  nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo; Task 1 (arquivo
novo) traz o conteúdo completo do `design.md` em vez de Old/New, já que
não existe versão anterior do arquivo pra diffar contra.

**Consistência de nomes:** `.content-row` é criada uma vez (Task 4 Step 3)
e referenciada no HTML dos Steps 1-2 da mesma task, sem redeclarar em
outro lugar. `--accent-2` é removida do `:root` (Task 2 Step 2) e todos os
4 usos restantes (`header button`, `.live .row.big .v`, `.present-head
.vi`, `.present-foot .v`) são atualizados na mesma task (Steps 3-6) — sem
deixar nenhuma referência órfã a uma variável indefinida.

**Independência das tasks:** Task 1 só cria arquivo novo. Task 2 só edita
`financiamento.html`. Tasks 3 e 4 editam `index.html` em regiões
disjuntas — Task 3 só dentro de `initMap()` (por volta da linha 864),
Task 4 só no HTML/CSS de `#contentPanel` (linhas ~121-228 e ~362-407) —
sem overlap de linha, podem rodar em qualquer ordem ou em paralelo.

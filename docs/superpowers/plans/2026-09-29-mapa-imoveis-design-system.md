# Mapa de Imóveis — Design System Overhaul (Fase 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Retema `mapa-imoveis/index.html` do escuro preto/azul/amarelo para o tema
claro azul/indigo da referência (imagem "uphome"), com sidebar e cartão
flutuante do mapa recolhíveis, ícones de pin distintos por tipo de imóvel, e
formulário de adicionar/editar reorganizado em seções com rodapé fixo.

**Architecture:** Todo o trabalho é dentro de um único arquivo HTML
(`mapa-imoveis/index.html`, ~1330 linhas: `<style>` + `<script>` inline, sem
build step). Cada task edita uma região isolada do arquivo (tokens CSS,
ícones JS, um componente de UI por vez) e é verificável sozinha rodando o
app num servidor local.

**Tech Stack:** HTML/CSS/JS vanilla + Alpine.js 3 (CDN) + Leaflet 1 (CDN).
Sem framework de teste — verificação via `grep` (checks estáticos) e via
Browser pane (`preview_start` com a config `mapa-imoveis` já existente em
`.claude/launch.json`, que serve o diretório com `http-server`) + leitura de
`console.assert` (padrão de self-check já usado no arquivo).

## Global Constraints

- Arquivo único: `mapa-imoveis/index.html`. Não criar novos arquivos nem
  dependências.
- Sem dark mode / toggle de tema — troca é total e definitiva pro tema claro.
- Nenhum campo novo no formulário nesta fase (isso é fase 2, fora de escopo).
- IDs e names de campos existentes (`f_*`, `c_*`) não podem mudar — outro
  código (`readPropertyForm`, `openPropertyForm`) depende deles.
- Toda lógica não-trivial nova ganha um `console.assert` de self-check,
  seguindo o padrão já usado em `selfCheckLogic()`.
- IndexedDB não funciona em `file://` — testar sempre via
  `preview_start({name: "mapa-imoveis"})`, nunca abrindo o arquivo direto.
- Ícones SVG seguem o padrão já estabelecido: sem atributo `fill` explícito
  no path (herdam de `<g fill="...">` do pai no pin, ou de `fill:` CSS nos
  outros usos) — ver `ICONS.house`/`ICONS.building`/`ICONS.store` como
  referência de estilo.

---

### Task 1: Retema (tokens CSS, fonte Inter, remoção do amarelo)

**Files:**
- Modify: `mapa-imoveis/index.html` (bloco `<head>` e `<style>` apenas)

**Interfaces:**
- Produces: variáveis `--accent-tint` e `--shadow-soft` (novas, reutilizadas
  por tasks futuras se necessário). Remove `--accent-2` (nenhuma task
  seguinte depende dela).
- Consumes: nada de outras tasks.

- [ ] **Step 1: Adicionar fonte Inter no `<head>`**

Localizar a linha do `<link rel="stylesheet" href="https://unpkg.com/leaflet@1/dist/leaflet.css">`
e adicionar antes dela:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
```

- [ ] **Step 2: Substituir o bloco `:root`**

Old:
```css
  :root {
    --bg: #050609; --panel: #12141c; --panel-2: #1b1e29; --line: #2a2e3d;
    --text: #eef0f5; --muted: #8b92a5; --accent: #2f6fed; --accent-2: #facc15;
    --ok-bg: #10361f; --ok-fg: #4ade80; --ok-line: #1f6b3a;
    --warn-bg: #3a1717; --warn-fg: #ff6b6b; --warn-line: #7a2a2a;
    --radius: 10px;
    --mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  }
```

New:
```css
  :root {
    --bg: #f5f6fa; --panel: #ffffff; --panel-2: #f0f1f6; --line: #e6e8f0;
    --text: #14161f; --muted: #8a8f9c; --accent: #4b5fee; --accent-tint: rgba(75,95,238,.12);
    --ok-bg: #e7f9ee; --ok-fg: #1aa34a; --ok-line: #b7ecc8;
    --warn-bg: #fdeaea; --warn-fg: #e5484d; --warn-line: #f5c2c2;
    --shadow-soft: 0 8px 24px rgba(20,24,50,.08);
    --radius: 16px;
    --mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
    --sans: 'Inter', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  }
```

- [ ] **Step 3: Trocar CTA do header de amarelo pra azul**

Old:
```css
  header button {
    background: var(--accent-2); color: #161300; border: 0; padding: 8px 16px;
    border-radius: 20px; font-weight: 700; cursor: pointer; font-size: 13px;
  }
  header button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
  header button.active { background: var(--ok-fg); color: #04121f; }
```

New:
```css
  header button {
    background: var(--accent); color: #fff; border: 0; padding: 8px 16px;
    border-radius: 20px; font-weight: 700; cursor: pointer; font-size: 13px;
  }
  header button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
  header button.active { background: var(--ok-fg); color: #fff; }
```

- [ ] **Step 4: Sombra suave no popup do mapa**

Old (dentro da regra `.leaflet-popup-content-wrapper`):
```css
  .leaflet-popup-content-wrapper { padding: 0; border-radius: 12px; overflow: hidden; background: var(--panel); color: var(--text); box-shadow: 0 8px 24px rgba(0,0,0,.4); }
```

New:
```css
  .leaflet-popup-content-wrapper { padding: 0; border-radius: 12px; overflow: hidden; background: var(--panel); color: var(--text); box-shadow: var(--shadow-soft); }
```

- [ ] **Step 5: Texto branco nos estados ativos de chip/seg-group**

Old:
```css
  .chip.active { background: var(--accent); border-color: var(--accent); color: #04121f; font-weight: 600; }
```
New:
```css
  .chip.active { background: var(--accent); border-color: var(--accent); color: #fff; font-weight: 600; }
```

Old:
```css
  .seg-group button.active { background: var(--accent); color: #04121f; }
```
New:
```css
  .seg-group button.active { background: var(--accent); color: #fff; }
```

- [ ] **Step 6: `.type-btn.active` usa o novo token de tint em vez do rgba fixo**

Old:
```css
  .type-btn.active { border-color: var(--accent); background: rgba(47,111,237,.15); color: var(--accent); fill: var(--accent); font-weight: 600; }
```
New:
```css
  .type-btn.active { border-color: var(--accent); background: var(--accent-tint); color: var(--accent); fill: var(--accent); font-weight: 600; }
```

- [ ] **Step 7: Sombra suave no cartão flutuante do mapa**

Old:
```css
  #mapFilterCard {
    position: absolute; top: 14px; left: 14px; z-index: 800; width: 250px;
    background: var(--panel); border-radius: 12px; box-shadow: 0 8px 24px rgba(20,20,40,.15);
    padding: 14px; max-height: calc(100% - 28px); overflow-y: auto;
  }
```
New:
```css
  #mapFilterCard {
    position: absolute; top: 14px; left: 14px; z-index: 800; width: 250px;
    background: var(--panel); border-radius: 12px; box-shadow: var(--shadow-soft);
    padding: 14px; max-height: calc(100% - 28px); overflow-y: auto;
  }
```

- [ ] **Step 8: Verificar que não sobrou resíduo do tema escuro/amarelo**

Run:
```bash
grep -n -- "--accent-2\|#04121f\|#161300\|rgba(0,0,0,.4)\|rgba(20,20,40,.15)\|rgba(47,111,237" mapa-imoveis/index.html
```
Expected: nenhuma saída (nenhum match).

- [ ] **Step 9: Verificação visual**

Abrir via Browser pane: `preview_start({name: "mapa-imoveis"})`, depois
`computer({action:"screenshot"})`. Confirmar: fundo claro, header/CTA azul
com texto branco, sidebar e cartão do mapa em branco/cinza claro, sem
nenhuma superfície preta/amarela restante. Checar `read_console_messages`
sem erros novos.

- [ ] **Step 10: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: retema mapa-imoveis para claro/indigo (fase 1 do redesign)"
```

---

### Task 2: Ícones distintos por tipo de imóvel no pin/popup/lista

**Files:**
- Modify: `mapa-imoveis/index.html` (seção `CONFIG` e `MAP`/`LIST` do `<script>`)

**Interfaces:**
- Consumes: `ICONS` (existente), `UNIT_TYPES` (existente, array de 6 chaves).
- Produces: `ICONS.sobrado`, `ICONS.cobertura`, `ICONS.plot` (novas chaves);
  `TYPE_ICONS` (const global, `{unitType: svgString}`); `buildMarkerIcon(status, unitType)`
  (assinatura nova — era `buildMarkerIcon(status)`). `filterApp().typeIcons`
  passa a apontar pra `TYPE_ICONS` (mesmo nome usado no template Alpine,
  `x-html="typeIcons[ut.value]"` continua funcionando sem mudança de HTML).

- [ ] **Step 1: Adicionar os 3 ícones novos em `ICONS`**

Old:
```js
  store: '<svg width="16" height="16" viewBox="0 0 24 24"><path d="M3 4h18l1 5H2l1-5zM4 10v10h6v-6h4v6h6V10H4z"/></svg>'
};
```
New:
```js
  store: '<svg width="16" height="16" viewBox="0 0 24 24"><path d="M3 4h18l1 5H2l1-5zM4 10v10h6v-6h4v6h6V10H4z"/></svg>',
  sobrado: '<svg width="16" height="16" viewBox="0 0 24 24"><path d="M4 21V11l8-7 8 7v10h-4v-4h-2v4h-4v-4H10v4H4zm4-8h2v-2H8v2zm6 0h2v-2h-2v2z"/></svg>',
  cobertura: '<svg width="16" height="16" viewBox="0 0 24 24"><path d="M3 6h18v2H3V6zm2 4h4v2H5v-2zm6 0h4v2h-4v-2zm6 0h2v2h-2v-2zM5 14h4v2H5v-2zm6 0h4v2h-4v-2zm6 0h2v2h-2v-2zM4 20h16v2H4v-2zM6 20v-4h2v4H6zm10 0v-4h2v4h-2z"/></svg>',
  plot: '<svg width="16" height="16" viewBox="0 0 24 24"><path d="M3 5h18v2H3V5zm0 12h18v2H3v-2zM3 5v14h2V5H3zm16 0v14h2V5h-2zM10 9c0-1.5 1-3 2-3s2 1.5 2 3-1 3-2 3-2-1.5-2-3zm1 5h2v5h-2v-5z"/></svg>'
};
```

- [ ] **Step 2: Promover `TYPE_ICONS` a constante global, logo após `UNIT_TYPE_LABELS`**

Old:
```js
const UNIT_TYPE_LABELS = { casa:'Casa', apartamento:'Apartamento', comercial:'Comercial', terreno:'Terreno', cobertura:'Cobertura', sobrado:'Sobrado' };
const STATUS_OPTIONS = ['disponivel','reservado','vendido','alugado'];
```
New:
```js
const UNIT_TYPE_LABELS = { casa:'Casa', apartamento:'Apartamento', comercial:'Comercial', terreno:'Terreno', cobertura:'Cobertura', sobrado:'Sobrado' };
const STATUS_OPTIONS = ['disponivel','reservado','vendido','alugado'];
const TYPE_ICONS = {
  casa: ICONS.house, sobrado: ICONS.sobrado,
  apartamento: ICONS.building, cobertura: ICONS.cobertura,
  comercial: ICONS.store, terreno: ICONS.plot
};
```

- [ ] **Step 3: Self-check dos ícones (adicionar ao final de `selfCheckLogic()`, antes do `console.log` final)**

Old:
```js
  console.assert(matchesFilters(propB, condosById, { ...baseFilters, amenities: ['gym'] }) === false, 'ASSERT FAIL: matchesFilters deveria rejeitar amenidade ausente');

  console.log('[self-check] Lógica de negócio OK');
```
New:
```js
  console.assert(matchesFilters(propB, condosById, { ...baseFilters, amenities: ['gym'] }) === false, 'ASSERT FAIL: matchesFilters deveria rejeitar amenidade ausente');

  console.assert(UNIT_TYPES.every(t => typeof TYPE_ICONS[t] === 'string' && TYPE_ICONS[t].length > 0), 'ASSERT FAIL: TYPE_ICONS deveria ter ícone para todos os UNIT_TYPES');
  console.assert(new Set(UNIT_TYPES.map(t => TYPE_ICONS[t])).size === UNIT_TYPES.length, 'ASSERT FAIL: TYPE_ICONS deveria ter ícone distinto por tipo (achou duplicata)');

  console.log('[self-check] Lógica de negócio OK');
```

- [ ] **Step 4: Rodar e conferir o self-check no console**

Via Browser pane: `preview_start({name: "mapa-imoveis"})`, depois
`read_console_messages()`. Expected: `[self-check] Lógica de negócio OK`
sem nenhuma linha `ASSERT FAIL`.

- [ ] **Step 5: `buildMarkerIcon` passa a receber `unitType`**

Old:
```js
function buildMarkerIcon(status) {
  const color = STATUS_COLORS[status] || '#8b9bab';
  const svg = `<svg width="30" height="40" viewBox="0 0 30 40" xmlns="http://www.w3.org/2000/svg">
      <path d="M15 0C6.7 0 0 6.7 0 15c0 11 15 25 15 25s15-14 15-25C30 6.7 23.3 0 15 0z" fill="${color}"/>
      <circle cx="15" cy="15" r="10" fill="#04121f" fill-opacity=".25"/>
      <circle cx="15" cy="15" r="9" fill="#0f1720"/>
      <g transform="translate(7,7)" fill="${color}">${ICONS.house}</g>
    </svg>`;
  return L.divIcon({
    className: 'property-pin',
    html: svg,
    iconSize: [30, 40],
    iconAnchor: [15, 40],
    popupAnchor: [0, -36]
  });
}
```
New:
```js
function buildMarkerIcon(status, unitType) {
  const color = STATUS_COLORS[status] || '#8b9bab';
  const typeIcon = TYPE_ICONS[unitType] || ICONS.house;
  const svg = `<svg width="30" height="40" viewBox="0 0 30 40" xmlns="http://www.w3.org/2000/svg">
      <path d="M15 0C6.7 0 0 6.7 0 15c0 11 15 25 15 25s15-14 15-25C30 6.7 23.3 0 15 0z" fill="${color}"/>
      <circle cx="15" cy="15" r="10" fill="#04121f" fill-opacity=".25"/>
      <circle cx="15" cy="15" r="9" fill="#0f1720"/>
      <g transform="translate(7,7)" fill="${color}">${typeIcon}</g>
    </svg>`;
  return L.divIcon({
    className: 'property-pin',
    html: svg,
    iconSize: [30, 40],
    iconAnchor: [15, 40],
    popupAnchor: [0, -36]
  });
}
```

- [ ] **Step 6: Atualizar os 2 call sites de `buildMarkerIcon`**

Old (em `upsertMarker`):
```js
  const marker = L.marker(property.coordinates, { draggable: true, icon: buildMarkerIcon(property.status) });
```
New:
```js
  const marker = L.marker(property.coordinates, { draggable: true, icon: buildMarkerIcon(property.status, property.unitType) });
```

Old (em `geocodeAddress`):
```js
    geocodePreviewMarker = L.marker([lat, lng], { icon: buildMarkerIcon('disponivel') }).addTo(markersLayer);
```
New:
```js
    geocodePreviewMarker = L.marker([lat, lng], { icon: buildMarkerIcon('disponivel', $('f_unitType').value) }).addTo(markersLayer);
```

- [ ] **Step 7: Popup e card da lista usam o ícone do tipo no placeholder de foto vazia**

Old (em `buildPopupHtml`):
```js
      ${photoUrl ? `<img src="${photoUrl}" class="property-popup-photo">` : `<div class="property-popup-photo property-popup-photo-empty">${ICONS.house}</div>`}
```
New:
```js
      ${photoUrl ? `<img src="${photoUrl}" class="property-popup-photo">` : `<div class="property-popup-photo property-popup-photo-empty">${TYPE_ICONS[property.unitType] || ICONS.house}</div>`}
```

Old (em `renderList`):
```js
      ${photoUrl ? `<img src="${photoUrl}" class="list-item-photo">` : `<div class="list-item-photo list-item-photo-empty">${ICONS.house}</div>`}
```
New:
```js
      ${photoUrl ? `<img src="${photoUrl}" class="list-item-photo">` : `<div class="list-item-photo list-item-photo-empty">${TYPE_ICONS[p.unitType] || ICONS.house}</div>`}
```

- [ ] **Step 8: Simplificar `typeIcons` dentro de `filterApp()` pra reusar a constante global**

Old:
```js
    typeIcons: {
      casa: ICONS.house, sobrado: ICONS.house,
      apartamento: ICONS.building, cobertura: ICONS.building,
      comercial: ICONS.store, terreno: ICONS.area
    },
```
New:
```js
    typeIcons: TYPE_ICONS,
```

- [ ] **Step 9: Teste manual — criar um imóvel de cada tipo e comparar os pins**

Via Browser pane, com o servidor `mapa-imoveis` já rodando, executar no
`javascript_tool`:

```js
const types = ['casa','apartamento','comercial','terreno','cobertura','sobrado'];
const svgs = types.map(t => buildMarkerIcon('disponivel', t).options.html);
JSON.stringify({ allDifferent: new Set(svgs).size === types.length });
```
Expected: `{"allDifferent":true}`.

- [ ] **Step 10: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: pin/popup/lista usam ícone do tipo de imóvel em vez de sempre casa"
```

---

### Task 3: Cartão flutuante do mapa ganha botão de recolher

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML de `#mapFilterCard`, CSS, seção
  `LAYOUT` do `<script>`)

**Interfaces:**
- Consumes: nenhuma interface de outras tasks (independente).
- Produces: `#btnCollapseMapFilter` (id), classe `.collapsed` em
  `#mapFilterCard` (nome de classe já usado em `#sidebar`, mesmo padrão,
  contextos distintos — sem conflito porque são seletores `#id.collapsed`).

- [ ] **Step 1: Adicionar o botão dentro do HTML de `#mapFilterCard`**

Old:
```html
    <div id="mapFilterCard">
      <div class="seg-group">
        <template x-for="opt in [{v:'ambos',l:'Ambos'},{v:'venda',l:'Comprar'},{v:'aluguel',l:'Alugar'}]" :key="opt.v">
```
New:
```html
    <div id="mapFilterCard">
      <button type="button" id="btnCollapseMapFilter" title="Ocultar filtros do mapa">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M18 15l-6-6-6 6"/></svg>
      </button>
      <div class="seg-group">
        <template x-for="opt in [{v:'ambos',l:'Ambos'},{v:'venda',l:'Comprar'},{v:'aluguel',l:'Alugar'}]" :key="opt.v">
```

- [ ] **Step 2: CSS do botão e do estado recolhido**

Old:
```css
  #mapFilterCard label { display: block; font-size: 12px; color: var(--muted); margin: 10px 0 4px; }
```
New:
```css
  #btnCollapseMapFilter {
    position: absolute; top: 10px; right: 10px; width: 24px; height: 24px; border-radius: 50%;
    background: var(--panel-2); border: 1px solid var(--line); color: var(--text);
    display: flex; align-items: center; justify-content: center; cursor: pointer; z-index: 2;
    transition: transform .2s ease;
  }
  #btnCollapseMapFilter:hover { border-color: var(--accent); color: var(--accent); }
  #btnCollapseMapFilter.collapsed { transform: rotate(180deg); }
  #mapFilterCard.collapsed { width: 40px; height: 40px; padding: 0; overflow: hidden; }
  #mapFilterCard.collapsed > *:not(#btnCollapseMapFilter) { display: none; }
  #mapFilterCard label { display: block; font-size: 12px; color: var(--muted); margin: 10px 0 4px; }
```

- [ ] **Step 3: JS do toggle, na seção `LAYOUT`, junto do `btnCollapseSidebar`**

Old:
```js
  $('btnCollapseSidebar').addEventListener('click', () => {
    const collapsed = $('sidebar').classList.toggle('collapsed');
    $('btnCollapseSidebar').classList.toggle('collapsed', collapsed);
    $('btnCollapseSidebar').title = collapsed ? 'Mostrar filtros' : 'Ocultar filtros';
    setTimeout(() => map.invalidateSize(), 210);
  });
```
New:
```js
  $('btnCollapseSidebar').addEventListener('click', () => {
    const collapsed = $('sidebar').classList.toggle('collapsed');
    $('btnCollapseSidebar').classList.toggle('collapsed', collapsed);
    $('btnCollapseSidebar').title = collapsed ? 'Mostrar filtros' : 'Ocultar filtros';
    setTimeout(() => map.invalidateSize(), 210);
  });

  $('btnCollapseMapFilter').addEventListener('click', () => {
    const collapsed = $('mapFilterCard').classList.toggle('collapsed');
    $('btnCollapseMapFilter').classList.toggle('collapsed', collapsed);
    $('btnCollapseMapFilter').title = collapsed ? 'Mostrar filtros do mapa' : 'Ocultar filtros do mapa';
  });
```

- [ ] **Step 4: Teste manual**

Via Browser pane: clicar `#btnCollapseMapFilter` (`computer` ou
`javascript_tool` disparando `.click()`), depois `read_page` ou
`javascript_tool` checando
`document.getElementById('mapFilterCard').classList.contains('collapsed')`.
Expected: `true` no primeiro clique, `false` no segundo. Confirmar também
que a sidebar continua recolhendo/expandindo independente (os dois botões
não devem interferir um no outro).

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: cartão de filtros do mapa também pode ser recolhido"
```

---

### Task 4: Formulário de imóvel em seções + rodapé fixo

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML de `#propertyForm`, CSS)

**Interfaces:**
- Consumes: nenhuma interface de outras tasks (independente).
- Produces: classes `.form-section` / `.form-section-title` (novas, CSS
  puro). Nenhum id de campo muda — `readPropertyForm`/`openPropertyForm`
  continuam funcionando sem alteração.

- [ ] **Step 1: CSS do layout em coluna + seções + rodapé fixo**

Old:
```css
  #propertyForm {
    position: fixed; top: 0; right: 0; height: 100%; width: 380px; max-width: 100vw;
    background: var(--panel); border-left: 1px solid var(--line); z-index: 40;
    transform: translateX(100%); transition: transform .25s cubic-bezier(.16,1,.3,1); overflow-y: auto; padding: 18px;
  }
  #propertyForm.open { transform: translateX(0); }
  #propertyForm h2 { font-size: 15px; margin: 0 0 14px; }
  #propertyForm label { display: block; font-size: 12px; color: var(--muted); margin: 10px 0 4px; }
  #propertyForm input, #propertyForm select, #propertyForm textarea {
    width: 100%; background: var(--panel-2); border: 1px solid var(--line); color: var(--text);
    padding: 8px 10px; border-radius: 8px; font-size: 14px;
  }
  #propertyForm .row { display: flex; gap: 10px; }
  #propertyForm .row > * { flex: 1; }
  #formErrors { background: var(--warn-bg); border: 1px solid var(--warn-line); color: var(--warn-fg); padding: 8px 10px; border-radius: 8px; font-size: 13px; margin-top: 10px; display: none; }
  #formErrors.show { display: block; }
  .btn-row { display: flex; gap: 10px; margin-top: 16px; }
  .btn-row button { flex: 1; }
```
New:
```css
  #propertyForm {
    position: fixed; top: 0; right: 0; height: 100%; width: 380px; max-width: 100vw;
    background: var(--panel); border-left: 1px solid var(--line); z-index: 40;
    transform: translateX(100%); transition: transform .25s cubic-bezier(.16,1,.3,1);
    display: flex; flex-direction: column; padding: 18px 18px 0;
  }
  #propertyForm.open { transform: translateX(0); }
  #propertyForm h2 { font-size: 15px; margin: 0 0 14px; flex-shrink: 0; }
  #propertyFormEl { flex: 1; min-height: 0; overflow-y: auto; padding-bottom: 4px; }
  .form-section { padding: 10px 0; border-bottom: 1px solid var(--line); }
  .form-section:last-of-type { border-bottom: 0; }
  .form-section-title { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: .4px; color: var(--muted); margin: 0 0 4px; }
  #propertyForm label { display: block; font-size: 12px; color: var(--muted); margin: 10px 0 4px; }
  #propertyForm input, #propertyForm select, #propertyForm textarea {
    width: 100%; background: var(--panel-2); border: 1px solid var(--line); color: var(--text);
    padding: 8px 10px; border-radius: 8px; font-size: 14px;
  }
  #propertyForm .row { display: flex; gap: 10px; }
  #propertyForm .row > * { flex: 1; }
  #formErrors { background: var(--warn-bg); border: 1px solid var(--warn-line); color: var(--warn-fg); padding: 8px 10px; border-radius: 8px; font-size: 13px; margin-top: 10px; display: none; flex-shrink: 0; }
  #formErrors.show { display: block; }
  .btn-row { display: flex; gap: 10px; flex-shrink: 0; padding: 12px 0 18px; border-top: 1px solid var(--line); background: var(--panel); }
  .btn-row button { flex: 1; }
```

- [ ] **Step 2: Envolver os campos em `<div class="form-section">` e mover o rodapé pra fora do `<form>`**

Old (arquivo inteiro do form, do `<form id="propertyFormEl">` até o fechamento de `</div>` do `#propertyForm`):
```html
  <form id="propertyFormEl">
    <input type="hidden" id="f_id">
    <input type="hidden" id="f_lat">
    <input type="hidden" id="f_lng">

    <label for="f_label">Apelido (opcional)</label>
    <input id="f_label" type="text" placeholder="Ex: Casa Capoeiras 296">

    <div class="row">
      <div>
        <label for="f_status">Status</label>
        <select id="f_status">
          <option value="disponivel">Disponível</option>
          <option value="reservado">Reservado</option>
          <option value="vendido">Vendido</option>
          <option value="alugado">Alugado</option>
        </select>
      </div>
      <div>
        <label for="f_dealType">Negócio</label>
        <select id="f_dealType">
          <option value="venda">Venda</option>
          <option value="aluguel">Aluguel</option>
          <option value="venda_e_aluguel">Venda e aluguel</option>
        </select>
      </div>
    </div>

    <label for="f_unitType">Tipo de imóvel</label>
    <select id="f_unitType">
      <option value="casa">Casa</option>
      <option value="apartamento">Apartamento</option>
      <option value="comercial">Comercial</option>
      <option value="terreno">Terreno</option>
      <option value="cobertura">Cobertura</option>
      <option value="sobrado">Sobrado</option>
    </select>

    <div class="row">
      <div>
        <label for="f_salePrice">Preço de venda (R$)</label>
        <input id="f_salePrice" type="number" min="0" step="1000">
      </div>
      <div>
        <label for="f_rentPrice">Preço de aluguel (R$)</label>
        <input id="f_rentPrice" type="number" min="0" step="50">
      </div>
    </div>

    <div class="row">
      <div>
        <label for="f_constructedArea">Área construída (m²)</label>
        <input id="f_constructedArea" type="number" min="0" step="0.01">
      </div>
      <div>
        <label for="f_landArea">Área do lote (m²)</label>
        <input id="f_landArea" type="number" min="0" step="0.01">
      </div>
    </div>

    <div class="row">
      <div>
        <label for="f_rooms">Quartos</label>
        <input id="f_rooms" type="number" min="0" step="1">
      </div>
      <div>
        <label for="f_suites">Suítes</label>
        <input id="f_suites" type="number" min="0" step="1">
      </div>
    </div>

    <div class="row">
      <div>
        <label for="f_bathrooms">Banheiros</label>
        <input id="f_bathrooms" type="number" min="0" step="1">
      </div>
      <div>
        <label for="f_parkingSpaces">Vagas</label>
        <input id="f_parkingSpaces" type="number" min="0" step="1">
      </div>
    </div>

    <div class="row">
      <div>
        <label for="f_floor">Andar</label>
        <input id="f_floor" type="number" step="1">
      </div>
      <div>
        <label for="f_furnished">Mobiliado</label>
        <select id="f_furnished">
          <option value="não">Não</option>
          <option value="sim">Sim</option>
          <option value="parcial">Parcial</option>
        </select>
      </div>
    </div>

    <label for="f_petsAllowed">Aceita pet</label>
    <select id="f_petsAllowed">
      <option value="false">Não</option>
      <option value="true">Sim</option>
    </select>

    <label for="f_street">Rua</label>
    <input id="f_street" type="text">
    <div class="row">
      <div>
        <label for="f_number">Número</label>
        <input id="f_number" type="text">
      </div>
      <div>
        <label for="f_neighborhood">Bairro</label>
        <input id="f_neighborhood" type="text">
      </div>
    </div>
    <div class="row">
      <div>
        <label for="f_city">Cidade</label>
        <input id="f_city" type="text" value="Florianópolis">
      </div>
      <div>
        <label for="f_complement">Complemento</label>
        <input id="f_complement" type="text">
      </div>
    </div>
    <button type="button" id="btnGeocodeAddress" class="ghost" style="width:100%;margin-top:6px;">🔍 Localizar endereço no mapa</button>

    <label for="f_agentResponsible">Corretor responsável</label>
    <input id="f_agentResponsible" type="text">

    <label for="f_condoId">Condomínio</label>
    <select id="f_condoId">
      <option value="">Nenhum</option>
    </select>
    <button type="button" id="btnNewCondo" class="ghost" style="margin-top:6px;">+ Novo condomínio</button>
    <div id="newCondoFields" style="display:none; margin-top:10px; padding:10px; background:var(--panel-2); border-radius:8px;">
      <label for="c_name">Nome do condomínio</label>
      <input id="c_name" type="text">
      <div class="row" style="margin-top:8px; flex-wrap: wrap;">
        <label><input type="checkbox" id="c_pool"> Piscina</label>
        <label><input type="checkbox" id="c_gym"> Academia</label>
        <label><input type="checkbox" id="c_partyRoom"> Salão de festas</label>
      </div>
      <div class="row" style="margin-top:4px; flex-wrap: wrap;">
        <label><input type="checkbox" id="c_playground"> Playground</label>
        <label><input type="checkbox" id="c_petArea"> Área pet</label>
        <label><input type="checkbox" id="c_security24h"> Portaria 24h</label>
      </div>
      <div class="row" style="margin-top:4px; flex-wrap: wrap;">
        <label><input type="checkbox" id="c_elevator"> Elevador</label>
        <label><input type="checkbox" id="c_gatedCommunity"> Condomínio fechado</label>
      </div>
      <label for="c_monthlyFee">Valor do condomínio (R$)</label>
      <input id="c_monthlyFee" type="number" min="0" step="10">
    </div>

    <label for="f_notes">Observações</label>
    <textarea id="f_notes" rows="3"></textarea>

    <label for="f_photos">Fotos</label>
    <input id="f_photos" type="file" accept="image/*" multiple>
    <div id="photoThumbs" style="margin-top:8px;"></div>

    <div id="formErrors"></div>

    <div class="btn-row">
      <button type="submit">Salvar</button>
      <button type="button" id="btnCancelForm" class="ghost">Cancelar</button>
    </div>
  </form>
</div>
```

New:
```html
  <form id="propertyFormEl">
    <input type="hidden" id="f_id">
    <input type="hidden" id="f_lat">
    <input type="hidden" id="f_lng">

    <div class="form-section">
      <div class="form-section-title">Básico</div>
      <label for="f_label">Apelido (opcional)</label>
      <input id="f_label" type="text" placeholder="Ex: Casa Capoeiras 296">

      <div class="row">
        <div>
          <label for="f_status">Status</label>
          <select id="f_status">
            <option value="disponivel">Disponível</option>
            <option value="reservado">Reservado</option>
            <option value="vendido">Vendido</option>
            <option value="alugado">Alugado</option>
          </select>
        </div>
        <div>
          <label for="f_dealType">Negócio</label>
          <select id="f_dealType">
            <option value="venda">Venda</option>
            <option value="aluguel">Aluguel</option>
            <option value="venda_e_aluguel">Venda e aluguel</option>
          </select>
        </div>
      </div>

      <label for="f_unitType">Tipo de imóvel</label>
      <select id="f_unitType">
        <option value="casa">Casa</option>
        <option value="apartamento">Apartamento</option>
        <option value="comercial">Comercial</option>
        <option value="terreno">Terreno</option>
        <option value="cobertura">Cobertura</option>
        <option value="sobrado">Sobrado</option>
      </select>
    </div>

    <div class="form-section">
      <div class="form-section-title">Preço e área</div>
      <div class="row">
        <div>
          <label for="f_salePrice">Preço de venda (R$)</label>
          <input id="f_salePrice" type="number" min="0" step="1000">
        </div>
        <div>
          <label for="f_rentPrice">Preço de aluguel (R$)</label>
          <input id="f_rentPrice" type="number" min="0" step="50">
        </div>
      </div>

      <div class="row">
        <div>
          <label for="f_constructedArea">Área construída (m²)</label>
          <input id="f_constructedArea" type="number" min="0" step="0.01">
        </div>
        <div>
          <label for="f_landArea">Área do lote (m²)</label>
          <input id="f_landArea" type="number" min="0" step="0.01">
        </div>
      </div>
    </div>

    <div class="form-section">
      <div class="form-section-title">Características</div>
      <div class="row">
        <div>
          <label for="f_rooms">Quartos</label>
          <input id="f_rooms" type="number" min="0" step="1">
        </div>
        <div>
          <label for="f_suites">Suítes</label>
          <input id="f_suites" type="number" min="0" step="1">
        </div>
      </div>

      <div class="row">
        <div>
          <label for="f_bathrooms">Banheiros</label>
          <input id="f_bathrooms" type="number" min="0" step="1">
        </div>
        <div>
          <label for="f_parkingSpaces">Vagas</label>
          <input id="f_parkingSpaces" type="number" min="0" step="1">
        </div>
      </div>

      <div class="row">
        <div>
          <label for="f_floor">Andar</label>
          <input id="f_floor" type="number" step="1">
        </div>
        <div>
          <label for="f_furnished">Mobiliado</label>
          <select id="f_furnished">
            <option value="não">Não</option>
            <option value="sim">Sim</option>
            <option value="parcial">Parcial</option>
          </select>
        </div>
      </div>

      <label for="f_petsAllowed">Aceita pet</label>
      <select id="f_petsAllowed">
        <option value="false">Não</option>
        <option value="true">Sim</option>
      </select>
    </div>

    <div class="form-section">
      <div class="form-section-title">Endereço</div>
      <label for="f_street">Rua</label>
      <input id="f_street" type="text">
      <div class="row">
        <div>
          <label for="f_number">Número</label>
          <input id="f_number" type="text">
        </div>
        <div>
          <label for="f_neighborhood">Bairro</label>
          <input id="f_neighborhood" type="text">
        </div>
      </div>
      <div class="row">
        <div>
          <label for="f_city">Cidade</label>
          <input id="f_city" type="text" value="Florianópolis">
        </div>
        <div>
          <label for="f_complement">Complemento</label>
          <input id="f_complement" type="text">
        </div>
      </div>
      <button type="button" id="btnGeocodeAddress" class="ghost" style="width:100%;margin-top:6px;">🔍 Localizar endereço no mapa</button>
    </div>

    <div class="form-section">
      <div class="form-section-title">Condomínio</div>
      <label for="f_agentResponsible">Corretor responsável</label>
      <input id="f_agentResponsible" type="text">

      <label for="f_condoId">Condomínio</label>
      <select id="f_condoId">
        <option value="">Nenhum</option>
      </select>
      <button type="button" id="btnNewCondo" class="ghost" style="margin-top:6px;">+ Novo condomínio</button>
      <div id="newCondoFields" style="display:none; margin-top:10px; padding:10px; background:var(--panel-2); border-radius:8px;">
        <label for="c_name">Nome do condomínio</label>
        <input id="c_name" type="text">
        <div class="row" style="margin-top:8px; flex-wrap: wrap;">
          <label><input type="checkbox" id="c_pool"> Piscina</label>
          <label><input type="checkbox" id="c_gym"> Academia</label>
          <label><input type="checkbox" id="c_partyRoom"> Salão de festas</label>
        </div>
        <div class="row" style="margin-top:4px; flex-wrap: wrap;">
          <label><input type="checkbox" id="c_playground"> Playground</label>
          <label><input type="checkbox" id="c_petArea"> Área pet</label>
          <label><input type="checkbox" id="c_security24h"> Portaria 24h</label>
        </div>
        <div class="row" style="margin-top:4px; flex-wrap: wrap;">
          <label><input type="checkbox" id="c_elevator"> Elevador</label>
          <label><input type="checkbox" id="c_gatedCommunity"> Condomínio fechado</label>
        </div>
        <label for="c_monthlyFee">Valor do condomínio (R$)</label>
        <input id="c_monthlyFee" type="number" min="0" step="10">
      </div>
    </div>

    <div class="form-section">
      <div class="form-section-title">Fotos e observações</div>
      <label for="f_notes">Observações</label>
      <textarea id="f_notes" rows="3"></textarea>

      <label for="f_photos">Fotos</label>
      <input id="f_photos" type="file" accept="image/*" multiple>
      <div id="photoThumbs" style="margin-top:8px;"></div>
    </div>
  </form>
  <div id="formErrors"></div>
  <div class="btn-row">
    <button type="submit" form="propertyFormEl">Salvar</button>
    <button type="button" id="btnCancelForm" class="ghost">Cancelar</button>
  </div>
</div>
```

- [ ] **Step 3: Verificar que nenhum id de campo foi perdido/duplicado**

Run:
```bash
grep -o 'id="f_[a-zA-Z]*"' mapa-imoveis/index.html | sort | uniq -c | sort -rn | head -5
```
Expected: cada `id="f_..."` aparece exatamente 1 vez (coluna de contagem = 1
em todas as linhas — se alguma tiver 2+, sobrou uma cópia duplicada do
campo e a Step 2 foi aplicada errado).

- [ ] **Step 4: Teste manual — abrir formulário, rolar, verificar rodapé fixo**

Via Browser pane com `mapa-imoveis` rodando: clicar "+ Adicionar imóvel",
depois `read_page` pra confirmar as 6 seções (`.form-section-title`)
presentes na ordem certa, e `javascript_tool` checando:
```js
getComputedStyle($('propertyFormEl')).overflowY;
```
Expected: `"auto"`. Depois checar que `$('formErrors')` e o `.btn-row` estão
fora do elemento `#propertyFormEl` (`$('propertyFormEl').contains($('formErrors'))` → `false`).

Testar submit continua funcionando (reaproveitar o mesmo fluxo do fim da
sessão anterior): preencher `f_salePrice`/`f_street`/`f_neighborhood` +
`f_lat`/`f_lng` via JS e clicar no botão Salvar (fora do `<form>` mas com
`form="propertyFormEl"`), confirmar que `savePropertyForm` disparou
(imóvel aparece em `appState.properties`).

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: formulário de imóvel em seções, com Salvar/Cancelar fixos no rodapé"
```

---

## Self-Review

**Cobertura do spec:**
- Tokens de tema + fonte Inter + remoção do amarelo → Task 1. ✅
- Sidebar recolhível → já existia, mantido (Task 1 só reskin). ✅
- Cartão flutuante do mapa recolhível (novo) → Task 3. ✅
- Formulário reorganizado em seções + rodapé fixo → Task 4. ✅
- Ícone do pin por tipo de imóvel (6 tipos distintos) → Task 2. ✅
- Fora de escopo (tiles do mapa, campos novos, página de detalhe,
  dashboard, dark mode) → nenhuma task toca nisso. ✅

**Placeholders:** nenhum "TBD"/"depois" — todo Old/New é código completo.

**Consistência de tipos/nomes:** `TYPE_ICONS` (Task 2) é o único nome usado
em `buildMarkerIcon`, `buildPopupHtml`, `renderList` e `filterApp().typeIcons`
— conferido, sem variação de nome entre tasks. `buildMarkerIcon(status, unitType)`
tem a mesma assinatura nos 2 call sites atualizados (Task 2 Step 6).

# Mapa de Imóveis — Dashboard (Fase 4) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dashboard vira a aba padrão do app, com KPIs, balanço de vendas
por período, inventário de unidades de lançamento, imóvel em destaque
(último adicionado) e tabela de imóveis recentes.

**Architecture:** Continua em `mapa-imoveis/index.html`. Task 1 entrega a
navegação (nova aba padrão + fix do "voltar" da página de detalhe) e os
KPIs — já é um dashboard funcional mínimo. Tasks 2 e 3 preenchem
containers vazios que a Task 1 deixa prontos (mesmo padrão de
`detailExtra`/`detailFooter` da fase 3), sem re-tocar navegação.

**Tech Stack:** Mesmo stack — HTML/CSS/JS vanilla + Alpine.js + Leaflet,
sem framework de teste, sem lib de gráfico (barras em CSS/SVG simples).
Verificação via Browser pane (`preview_start({name: "mapa-imoveis"})`) +
`console.assert`.

## Global Constraints

- Arquivo único: `mapa-imoveis/index.html`. Sem novos arquivos/dependências.
- Sem mini-mapa (não instanciar um 2º Leaflet).
- `updatedAt` é usado como proxy de "data da venda/aluguel" pro balanço de
  vendas — simplificação documentada no spec, não é pra "corrigir" ou
  adicionar um campo novo de data.
- Texto livre do usuário (`label`, `agentResponsible`) que for renderizado
  em HTML novo desta fase passa por `escapeHtml()` — não repetir o XSS que
  a fase 3 corrigiu em `notes`. Não mexer no `innerHTML` pré-existente de
  popup/lista/detalhe (fora de escopo).
- IDs de elementos existentes não podem mudar.
- IndexedDB não funciona em `file://` — testar via `preview_start`.

---

### Task 1: Dashboard como aba padrão + KPIs + fix de navegação do detalhe

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML do header/tabs, `#mapView`,
  novo `#dashboardView`; CSS; JS: `closeDetailView`, `openPropertyDetail`,
  `renderDashboard`, handlers de aba)

**Interfaces:**
- Produces: `renderDashboard()` (função global, chamada pelo listener de
  `properties-updated` — já existente, disparado por save/delete/import/
  ajuste de unidades). `closeDetailView()` (substitui a lógica de
  navegação duplicada de `btnDetailBack` e do rodapé de excluir).
  `#dashKpis`, `#dashSalesCard`, `#dashLaunchCard`, `#dashFeatured`,
  `#dashTableWrap` (containers vazios que as Tasks 2/3 preenchem).
- Consumes: `appState.properties` (já existe). `openPropertyDetail`,
  `deleteProperty` (fase 3, já existentes).

- [ ] **Step 1: Header — 3ª aba "Dashboard", ativa por padrão**

Old:
```html
    <div class="view-tabs">
      <button type="button" id="tabMap" class="view-tab active">Mapa</button>
      <button type="button" id="tabFinancing" class="view-tab">Financiamento</button>
    </div>
```
New:
```html
    <div class="view-tabs">
      <button type="button" id="tabDashboard" class="view-tab active">Dashboard</button>
      <button type="button" id="tabMap" class="view-tab">Mapa</button>
      <button type="button" id="tabFinancing" class="view-tab">Financiamento</button>
    </div>
```

- [ ] **Step 2: `#mapView` começa escondido**

Old:
```html
  <div class="layout-body" id="mapView">
```
New:
```html
  <div class="layout-body" id="mapView" style="display:none;">
```

- [ ] **Step 3: HTML de `#dashboardView`, antes de `#mapView`**

Old:
```html
  <div class="layout-body" id="mapView" style="display:none;">
```
New:
```html
  <div id="dashboardView">
    <div class="dash-kpis" id="dashKpis"></div>
    <div class="dash-grid">
      <div class="dash-main">
        <div id="dashSalesCard"></div>
        <div id="dashLaunchCard"></div>
        <div id="dashTableWrap"></div>
      </div>
      <div class="dash-side">
        <div id="dashFeatured"></div>
      </div>
    </div>
  </div>
  <div class="layout-body" id="mapView" style="display:none;">
```

- [ ] **Step 4: CSS de `#dashboardView`, logo após `#financingFrame`**

Old:
```css
  #financingFrame { width: 100%; height: 100%; border: 0; display: block; }
```
New:
```css
  #financingFrame { width: 100%; height: 100%; border: 0; display: block; }
  #dashboardView { flex: 1; min-height: 0; overflow-y: auto; padding: 28px 32px; }
  .dash-kpis { display: flex; gap: 16px; flex-wrap: wrap; margin-bottom: 24px; }
  .dash-kpi { background: var(--panel); border: 1px solid var(--line); border-radius: 12px; padding: 16px 20px; min-width: 120px; box-shadow: var(--shadow-soft); }
  .dash-kpi strong { display: block; font-size: 24px; }
  .dash-kpi span { color: var(--muted); font-size: 12px; }
  .dash-grid { display: grid; grid-template-columns: 1.6fr 1fr; gap: 24px; align-items: start; }
  .dash-card { background: var(--panel); border: 1px solid var(--line); border-radius: 12px; padding: 18px; box-shadow: var(--shadow-soft); margin-bottom: 20px; }
  .dash-card h3 { font-size: 14px; margin: 0 0 12px; }
  .dash-main button, .dash-side button { background: var(--accent); color: #fff; border: 0; padding: 6px 14px; border-radius: 8px; font-weight: 700; cursor: pointer; font-size: 12px; }
  .dash-main button.ghost, .dash-side button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
```

- [ ] **Step 5: `renderDashboard` — só os KPIs nesta task, logo antes de `function buildPopupHtml`**

Old:
```js
function buildPopupHtml(property) {
```
New:
```js
function renderDashboard() {
  const props = appState.properties;
  const countByStatus = (s) => props.filter(p => p.status === s).length;
  $('dashKpis').innerHTML = [
    { label: 'Total de imóveis', value: props.length },
    { label: 'Disponíveis', value: countByStatus('disponivel') },
    { label: 'Reservados', value: countByStatus('reservado') },
    { label: 'Vendidos', value: countByStatus('vendido') },
    { label: 'Alugados', value: countByStatus('alugado') }
  ].map(k => `<div class="dash-kpi"><strong>${k.value}</strong><span>${k.label}</span></div>`).join('');
  $('dashSalesCard').innerHTML = '';
  $('dashLaunchCard').innerHTML = '';
  $('dashFeatured').innerHTML = '';
  $('dashTableWrap').innerHTML = '';
}

window.addEventListener('properties-updated', () => renderDashboard());

function buildPopupHtml(property) {
```

Nota: `$('dashSalesCard')`/`$('dashLaunchCard')`/`$('dashFeatured')`/
`$('dashTableWrap')` ficam vazios nesta task de propósito — as Tasks 2/3
substituem essas 4 linhas por lógica que os preenche.

- [ ] **Step 6: `closeDetailView` + `openPropertyDetail` grava de onde veio**

Old:
```js
function openPropertyDetail(id) {
  const property = appState.properties.find(p => p.id === id);
  if (!property) return;
  $('detailView').dataset.id = id;
  const addr = property.address || {};
```
New:
```js
function closeDetailView() {
  $('detailView').style.display = 'none';
  if ($('detailView').dataset.returnTo === 'dashboard') {
    $('dashboardView').style.display = '';
    $('tabDashboard').classList.add('active');
    $('tabMap').classList.remove('active');
  } else {
    $('mapView').style.display = 'flex';
    $('tabMap').classList.add('active');
    $('tabDashboard').classList.remove('active');
    map.invalidateSize();
  }
}

function openPropertyDetail(id) {
  const property = appState.properties.find(p => p.id === id);
  if (!property) return;
  $('detailView').dataset.id = id;
  $('detailView').dataset.returnTo = getComputedStyle($('dashboardView')).display !== 'none' ? 'dashboard' : 'map';
  const addr = property.address || {};
```

- [ ] **Step 7: `openPropertyDetail` esconde o dashboard também ao abrir**

Old:
```js
  $('detailFooter').innerHTML = `<button type="button" class="ghost" onclick="deleteProperty('${property.id}').then(deleted => { if (deleted) { $('detailView').style.display = 'none'; $('mapView').style.display = ''; map.invalidateSize(); } })">Excluir imóvel</button>`;
  $('mapView').style.display = 'none';
  $('financingView').style.display = 'none';
  $('detailView').style.display = 'block';
}
```
New:
```js
  $('detailFooter').innerHTML = `<button type="button" class="ghost" onclick="deleteProperty('${property.id}').then(deleted => { if (deleted) closeDetailView(); })">Excluir imóvel</button>`;
  $('dashboardView').style.display = 'none';
  $('mapView').style.display = 'none';
  $('financingView').style.display = 'none';
  $('detailView').style.display = 'block';
}
```

- [ ] **Step 8: Handlers de aba — Dashboard nova, Mapa/Financiamento escondem o dashboard também, `btnDetailBack` usa `closeDetailView`**

Old:
```js
  $('tabMap').addEventListener('click', () => {
    $('mapView').style.display = '';
    $('financingView').style.display = 'none';
    $('detailView').style.display = 'none';
    $('tabMap').classList.add('active');
    $('tabFinancing').classList.remove('active');
    map.invalidateSize();
  });
  $('tabFinancing').addEventListener('click', () => {
    $('mapView').style.display = 'none';
    $('financingView').style.display = 'flex';
    $('detailView').style.display = 'none';
    $('tabFinancing').classList.add('active');
    $('tabMap').classList.remove('active');
    if (!$('financingFrame').src) $('financingFrame').src = 'financiamento.html';
  });
  $('btnDetailBack').addEventListener('click', () => {
    $('detailView').style.display = 'none';
    $('mapView').style.display = '';
    map.invalidateSize();
  });
});
```
New:
```js
  $('tabDashboard').addEventListener('click', () => {
    $('dashboardView').style.display = '';
    $('mapView').style.display = 'none';
    $('financingView').style.display = 'none';
    $('detailView').style.display = 'none';
    $('tabDashboard').classList.add('active');
    $('tabMap').classList.remove('active');
    $('tabFinancing').classList.remove('active');
  });
  $('tabMap').addEventListener('click', () => {
    $('dashboardView').style.display = 'none';
    $('mapView').style.display = 'flex';
    $('financingView').style.display = 'none';
    $('detailView').style.display = 'none';
    $('tabMap').classList.add('active');
    $('tabDashboard').classList.remove('active');
    $('tabFinancing').classList.remove('active');
    map.invalidateSize();
  });
  $('tabFinancing').addEventListener('click', () => {
    $('dashboardView').style.display = 'none';
    $('mapView').style.display = 'none';
    $('financingView').style.display = 'flex';
    $('detailView').style.display = 'none';
    $('tabFinancing').classList.add('active');
    $('tabDashboard').classList.remove('active');
    $('tabMap').classList.remove('active');
    if (!$('financingFrame').src) $('financingFrame').src = 'financiamento.html';
  });
  $('btnDetailBack').addEventListener('click', closeDetailView);
});
```

Nota: `#mapView` agora usa `style.display = 'flex'` (não mais `''`) pra
mostrar, já que o estado padrão dele mudou pra `display:none` inline
(Step 2) — `''` voltaria a herdar esse `none`, não o `flex` da classe
`.layout-body`.

- [ ] **Step 9: Teste manual — Dashboard como aba padrão, KPIs corretos, navegação de volta**

Via Browser pane com `preview_start({name: "mapa-imoveis"})`: recarregar
e confirmar que `getComputedStyle($('dashboardView')).display !== 'none'`
e `getComputedStyle($('mapView')).display === 'none'` logo no load, com
`#tabDashboard` tendo a classe `active`. Confirmar `#dashKpis` tem 5
`.dash-kpi` com números batendo com `appState.properties` (criar 1-2
imóveis com status diferentes e conferir a contagem). Clicar num imóvel
existente (via `openPropertyForm`+popup, ou direto `openPropertyDetail`
chamado a partir do estado atual = dashboard) e confirmar
`$('detailView').dataset.returnTo === 'dashboard'`; clicar "← Voltar" e
confirmar que volta pro dashboard (`#dashboardView` visível, não
`#mapView`). Trocar pra aba Mapa, abrir um imóvel pelo popup, voltar, e
confirmar que dessa vez volta pro Mapa. Trocar entre as 3 abas e
confirmar que sempre só uma view fica visível por vez.

- [ ] **Step 10: Rodar self-check no console**

`read_console_messages()` — `[self-check] Lógica de negócio OK` sem
`ASSERT FAIL`.

- [ ] **Step 11: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: dashboard como aba padrão, com KPIs, e fix de navegação da página de detalhe"
```

---

### Task 2: Balanço de vendas por período + unidades de lançamento

**Files:**
- Modify: `mapa-imoveis/index.html` (CSS; JS: `salesBalanceFor`,
  `renderSalesCard`, `renderLaunchCard`, extensão de `renderDashboard`)

**Interfaces:**
- Consumes: `renderDashboard` (Task 1) — substitui as linhas
  `$('dashSalesCard').innerHTML = ''` / `$('dashLaunchCard').innerHTML = ''`.
  `formatCurrency` (já existe).
- Produces: `salesBalanceFor(days)`, `renderSalesCard(days)`,
  `renderLaunchCard()` (funções globais, só usadas dentro desta seção).

- [ ] **Step 1: CSS das barras e do seletor de período, logo após `.dash-card h3`**

Old:
```css
  .dash-card h3 { font-size: 14px; margin: 0 0 12px; }
```
New:
```css
  .dash-card h3 { font-size: 14px; margin: 0 0 12px; }
  .dash-period { background: var(--panel-2); border: 1px solid var(--line); color: var(--text); padding: 6px 10px; border-radius: 8px; font-size: 12px; margin-bottom: 14px; }
  .dash-bar-row { display: flex; align-items: center; gap: 10px; margin-bottom: 10px; font-size: 13px; }
  .dash-bar-row span { width: 110px; flex-shrink: 0; color: var(--muted); }
  .dash-bar-row strong { width: 110px; flex-shrink: 0; text-align: right; }
  .dash-bar { flex: 1; height: 10px; background: var(--panel-2); border-radius: 6px; overflow: hidden; }
  .dash-bar-fill { height: 100%; border-radius: 6px; }
  .dash-bar-caption { display: block; margin-top: 8px; font-size: 12px; color: var(--muted); }
```

- [ ] **Step 2: `salesBalanceFor` + variável de período selecionado + `renderSalesCard`, logo antes de `renderDashboard`**

Old:
```js
function renderDashboard() {
```
New:
```js
function salesBalanceFor(days) {
  const since = Date.now() - days * 86400000;
  const sold = appState.properties.filter(p => p.status === 'vendido' && p.updatedAt >= since);
  const rented = appState.properties.filter(p => p.status === 'alugado' && p.updatedAt >= since);
  const soldTotal = sold.reduce((sum, p) => sum + (p.salePrice || 0), 0);
  const rentedTotal = rented.reduce((sum, p) => sum + (p.rentPrice || 0), 0);
  return { soldCount: sold.length, soldTotal, rentedCount: rented.length, rentedTotal };
}

let dashSalesPeriodDays = 30;

function renderSalesCard(days) {
  dashSalesPeriodDays = days;
  const { soldCount, soldTotal, rentedCount, rentedTotal } = salesBalanceFor(days);
  const maxVal = Math.max(soldTotal, rentedTotal, 1);
  const soldPct = Math.round((soldTotal / maxVal) * 100);
  const rentedPct = Math.round((rentedTotal / maxVal) * 100);
  $('dashSalesCard').innerHTML = `
    <div class="dash-card">
      <h3>Balanço de vendas</h3>
      <select id="dashPeriod" class="dash-period">
        <option value="30" ${days === 30 ? 'selected' : ''}>Mensal</option>
        <option value="90" ${days === 90 ? 'selected' : ''}>Trimestral</option>
        <option value="180" ${days === 180 ? 'selected' : ''}>Semestral</option>
        <option value="365" ${days === 365 ? 'selected' : ''}>Anual</option>
      </select>
      <div class="dash-bar-row"><span>Vendido (${soldCount})</span><div class="dash-bar"><div class="dash-bar-fill" style="width:${soldPct}%;background:var(--accent);"></div></div><strong>${formatCurrency(soldTotal)}</strong></div>
      <div class="dash-bar-row"><span>Alugado (${rentedCount})</span><div class="dash-bar"><div class="dash-bar-fill" style="width:${rentedPct}%;background:var(--ok-fg);"></div></div><strong>${formatCurrency(rentedTotal)}</strong></div>
    </div>`;
  $('dashPeriod').addEventListener('change', (e) => renderSalesCard(Number(e.target.value)));
}

function renderLaunchCard() {
  const launches = appState.properties.filter(p => p.isLaunch);
  if (!launches.length) { $('dashLaunchCard').innerHTML = ''; return; }
  const totalUnits = launches.reduce((s, p) => s + (p.totalUnits || 0), 0);
  const availableUnits = launches.reduce((s, p) => s + (p.availableUnits || 0), 0);
  const soldUnits = totalUnits - availableUnits;
  const pct = totalUnits > 0 ? Math.round((soldUnits / totalUnits) * 100) : 0;
  $('dashLaunchCard').innerHTML = `
    <div class="dash-card">
      <h3>Unidades de lançamento</h3>
      <p>${availableUnits} de ${totalUnits} disponíveis em ${launches.length} lançamento${launches.length > 1 ? 's' : ''}</p>
      <div class="dash-bar"><div class="dash-bar-fill" style="width:${pct}%;background:var(--accent);"></div></div>
      <span class="dash-bar-caption">${soldUnits} unidades vendidas (${pct}%)</span>
    </div>`;
}

function renderDashboard() {
```

- [ ] **Step 3: `renderDashboard` chama as 2 funções novas em vez de zerar os containers**

Old:
```js
  $('dashSalesCard').innerHTML = '';
  $('dashLaunchCard').innerHTML = '';
  $('dashFeatured').innerHTML = '';
  $('dashTableWrap').innerHTML = '';
}
```
New:
```js
  renderSalesCard(dashSalesPeriodDays);
  renderLaunchCard();
  $('dashFeatured').innerHTML = '';
  $('dashTableWrap').innerHTML = '';
}
```

- [ ] **Step 4: Teste manual — período recalcula, seleção persiste, card de lançamento**

Via Browser pane: criar um imóvel com `status: 'vendido'` (editar o
status de um existente e salvar, ou criar um novo já como vendido) e
`updatedAt` recente (é automático, `savePropertyForm` já seta), confirmar
que ele aparece no total "Vendido" do período Mensal (`$('dashPeriod').value === '30'`
por padrão). Trocar o select pra "Anual" via `javascript_tool`
(`$('dashPeriod').value = '365'; $('dashPeriod').dispatchEvent(new Event('change'))`),
confirmar que o total não muda (mesmo imóvel, dentro da janela maior) e
que criar/editar OUTRO imóvel em seguida (disparando `properties-updated`)
NÃO reseta o select de volta pra "Mensal" (`$('dashPeriod').value` continua
`'365'` depois). Confirmar que `#dashLaunchCard` só aparece com conteúdo
se existir imóvel `isLaunch: true` (testar com 0 lançamentos = card vazio,
depois criar 1 = card aparece com os números certos).

- [ ] **Step 5: Rodar self-check no console**

`read_console_messages()` — sem `ASSERT FAIL`.

- [ ] **Step 6: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: balanço de vendas por período e card de unidades de lançamento no dashboard"
```

---

### Task 3: Imóvel em destaque + tabela de recentes

**Files:**
- Modify: `mapa-imoveis/index.html` (CSS; JS: `escapeHtml`,
  `renderFeaturedProperty`, `renderRecentTable`, extensão de
  `renderDashboard`)

**Interfaces:**
- Consumes: `renderDashboard` (Tasks 1-2) — substitui as linhas
  `$('dashFeatured').innerHTML = ''` / `$('dashTableWrap').innerHTML = ''`.
  `priceLabelFor`, `TYPE_ICONS`, `ICONS`, `STATUS_COLORS`,
  `STATUS_TEXT_COLORS`, `STATUS_LABELS`, `UNIT_TYPE_LABELS`,
  `openPropertyDetail` (já existentes).
- Produces: `escapeHtml(str)` (helper global, reutilizável por qualquer
  código futuro que precise inserir texto livre em `innerHTML`).

- [ ] **Step 1: CSS do destaque e da tabela, logo após `.dash-main button.ghost, .dash-side button.ghost`**

Old:
```css
  .dash-main button.ghost, .dash-side button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
```
New:
```css
  .dash-main button.ghost, .dash-side button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
  .dash-featured-photo { width: 100%; height: 160px; object-fit: cover; border-radius: 10px; background: var(--panel-2); display: flex; align-items: center; justify-content: center; fill: var(--muted); }
  .dash-featured-price { display: block; font-size: 18px; margin-top: 12px; }
  .dash-featured-specs { display: flex; gap: 10px; margin: 8px 0 14px; font-size: 12px; color: var(--text); flex-wrap: wrap; }
  .dash-featured-specs span { display: flex; align-items: center; gap: 4px; fill: var(--muted); }
  .dash-table { width: 100%; border-collapse: collapse; font-size: 13px; }
  .dash-table th { text-align: left; color: var(--muted); font-size: 11px; text-transform: uppercase; padding: 6px 8px; border-bottom: 1px solid var(--line); }
  .dash-table td { padding: 8px; border-bottom: 1px solid var(--line); }
```

- [ ] **Step 2: `escapeHtml` + `renderFeaturedProperty` + `renderRecentTable`, logo antes de `renderDashboard`**

Old:
```js
function renderDashboard() {
```
New:
```js
function escapeHtml(str) {
  const div = document.createElement('div');
  div.textContent = str == null ? '' : String(str);
  return div.innerHTML;
}

function renderFeaturedProperty() {
  const sorted = [...appState.properties].sort((a, b) => (b.createdAt || 0) - (a.createdAt || 0));
  const featured = sorted[0];
  if (!featured) { $('dashFeatured').innerHTML = ''; return; }
  const photoUrls = (featured.photos || []).map(b => URL.createObjectURL(b));
  const photoHtml = photoUrls.length
    ? `<img src="${photoUrls[0]}" class="dash-featured-photo">`
    : `<div class="dash-featured-photo">${TYPE_ICONS[featured.unitType] || ICONS.house}</div>`;
  const specs = [
    featured.rooms ? `<span>${ICONS.bed}${featured.rooms}</span>` : '',
    featured.bathrooms ? `<span>${ICONS.bath}${featured.bathrooms}</span>` : '',
    featured.constructedArea ? `<span>${ICONS.area}${featured.constructedArea}m²</span>` : ''
  ].filter(Boolean).join('');
  $('dashFeatured').innerHTML = `
    <div class="dash-card">
      <h3>Último imóvel adicionado</h3>
      ${photoHtml}
      <strong class="dash-featured-price">${priceLabelFor(featured)}</strong>
      <div class="dash-featured-specs">${specs}</div>
      <button type="button" onclick="openPropertyDetail('${featured.id}')">Ver detalhes</button>
    </div>`;
}

function renderRecentTable() {
  const recent = [...appState.properties].sort((a, b) => (b.createdAt || 0) - (a.createdAt || 0)).slice(0, 10);
  if (!recent.length) { $('dashTableWrap').innerHTML = ''; return; }
  $('dashTableWrap').innerHTML = `
    <div class="dash-card">
      <h3>Imóveis recentes</h3>
      <table class="dash-table">
        <thead><tr><th>Imóvel</th><th>Tipo</th><th>Corretor</th><th>Preço</th><th>Status</th><th></th></tr></thead>
        <tbody>${recent.map(p => `
          <tr>
            <td>${escapeHtml(p.label || (p.address && p.address.street) || '—')}</td>
            <td>${UNIT_TYPE_LABELS[p.unitType] || p.unitType}</td>
            <td>${escapeHtml(p.agentResponsible || '—')}</td>
            <td>${priceLabelFor(p)}</td>
            <td><span class="list-item-status" style="background:${STATUS_COLORS[p.status] || '#8b9bab'}22;color:${STATUS_TEXT_COLORS[p.status] || '#5a6270'};">${STATUS_LABELS[p.status] || p.status}</span></td>
            <td><button type="button" class="ghost" onclick="openPropertyDetail('${p.id}')">Ver</button></td>
          </tr>`).join('')}</tbody>
      </table>
    </div>`;
}

function renderDashboard() {
```

- [ ] **Step 3: `renderDashboard` chama as 2 funções novas**

Old:
```js
  renderSalesCard(dashSalesPeriodDays);
  renderLaunchCard();
  $('dashFeatured').innerHTML = '';
  $('dashTableWrap').innerHTML = '';
}
```
New:
```js
  renderSalesCard(dashSalesPeriodDays);
  renderLaunchCard();
  renderFeaturedProperty();
  renderRecentTable();
}
```

- [ ] **Step 4: Teste manual — destaque, tabela, XSS na tabela**

Via Browser pane: criar 2 imóveis em sequência; confirmar que
`#dashFeatured` mostra sempre o 2º (mais recente por `createdAt`). Criar
um 3º com `f_label` ou `f_agentResponsible` contendo
`<img src=x onerror="window.__xss=true">`, confirmar na tabela que
`window.__xss` não vira `true` e que a célula mostra o texto literal
(`document.querySelector('.dash-table td').textContent` contém o texto
cru, `.innerHTML` não contém `<img`). Clicar no botão "Ver" de uma linha
da tabela, confirmar que abre a página de detalhe do imóvel certo.

- [ ] **Step 5: Rodar self-check no console**

`read_console_messages()` — sem `ASSERT FAIL`.

- [ ] **Step 6: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: imóvel em destaque e tabela de recentes no dashboard"
```

---

## Self-Review

**Cobertura do spec:**
- Dashboard como aba padrão → Task 1. ✅
- Fix de navegação da página de detalhe (voltar pro lugar certo) →
  Task 1. ✅
- KPIs → Task 1. ✅
- Balanço de vendas por período (janela móvel, proxy `updatedAt`) →
  Task 2. ✅
- Unidades de lançamento → Task 2. ✅
- Imóvel em destaque (último adicionado) → Task 3. ✅
- Tabela de recentes (sem coluna Views) → Task 3. ✅
- `escapeHtml` pra não repetir o XSS da fase 3 → Task 3. ✅
- Fora de escopo (mini-mapa, campo novo, fix do innerHTML pré-existente
  de popup/lista/detalhe, gráfico dia-a-dia) → nenhuma task toca nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo.

**Consistência de nomes:** `renderDashboard()` é reescrita 3 vezes ao
longo do plano (uma por task), sempre no MESMO bloco Old/New que a task
anterior deixou — conferido que cada Old bate exatamente com o New da
task anterior (Task 2 Step 3's Old é o New da Task 1 Step 5; Task 3
Step 3's Old é o New da Task 2 Step 3). `dashSalesPeriodDays` só
declarada uma vez (Task 2 Step 2), lida em `renderDashboard` sem
redeclarar. `closeDetailView()` definida uma vez (Task 1), usada em 2
lugares (`btnDetailBack` e o rodapé de excluir) sem duplicar lógica.

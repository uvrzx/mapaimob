# Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Clicar num imóvel (foto/preço do card da lista, ou "Ver detalhes"
no popup) abre uma página de detalhe cheia (título, endereço, preço,
specs, galeria, descrição, comodidades, unidades de lançamento), estilo
imagem 2, navegável sem backend via troca de view.

**Architecture:** Continua em `mapa-imoveis/index.html`. Task 1 entrega a
"casca" navegável com o conteúdo essencial (título/endereço/preço/status/
specs/galeria/Editar/Voltar) — já é uma página de detalhe funcional e
testável sozinha. Task 2 acrescenta o conteúdo restante (descrição,
comodidades, lançamento, excluir) dentro de containers vazios que a Task 1
deixa prontos, sem re-tocar a navegação.

**Tech Stack:** Mesmo stack — HTML/CSS/JS vanilla + Alpine.js + Leaflet,
sem framework de teste. Verificação via Browser pane
(`preview_start({name: "mapa-imoveis"})`) + `console.assert`.

## Global Constraints

- Arquivo único: `mapa-imoveis/index.html`. Sem novos arquivos/dependências.
- Sem router real — `#detailView` segue o mesmo padrão de `#mapView`/
  `#financingView` (mostrar/esconder via `style.display`).
- IDs de campos/elementos existentes não podem mudar.
- IndexedDB não funciona em `file://` — testar via `preview_start`.
- `.property-popup-actions` e `.unit-stepper` (CSS) já existem — reusar,
  não recriar.

---

### Task 1: Casca navegável + conteúdo essencial

**Files:**
- Modify: `mapa-imoveis/index.html` (CSS, HTML de `#detailView`, LOGIC
  `priceLabelFor`, `buildPopupHtml`/`renderList` (usar a função + gatilhos),
  nova função `openPropertyDetail`, listeners de navegação em LAYOUT)

**Interfaces:**
- Produces: `priceLabelFor(property)` (função global, retorna a string de
  preço formatada — usada por `buildPopupHtml`, `renderList` e
  `openPropertyDetail`). `openPropertyDetail(id)` (função global, usada via
  `onclick` inline, mesmo padrão de `deleteProperty`/`openPropertyForm`).
  Elementos `#detailExtra` e `#detailFooter` (containers vazios dentro de
  `#detailView`, que a Task 2 vai popular — Task 1 só garante que existem e
  ficam vazios a cada abertura).
- Consumes: `appState.properties`, `markersById` (MAP), `TYPE_ICONS`,
  `ICONS`, `STATUS_COLORS`, `STATUS_TEXT_COLORS`, `STATUS_LABELS`,
  `UNIT_TYPE_LABELS`, `formatCurrency` (todos já existentes).

- [ ] **Step 1: `priceLabelFor` — extrai a lógica de preço duplicada, logo após `formatCurrency`**

Old:
```js
function formatCurrency(value) {
  if (value == null || isNaN(value)) return 'R$ 0,00';
  return 'R$ ' + Number(value).toLocaleString('pt-BR', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}
```
New:
```js
function formatCurrency(value) {
  if (value == null || isNaN(value)) return 'R$ 0,00';
  return 'R$ ' + Number(value).toLocaleString('pt-BR', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}

function priceLabelFor(property) {
  const price = property.dealType === 'aluguel' ? property.rentPrice : property.salePrice;
  return property.dealType === 'aluguel' ? formatCurrency(price) + '/mês' : formatCurrency(price);
}
```

- [ ] **Step 2: `buildPopupHtml` usa `priceLabelFor` e ganha o botão "Ver detalhes"**

Old:
```js
function buildPopupHtml(property) {
  const price = property.dealType === 'aluguel' ? property.rentPrice : property.salePrice;
  const priceLabel = property.dealType === 'aluguel' ? formatCurrency(price) + '/mês' : formatCurrency(price);
  const addr = property.address || {};
```
New:
```js
function buildPopupHtml(property) {
  const priceLabel = priceLabelFor(property);
  const addr = property.address || {};
```

Old:
```js
        <div class="property-popup-actions">
          <button type="button" onclick="openPropertyForm(null, appState.properties.find(p => p.id === '${property.id}'))">Editar</button>
          <button type="button" class="ghost" onclick="deleteProperty('${property.id}')">Excluir</button>
        </div>
```
New:
```js
        <div class="property-popup-actions">
          <button type="button" onclick="openPropertyDetail('${property.id}')">Ver detalhes</button>
          <button type="button" class="ghost" onclick="openPropertyForm(null, appState.properties.find(p => p.id === '${property.id}'))">Editar</button>
          <button type="button" class="ghost" onclick="deleteProperty('${property.id}')">Excluir</button>
        </div>
```

- [ ] **Step 3: `renderList` usa `priceLabelFor` e ganha os gatilhos de detalhe (foto + preço)**

Old:
```js
  wrap.innerHTML = properties.map(p => {
    const price = p.dealType === 'aluguel' ? p.rentPrice : p.salePrice;
    const priceLabel = p.dealType === 'aluguel' ? formatCurrency(price) + '/mês' : formatCurrency(price);
    const addr = p.address || {};
```
New:
```js
  wrap.innerHTML = properties.map(p => {
    const priceLabel = priceLabelFor(p);
    const addr = p.address || {};
```

Old:
```js
    return `<div class="list-item" data-id="${p.id}">
      ${photoUrl ? `<img src="${photoUrl}" class="list-item-photo">` : `<div class="list-item-photo list-item-photo-empty">${TYPE_ICONS[p.unitType] || ICONS.house}</div>`}
      <div class="list-item-body">
        <div class="list-item-top">
          <strong>${priceLabel}</strong>
```
New:
```js
    return `<div class="list-item" data-id="${p.id}">
      ${photoUrl ? `<img src="${photoUrl}" class="list-item-photo" onclick="event.stopPropagation(); openPropertyDetail('${p.id}')">` : `<div class="list-item-photo list-item-photo-empty" onclick="event.stopPropagation(); openPropertyDetail('${p.id}')">${TYPE_ICONS[p.unitType] || ICONS.house}</div>`}
      <div class="list-item-body">
        <div class="list-item-top">
          <strong onclick="event.stopPropagation(); openPropertyDetail('${p.id}')" style="cursor:pointer;">${priceLabel}</strong>
```

- [ ] **Step 4: CSS de `#detailView`, logo após a regra de `#financingFrame`**

Old:
```css
  #financingFrame { width: 100%; height: 100%; border: 0; display: block; }
```
New:
```css
  #financingFrame { width: 100%; height: 100%; border: 0; display: block; }
  #detailView { display: none; flex: 1; min-height: 0; overflow-y: auto; padding: 28px 32px; }
  #detailView > button, .detail-main button { background: var(--accent); color: #fff; border: 0; padding: 8px 16px; border-radius: 8px; font-weight: 700; cursor: pointer; font-size: 13px; }
  #detailView > button.ghost, .detail-main button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
  #btnDetailBack { margin-bottom: 20px; }
  .detail-grid { display: grid; grid-template-columns: 1.1fr 1fr; gap: 32px; align-items: start; }
  .detail-title { font-size: 24px; margin: 0 0 4px; }
  .detail-address { color: var(--muted); font-size: 14px; margin: 0 0 16px; }
  .detail-price-row { display: flex; align-items: center; gap: 12px; margin-bottom: 16px; flex-wrap: wrap; }
  .detail-price { font-size: 22px; font-weight: 700; }
  .detail-specs { display: flex; gap: 20px; margin-bottom: 20px; font-size: 14px; color: var(--text); flex-wrap: wrap; }
  .detail-specs span { display: flex; align-items: center; gap: 6px; fill: var(--muted); }
  .detail-section { margin-top: 24px; }
  .detail-section h3 { font-size: 15px; margin: 0 0 10px; }
  .detail-hero { width: 100%; height: 320px; object-fit: cover; border-radius: 16px; background: var(--panel-2); display: flex; align-items: center; justify-content: center; fill: var(--muted); }
  .detail-thumbs { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 12px; }
  .detail-thumbs img { width: 100%; height: 140px; object-fit: cover; border-radius: 12px; }
```

Nota sobre a ordem: `.detail-main button` e `.unit-stepper button` (definida
mais abaixo no arquivo, fase 2) têm a mesma especificidade CSS (1 classe +
1 elemento). Como `.unit-stepper button` vem DEPOIS no arquivo, ela vence
o empate e o stepper de lançamento (Task 2) mantém os botões pequenos e
circulares mesmo estando dentro de `.detail-main` — não precisa (e não
deve) adicionar `!important` nem reordenar nada pra isso funcionar.

- [ ] **Step 5: HTML de `#detailView`, sibling de `#financingView`**

Old:
```html
  <div id="financingView">
    <iframe id="financingFrame" title="Simulador de financiamento"></iframe>
  </div>
</div>
```
New:
```html
  <div id="financingView">
    <iframe id="financingFrame" title="Simulador de financiamento"></iframe>
  </div>
  <div id="detailView">
    <button type="button" id="btnDetailBack" class="ghost">← Voltar</button>
    <div class="detail-grid">
      <div class="detail-main">
        <h1 class="detail-title" id="detailTitle"></h1>
        <p class="detail-address" id="detailAddress"></p>
        <div class="detail-price-row">
          <span class="detail-price" id="detailPrice"></span>
          <span class="property-popup-status" id="detailStatus"></span>
          <button type="button" id="btnDetailEdit">Editar imóvel</button>
        </div>
        <div class="detail-specs" id="detailSpecs"></div>
        <div id="detailExtra"></div>
        <div id="detailFooter"></div>
      </div>
      <div class="detail-gallery" id="detailGallery"></div>
    </div>
  </div>
</div>
```

- [ ] **Step 6: `openPropertyDetail`, logo após `buildMarkerIcon` (seção MAP, antes de `buildPopupHtml`)**

Old:
```js
function buildPopupHtml(property) {
```
New:
```js
function openPropertyDetail(id) {
  const property = appState.properties.find(p => p.id === id);
  if (!property) return;
  const addr = property.address || {};
  $('detailTitle').textContent = property.label || `${UNIT_TYPE_LABELS[property.unitType] || property.unitType} em ${addr.neighborhood || addr.city || ''}`;
  $('detailAddress').textContent = `${addr.street || ''} ${addr.number || ''} — ${addr.neighborhood || ''}, ${addr.city || ''}`;
  $('detailPrice').textContent = priceLabelFor(property);
  const statusColor = STATUS_COLORS[property.status] || '#8b9bab';
  const statusTextColor = STATUS_TEXT_COLORS[property.status] || '#5a6270';
  $('detailStatus').textContent = STATUS_LABELS[property.status] || property.status;
  $('detailStatus').style.background = `${statusColor}22`;
  $('detailStatus').style.color = statusTextColor;
  $('btnDetailEdit').onclick = () => openPropertyForm(null, property);
  const specs = [
    property.rooms ? `<span>${ICONS.bed}${property.rooms} quartos</span>` : '',
    property.suites ? `<span>${ICONS.bed}${property.suites} suítes</span>` : '',
    property.bathrooms ? `<span>${ICONS.bath}${property.bathrooms} banheiros</span>` : '',
    property.parkingSpaces ? `<span>${ICONS.car}${property.parkingSpaces} vagas</span>` : '',
    property.constructedArea ? `<span>${ICONS.area}${property.constructedArea}m²</span>` : ''
  ].filter(Boolean).join('');
  $('detailSpecs').innerHTML = specs;
  const photoUrls = (property.photos || []).map(b => URL.createObjectURL(b));
  $('detailGallery').innerHTML = photoUrls.length
    ? `<img src="${photoUrls[0]}" class="detail-hero">` + (photoUrls.length > 1 ? `<div class="detail-thumbs">${photoUrls.slice(1, 3).map(u => `<img src="${u}">`).join('')}</div>` : '')
    : `<div class="detail-hero">${TYPE_ICONS[property.unitType] || ICONS.house}</div>`;
  $('detailExtra').innerHTML = '';
  $('detailFooter').innerHTML = '';
  $('mapView').style.display = 'none';
  $('financingView').style.display = 'none';
  $('detailView').style.display = 'block';
}

function buildPopupHtml(property) {
```

Nota: `$('detailExtra')`/`$('detailFooter')` ficam vazios nesta task de
propósito — a Task 2 substitui essas 2 linhas por lógica que os preenche.

- [ ] **Step 7: Navegação — `btnDetailBack` e robustez nas abas (seção LAYOUT, junto dos outros listeners de view)**

Old:
```js
  $('tabMap').addEventListener('click', () => {
    $('mapView').style.display = '';
    $('financingView').style.display = 'none';
    $('tabMap').classList.add('active');
    $('tabFinancing').classList.remove('active');
    map.invalidateSize();
  });
  $('tabFinancing').addEventListener('click', () => {
    $('mapView').style.display = 'none';
    $('financingView').style.display = 'flex';
    $('tabFinancing').classList.add('active');
    $('tabMap').classList.remove('active');
    if (!$('financingFrame').src) $('financingFrame').src = 'financiamento.html';
  });
});
```
New:
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

- [ ] **Step 8: Teste manual — abrir detalhe pelos 2 gatilhos, ver conteúdo essencial, voltar**

Via Browser pane com `preview_start({name: "mapa-imoveis"})` e pelo menos
1 imóvel salvo (criar um via `javascript_tool` se precisar): clicar na foto
do card da lista via `.click()`, confirmar `getComputedStyle($('detailView')).display !== 'none'`
e `getComputedStyle($('mapView')).display === 'none'`, e que
`$('detailTitle').textContent`/`$('detailPrice').textContent` não estão
vazios. Clicar `$('btnDetailBack')`, confirmar que volta pra `#mapView`.
Repetir abrindo pelo botão "Ver detalhes" do popup. Clicar
`$('btnDetailEdit')`, confirmar que `#propertyForm` abre com os dados do
mesmo imóvel (`$('f_id').value` bate com o id).

- [ ] **Step 9: Rodar self-check no console**

`read_console_messages()` — esperar `[self-check] Lógica de negócio OK`
sem `ASSERT FAIL` (nenhuma lógica nova de self-check nesta task, só
confirma que nada quebrou).

- [ ] **Step 10: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: página de detalhe do imóvel (título, preço, specs, galeria, editar)"
```

---

### Task 2: Descrição, comodidades, unidades de lançamento e excluir

**Files:**
- Modify: `mapa-imoveis/index.html` (CONFIG: `AMENITY_CHECK_ICON` +
  `AMENITY_LABELS`; FILTERS: `filterApp().amenityOptions`; `deleteProperty`;
  `openPropertyDetail`'s `detailExtra`/`detailFooter`; CSS de
  `.detail-amenities`)

**Interfaces:**
- Consumes: `openPropertyDetail` (Task 1) — estende o corpo da função,
  substituindo as 2 linhas `$('detailExtra').innerHTML = ''` /
  `$('detailFooter').innerHTML = ''`. `appState.condosById` (já existe).
- Produces: `AMENITY_LABELS` (constante global `{value: label}`),
  `AMENITY_CHECK_ICON` (string SVG). `deleteProperty(id)` passa a
  `return true`/`return false` (era `return;` implícito) — mudança
  compatível, o único outro caller (popup) ignora o retorno.

- [ ] **Step 1: `AMENITY_CHECK_ICON` + `AMENITY_LABELS`, logo após `TYPE_ICONS`**

Old:
```js
const TYPE_ICONS = {
  casa: ICONS.house, sobrado: ICONS.sobrado,
  apartamento: ICONS.building, cobertura: ICONS.cobertura,
  comercial: ICONS.store, terreno: ICONS.plot
};
```
New:
```js
const TYPE_ICONS = {
  casa: ICONS.house, sobrado: ICONS.sobrado,
  apartamento: ICONS.building, cobertura: ICONS.cobertura,
  comercial: ICONS.store, terreno: ICONS.plot
};
const AMENITY_CHECK_ICON = '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6L9 17l-5-5"/></svg>';
const AMENITY_LABELS = {
  pool: 'Piscina', gym: 'Academia', partyRoom: 'Salão de festas', playground: 'Playground',
  petArea: 'Área pet', security24h: 'Portaria 24h', elevator: 'Elevador', gatedCommunity: 'Condomínio fechado'
};
```

- [ ] **Step 2: `filterApp().amenityOptions` reaproveita `AMENITY_LABELS`**

Old:
```js
    amenityOptions: [
      { value: 'pool', label: 'Piscina' }, { value: 'gym', label: 'Academia' },
      { value: 'partyRoom', label: 'Salão de festas' }, { value: 'playground', label: 'Playground' },
      { value: 'petArea', label: 'Área pet' }, { value: 'security24h', label: 'Portaria 24h' },
      { value: 'elevator', label: 'Elevador' }, { value: 'gatedCommunity', label: 'Condomínio fechado' }
    ],
```
New:
```js
    amenityOptions: Object.entries(AMENITY_LABELS).map(([value, label]) => ({ value, label })),
```

- [ ] **Step 3: `deleteProperty` sinaliza se realmente excluiu**

Old:
```js
async function deleteProperty(id) {
  if (!confirm('Excluir este imóvel?')) return;
  await dbDelete('properties', id);
  appState.properties = appState.properties.filter(p => p.id !== id);
  removeMarker(id);
  window.dispatchEvent(new CustomEvent('properties-updated'));
}
```
New:
```js
async function deleteProperty(id) {
  if (!confirm('Excluir este imóvel?')) return false;
  await dbDelete('properties', id);
  appState.properties = appState.properties.filter(p => p.id !== id);
  removeMarker(id);
  window.dispatchEvent(new CustomEvent('properties-updated'));
  return true;
}
```

- [ ] **Step 4: CSS de `.detail-amenities`, logo após `.detail-thumbs img`**

Old:
```css
  .detail-thumbs img { width: 100%; height: 140px; object-fit: cover; border-radius: 12px; }
```
New:
```css
  .detail-thumbs img { width: 100%; height: 140px; object-fit: cover; border-radius: 12px; }
  .detail-amenities { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
  .detail-amenities span { display: flex; align-items: center; gap: 8px; font-size: 13px; color: var(--text); fill: none; stroke: var(--accent); }
```

- [ ] **Step 5: `openPropertyDetail` ganha descrição, comodidades, lançamento e rodapé de excluir**

Old:
```js
  $('detailExtra').innerHTML = '';
  $('detailFooter').innerHTML = '';
  $('mapView').style.display = 'none';
```
New:
```js
  const notes = property.notes || '';
  const isLongNotes = notes.length > 220;
  const shortNotes = isLongNotes ? notes.slice(0, 220) + '…' : notes;
  const descriptionHtml = notes
    ? `<div class="detail-section"><h3>Descrição</h3><p id="detailDescText">${shortNotes}</p>${isLongNotes ? `<button type="button" class="ghost" id="btnDetailDescToggle">Mostrar mais</button>` : ''}</div>`
    : '';
  const condo = property.condoId ? appState.condosById[property.condoId] : null;
  const activeAmenities = condo ? Object.keys(AMENITY_LABELS).filter(k => condo[k]) : [];
  const amenitiesHtml = activeAmenities.length
    ? `<div class="detail-section"><h3>Comodidades do condomínio</h3><div class="detail-amenities">${activeAmenities.map(k => `<span>${AMENITY_CHECK_ICON}${AMENITY_LABELS[k]}</span>`).join('')}</div></div>`
    : '';
  const launchHtml = property.isLaunch
    ? `<div class="detail-section"><h3>Unidades</h3><div class="unit-stepper"><button type="button" onclick="adjustAvailableUnits('${property.id}', -1); openPropertyDetail('${property.id}')">−</button><span>${property.availableUnits} de ${property.totalUnits} disponíveis</span><button type="button" onclick="adjustAvailableUnits('${property.id}', 1); openPropertyDetail('${property.id}')">+</button></div></div>`
    : '';
  $('detailExtra').innerHTML = descriptionHtml + amenitiesHtml + launchHtml;
  if (isLongNotes) {
    $('btnDetailDescToggle').addEventListener('click', () => {
      const expanded = $('btnDetailDescToggle').textContent === 'Mostrar menos';
      $('detailDescText').textContent = expanded ? shortNotes : notes;
      $('btnDetailDescToggle').textContent = expanded ? 'Mostrar mais' : 'Mostrar menos';
    });
  }
  $('detailFooter').innerHTML = `<button type="button" class="ghost" onclick="deleteProperty('${property.id}').then(deleted => { if (deleted) { $('detailView').style.display = 'none'; $('mapView').style.display = ''; map.invalidateSize(); } })">Excluir imóvel</button>`;
  $('mapView').style.display = 'none';
```

- [ ] **Step 6: Teste manual — conteúdo completo + excluir pela página de detalhe**

Via Browser pane: usar um imóvel com `notes` longa (>220 chars), `condoId`
setado com pelo menos 1 amenidade `true`, e `isLaunch: true` (reaproveitar
os de fases anteriores ou criar um novo com todos os campos). Abrir o
detalhe, confirmar: `#detailExtra` contém descrição truncada + botão
"Mostrar mais"; clicar nele alterna pro texto completo e pro label
"Mostrar menos"; comodidades aparecem só as marcadas no condomínio;
stepper de unidades ajusta e a página se atualiza (chamando
`openPropertyDetail` de novo) refletindo o novo valor. Clicar "Excluir
imóvel", confirmar o dialog, verificar que o imóvel some de
`appState.properties` E que a view volta pro mapa automaticamente
(`getComputedStyle($('mapView')).display !== 'none'`).

- [ ] **Step 7: Rodar self-check no console**

`read_console_messages()` — `[self-check] Lógica de negócio OK` sem
`ASSERT FAIL`.

- [ ] **Step 8: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: descrição, comodidades, unidades e excluir na página de detalhe"
```

---

## Self-Review

**Cobertura do spec:**
- Gatilhos (popup + card da lista) → Task 1. ✅
- Layout 2 colunas, título/endereço/preço/status/specs/galeria/editar →
  Task 1. ✅
- Navegação sem router (view show/hide + voltar) → Task 1. ✅
- Descrição com show more/less → Task 2. ✅
- Comodidades (ícone genérico) → Task 2. ✅
- Stepper de lançamento → Task 2. ✅
- Excluir + retorno automático ao mapa → Task 2. ✅
- `priceLabelFor`/`AMENITY_LABELS` reuso → Task 1 / Task 2. ✅
- Fora de escopo (dashboard, campos novos, URL compartilhável, redesign de
  popup/lista além dos gatilhos) → nenhuma task toca nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo.

**Consistência de nomes:** `openPropertyDetail(id)` mesma assinatura nos 3
call sites (popup, lista×2, stepper de lançamento dentro do próprio
detalhe). `priceLabelFor(property)` mesmo nome/parâmetro nos 3 usos
(popup, lista, detalhe). `AMENITY_LABELS`/`AMENITY_CHECK_ICON` só
definidos uma vez (Task 2 Step 1), consumidos por `filterApp()` e por
`openPropertyDetail` sem duplicar o mapeamento.

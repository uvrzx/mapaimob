# Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Condomínio ganha endereço/fotos/descrição e vira "prédio"
clicável no mapa; HUD flutuante alterna entre ver unidades ou prédios;
pin de prédio mostra foto + % de preenchimento; popup do prédio mostra
galeria, descrição, amenidades e unidades agenciadas (linkando pra
página de detalhe já existente).

**Architecture:** Continua em `mapa-imoveis/index.html`. Task 1 entrega o
modelo de dados (condomínio com endereço/fotos/descrição) e generaliza
2 funções existentes (`geocodeAddress`, `renderPhotoThumbnails`) pra
servir tanto o formulário de imóvel quanto o de condomínio — sem
duplicar lógica. Task 2 entrega o HUD + a camada de marcadores de
condomínio com popup mínimo. Task 3 completa o popup (galeria, seções,
unidades agenciadas, botão Agenciar).

**Tech Stack:** Mesmo stack — HTML/CSS/JS vanilla + Alpine.js + Leaflet,
sem framework de teste, sem lib nova. Geocodificação reaproveita o
Nominatim já usado. Verificação via Browser pane
(`preview_start({name: "mapa-imoveis"})`) + `console.assert`.

## Global Constraints

- Arquivo único: `mapa-imoveis/index.html`. Sem novos arquivos/
  dependências.
- `condo.name`, `condo.description` e `condo.address.*` são texto livre
  — todo HTML novo que os renderiza passa por `escapeHtml()` (já existe,
  fase 4). Não repetir o XSS que a fase 3 corrigiu.
- Faixas de cor do badge de preenchimento (34%/67%) são simplificação
  deliberada (terços) — não é regra validada pelo cliente, documentada
  no spec. Não tratar como "bug" se o cliente pedir outra faixa depois.
- Fora de escopo: legenda de amenidades reais no mapa (POI externo),
  webhook ClickUp no botão Agenciar, popup dedicado de unidade (reusa
  `#detailView`), edição de condomínio existente.
- IDs de elementos existentes não podem mudar.
- IndexedDB não funciona em `file://` — testar via `preview_start`.

---

### Task 1: Condomínio ganha endereço, coordenadas, fotos e descrição

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML de `#newCondoFields`; JS:
  `geocodeAddress`, `renderPhotoThumbnails`, `handleCondoCreateIfNeeded`,
  `closePropertyForm`, listeners de `DOMContentLoaded`)

**Interfaces:**
- Produces: `condo.address`, `condo.coordinates`, `condo.description`,
  `condo.photos` (novos campos no registro salvo em `dbPut('condos', …)`).
  `geocodeAddress(prefix)` (generalizada, aceita `'f'` ou `'c'`).
  `renderFormPhotoThumbs()`, `renderCondoPhotoThumbs()` (wrappers de
  `renderPhotoThumbnails(photos, wrapId, onRemove)`, generalizada).
- Consumes: `dbPut`, `buildMarkerIcon`, `resizeImageToBlob` (já
  existentes).

- [ ] **Step 1: HTML — campos novos em `#newCondoFields`**

Old:
```html
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
```
New:
```html
      <div id="newCondoFields" style="display:none; margin-top:10px; padding:10px; background:var(--panel-2); border-radius:8px;">
        <label for="c_name">Nome do condomínio</label>
        <input id="c_name" type="text">
        <input type="hidden" id="c_lat">
        <input type="hidden" id="c_lng">
        <label for="c_street">Rua</label>
        <input id="c_street" type="text">
        <div class="row">
          <div>
            <label for="c_number">Número</label>
            <input id="c_number" type="text">
          </div>
          <div>
            <label for="c_neighborhood">Bairro</label>
            <input id="c_neighborhood" type="text">
          </div>
        </div>
        <label for="c_city">Cidade</label>
        <input id="c_city" type="text" value="Florianópolis">
        <button type="button" id="btnGeocodeCondo" class="ghost" style="width:100%;margin-top:6px;">🔍 Localizar endereço no mapa</button>
        <label for="c_description">Sobre o prédio</label>
        <textarea id="c_description" rows="2"></textarea>
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
        <label for="c_photos">Fotos do prédio</label>
        <input id="c_photos" type="file" accept="image/*" multiple>
        <div id="condoPhotoThumbs" style="margin-top:8px;"></div>
      </div>
```

- [ ] **Step 2: JS — generalizar `geocodeAddress` pra aceitar prefixo `f`/`c`**

Old:
```js
async function geocodeAddress() {
  const street = $('f_street').value.trim();
  const number = $('f_number').value.trim();
  const neighborhood = $('f_neighborhood').value.trim();
  const city = $('f_city').value.trim() || 'Florianópolis';
  if (!street) { alert('Preencha ao menos a rua antes de localizar.'); return; }
  const query = [street, number, neighborhood, city, 'SC', 'Brasil'].filter(Boolean).join(', ');
  const btn = $('btnGeocodeAddress');
  const originalLabel = btn.textContent;
  btn.disabled = true;
  btn.textContent = 'Buscando...';
  try {
    const url = 'https://nominatim.openstreetmap.org/search?format=json&limit=1&q=' + encodeURIComponent(query);
    const res = await fetch(url, { headers: { 'Accept-Language': 'pt-BR' } });
    const results = await res.json();
    if (!results.length) {
      alert('Endereço não encontrado. Ajuste manualmente clicando no mapa.');
      return;
    }
    const lat = parseFloat(results[0].lat), lng = parseFloat(results[0].lon);
    $('f_lat').value = lat;
    $('f_lng').value = lng;
    map.flyTo([lat, lng], 17, { duration: 0.8 });
    clearGeocodePreview();
    geocodePreviewMarker = L.marker([lat, lng], { icon: buildMarkerIcon('disponivel', $('f_unitType').value) }).addTo(markersLayer);
  } catch (err) {
    console.error(err);
    alert('Erro ao buscar endereço. Tente novamente.');
  } finally {
    btn.disabled = false;
    btn.textContent = originalLabel;
  }
}
```
New:
```js
const GEOCODE_CONFIG = {
  f: { btn: 'btnGeocodeAddress', preview: true },
  c: { btn: 'btnGeocodeCondo', preview: false }
};

async function geocodeAddress(prefix) {
  const street = $(`${prefix}_street`).value.trim();
  const number = $(`${prefix}_number`).value.trim();
  const neighborhood = $(`${prefix}_neighborhood`).value.trim();
  const city = $(`${prefix}_city`).value.trim() || 'Florianópolis';
  if (!street) { alert('Preencha ao menos a rua antes de localizar.'); return; }
  const query = [street, number, neighborhood, city, 'SC', 'Brasil'].filter(Boolean).join(', ');
  const { btn: btnId, preview } = GEOCODE_CONFIG[prefix];
  const btn = $(btnId);
  const originalLabel = btn.textContent;
  btn.disabled = true;
  btn.textContent = 'Buscando...';
  try {
    const url = 'https://nominatim.openstreetmap.org/search?format=json&limit=1&q=' + encodeURIComponent(query);
    const res = await fetch(url, { headers: { 'Accept-Language': 'pt-BR' } });
    const results = await res.json();
    if (!results.length) {
      alert('Endereço não encontrado. Ajuste manualmente clicando no mapa.');
      return;
    }
    const lat = parseFloat(results[0].lat), lng = parseFloat(results[0].lon);
    $(`${prefix}_lat`).value = lat;
    $(`${prefix}_lng`).value = lng;
    map.flyTo([lat, lng], 17, { duration: 0.8 });
    clearGeocodePreview();
    if (preview) geocodePreviewMarker = L.marker([lat, lng], { icon: buildMarkerIcon('disponivel', $('f_unitType').value) }).addTo(markersLayer);
  } catch (err) {
    console.error(err);
    alert('Erro ao buscar endereço. Tente novamente.');
  } finally {
    btn.disabled = false;
    btn.textContent = originalLabel;
  }
}
```

- [ ] **Step 3: JS — generalizar `renderPhotoThumbnails` e criar wrappers**

Old:
```js
/* === SECTION: PHOTOS === */
let formPhotos = [];

function resizeImageToBlob(file, maxWidth) {
```
New:
```js
/* === SECTION: PHOTOS === */
let formPhotos = [];
let condoFormPhotos = [];

function resizeImageToBlob(file, maxWidth) {
```

Old:
```js
function renderPhotoThumbnails() {
  const wrap = $('photoThumbs');
  wrap.innerHTML = '';
  formPhotos.forEach((blob, idx) => {
    const url = URL.createObjectURL(blob);
    const thumb = document.createElement('div');
    thumb.style.cssText = 'position:relative;display:inline-block;margin:4px;';
    thumb.innerHTML = `<img src="${url}" style="width:64px;height:64px;object-fit:cover;border-radius:6px;">
      <button type="button" data-idx="${idx}" style="position:absolute;top:-6px;right:-6px;width:18px;height:18px;padding:0;border-radius:50%;background:var(--warn-fg);color:#fff;border:0;cursor:pointer;font-size:11px;line-height:1;">×</button>`;
    wrap.appendChild(thumb);
  });
  wrap.querySelectorAll('button[data-idx]').forEach(btn => {
    btn.addEventListener('click', () => {
      formPhotos.splice(Number(btn.dataset.idx), 1);
      renderPhotoThumbnails();
    });
  });
}
```
New:
```js
function renderPhotoThumbnails(photos, wrapId, onRemove) {
  const wrap = $(wrapId);
  wrap.innerHTML = '';
  photos.forEach((blob, idx) => {
    const url = URL.createObjectURL(blob);
    const thumb = document.createElement('div');
    thumb.style.cssText = 'position:relative;display:inline-block;margin:4px;';
    thumb.innerHTML = `<img src="${url}" style="width:64px;height:64px;object-fit:cover;border-radius:6px;">
      <button type="button" data-idx="${idx}" style="position:absolute;top:-6px;right:-6px;width:18px;height:18px;padding:0;border-radius:50%;background:var(--warn-fg);color:#fff;border:0;cursor:pointer;font-size:11px;line-height:1;">×</button>`;
    wrap.appendChild(thumb);
  });
  wrap.querySelectorAll('button[data-idx]').forEach(btn => {
    btn.addEventListener('click', () => onRemove(Number(btn.dataset.idx)));
  });
}

function renderFormPhotoThumbs() {
  renderPhotoThumbnails(formPhotos, 'photoThumbs', (idx) => { formPhotos.splice(idx, 1); renderFormPhotoThumbs(); });
}

function renderCondoPhotoThumbs() {
  renderPhotoThumbnails(condoFormPhotos, 'condoPhotoThumbs', (idx) => { condoFormPhotos.splice(idx, 1); renderCondoPhotoThumbs(); });
}
```

- [ ] **Step 4: JS — `f_photos`/novo `c_photos` listener, chamando os wrappers**

Old:
```js
window.addEventListener('DOMContentLoaded', () => {
  $('f_photos').addEventListener('change', async (e) => {
    for (const file of e.target.files) {
      const blob = await resizeImageToBlob(file, 1600);
      formPhotos.push(blob);
    }
    renderPhotoThumbnails();
    e.target.value = '';
  });
});
```
New:
```js
window.addEventListener('DOMContentLoaded', () => {
  $('f_photos').addEventListener('change', async (e) => {
    for (const file of e.target.files) {
      const blob = await resizeImageToBlob(file, 1600);
      formPhotos.push(blob);
    }
    renderFormPhotoThumbs();
    e.target.value = '';
  });
  $('c_photos').addEventListener('change', async (e) => {
    for (const file of e.target.files) {
      const blob = await resizeImageToBlob(file, 1600);
      condoFormPhotos.push(blob);
    }
    renderCondoPhotoThumbs();
    e.target.value = '';
  });
});
```

- [ ] **Step 5: JS — `openPropertyForm` chama o wrapper novo**

Old:
```js
  formPhotos = p.photos ? p.photos.slice() : [];
  renderPhotoThumbnails();
  $('propertyForm').classList.add('open');
```
New:
```js
  formPhotos = p.photos ? p.photos.slice() : [];
  renderFormPhotoThumbs();
  $('propertyForm').classList.add('open');
```

- [ ] **Step 6: JS — `handleCondoCreateIfNeeded` grava os campos novos**

Old:
```js
async function handleCondoCreateIfNeeded() {
  const nameField = $('c_name');
  if ($('newCondoFields').style.display === 'none' || !nameField.value.trim()) return null;
  const condo = {
    id: 'c_' + Date.now() + '_' + Math.random().toString(36).slice(2, 8),
    name: nameField.value.trim(),
    pool: $('c_pool').checked,
    gym: $('c_gym').checked,
    partyRoom: $('c_partyRoom').checked,
    playground: $('c_playground').checked,
    petArea: $('c_petArea').checked,
    security24h: $('c_security24h').checked,
    elevator: $('c_elevator').checked,
    gatedCommunity: $('c_gatedCommunity').checked,
    monthlyFee: Number($('c_monthlyFee').value) || 0
  };
  await dbPut('condos', condo);
  appState.condos.push(condo);
  appState.condosById[condo.id] = condo;
  refreshCondoSelect();
  return condo.id;
}
```
New:
```js
async function handleCondoCreateIfNeeded() {
  const nameField = $('c_name');
  if ($('newCondoFields').style.display === 'none' || !nameField.value.trim()) return null;
  const condo = {
    id: 'c_' + Date.now() + '_' + Math.random().toString(36).slice(2, 8),
    name: nameField.value.trim(),
    address: {
      street: $('c_street').value.trim(),
      number: $('c_number').value.trim(),
      neighborhood: $('c_neighborhood').value.trim(),
      city: $('c_city').value.trim() || 'Florianópolis'
    },
    coordinates: ($('c_lat').value && $('c_lng').value) ? [Number($('c_lat').value), Number($('c_lng').value)] : null,
    description: $('c_description').value.trim(),
    photos: condoFormPhotos.slice(),
    pool: $('c_pool').checked,
    gym: $('c_gym').checked,
    partyRoom: $('c_partyRoom').checked,
    playground: $('c_playground').checked,
    petArea: $('c_petArea').checked,
    security24h: $('c_security24h').checked,
    elevator: $('c_elevator').checked,
    gatedCommunity: $('c_gatedCommunity').checked,
    monthlyFee: Number($('c_monthlyFee').value) || 0
  };
  await dbPut('condos', condo);
  appState.condos.push(condo);
  appState.condosById[condo.id] = condo;
  refreshCondoSelect();
  condoFormPhotos = [];
  return condo.id;
}
```

- [ ] **Step 7: JS — reset de `condoFormPhotos` ao fechar o formulário**

Old:
```js
function closePropertyForm() {
  $('propertyForm').classList.remove('open');
  $('formBackdrop').classList.remove('show');
  $('map').classList.remove('form-open');
  appState.addMode = false;
  appState.editingPropertyId = null;
  $('btnAddProperty').classList.remove('active');
  clearGeocodePreview();
}
```
New:
```js
function closePropertyForm() {
  $('propertyForm').classList.remove('open');
  $('formBackdrop').classList.remove('show');
  $('map').classList.remove('form-open');
  appState.addMode = false;
  appState.editingPropertyId = null;
  $('btnAddProperty').classList.remove('active');
  clearGeocodePreview();
  condoFormPhotos = [];
  $('newCondoFields').style.display = 'none';
}
```

- [ ] **Step 8: JS — bind dos botões de geocodificação com o prefixo certo**

Old:
```js
window.addEventListener('DOMContentLoaded', () => {
  $('propertyFormEl').addEventListener('submit', savePropertyForm);
  $('btnCancelForm').addEventListener('click', closePropertyForm);
  $('btnGeocodeAddress').addEventListener('click', geocodeAddress);
  $('formBackdrop').addEventListener('click', closePropertyForm);
```
New:
```js
window.addEventListener('DOMContentLoaded', () => {
  $('propertyFormEl').addEventListener('submit', savePropertyForm);
  $('btnCancelForm').addEventListener('click', closePropertyForm);
  $('btnGeocodeAddress').addEventListener('click', () => geocodeAddress('f'));
  $('btnGeocodeCondo').addEventListener('click', () => geocodeAddress('c'));
  $('formBackdrop').addEventListener('click', closePropertyForm);
```

- [ ] **Step 9: Teste manual — criar condomínio com endereço, foto e descrição**

Via Browser pane: abrir "+ Adicionar imóvel", clicar no mapa, no form
clicar "+ Novo condomínio", preencher nome + rua/número/bairro, clicar
"🔍 Localizar endereço no mapa" (confirma que `$('c_lat').value` e
`$('c_lng').value` preenchem), preencher "Sobre o prédio", anexar 1 foto
em "Fotos do prédio" (confirma que a miniatura aparece em
`#condoPhotoThumbs` e que o botão "×" remove), salvar o imóvel.
Confirmar via `dbGet('condos', id)` que o registro salvo tem `address`,
`coordinates`, `description` e `photos` (array com 1 Blob) preenchidos.
Confirmar que o formulário de imóvel (fotos/endereço do imóvel em si)
continua funcionando normalmente (sem regressão — criar um 2º imóvel
sem condomínio novo, só confirmando que `formPhotos`/`renderFormPhotoThumbs`
não quebrou).

- [ ] **Step 10: Rodar self-check no console**

`read_console_messages()` — `[self-check] Lógica de negócio OK` e
`[self-check] IndexedDB OK`, sem `ASSERT FAIL`.

- [ ] **Step 11: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: condomínio ganha endereço, coordenadas, fotos e descrição"
```

---

### Task 2: HUD flutuante (toggle Unidade/Condomínio) + pins de condomínio

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML de `#mapWrap`; CSS; JS:
  `appState`, `initMap`, novas funções de marcador de condomínio,
  `apply()` do Alpine, listener de `filters-applied`, novo listener de
  `DOMContentLoaded`)

**Interfaces:**
- Consumes: Task 1 (`condo.address`, `condo.coordinates`,
  `condo.photos`). `AMENITY_LABELS`, `ICONS.building`, `defaultFilters`
  (já existentes).
- Produces: `appState.mapMode`, `condoMarkersLayer`, `condoMarkersById`,
  `condoFillPercent(condoId, properties)`, `fillBadgeColor(pct)`,
  `condoMatchesFilters(condo, filters)`, `buildCondoMarkerIcon(condo,
  pct)`, `buildCondoPopupHtml(condo)` (versão mínima — Task 3 completa),
  `upsertCondoMarker`, `removeCondoMarker`, `renderCondoMarkers`,
  `lastFilters` (módulo-level, espelha os filtros atuais do Alpine pra
  uso fora do componente).

- [ ] **Step 1: HTML — barra HUD dentro de `#mapWrap`**

Old:
```html
    <div id="mapWrap">
    <div id="map"></div>
    </div>
```
New:
```html
    <div id="mapWrap">
    <div id="map"></div>
    <div id="mapHud">
      <div class="seg-group" id="mapModeToggle">
        <button type="button" data-mode="unidade" class="active">Unidade</button>
        <button type="button" data-mode="condominio">Condomínio</button>
      </div>
      <button type="button" id="btnHudFilter">Filtrar</button>
    </div>
    </div>
```

- [ ] **Step 2: CSS — `#mapHud` e pin/popup de condomínio**

Old:
```css
  #map.form-open .leaflet-control-zoom { margin-right: 390px; transition: margin-right .25s cubic-bezier(.16,1,.3,1); }
```
New:
```css
  #map.form-open .leaflet-control-zoom { margin-right: 390px; transition: margin-right .25s cubic-bezier(.16,1,.3,1); }
  #mapHud { position: absolute; top: 14px; left: 50%; transform: translateX(-50%); z-index: 1000; display: flex; align-items: center; gap: 8px; background: var(--panel); border-radius: 24px; padding: 4px; box-shadow: var(--shadow-soft); }
  #mapHud .seg-group { margin: 0; }
  #mapHud #btnHudFilter { background: var(--accent); color: #fff; border: 0; padding: 7px 16px; border-radius: 20px; font-weight: 700; font-size: 12px; cursor: pointer; }
```

Old:
```css
  .property-popup-actions button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }
```
New:
```css
  .property-popup-actions button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); font-weight: 600; }

  /* pin de condomínio */
  .condo-pin { position: relative; width: 52px; }
  .condo-pin-photo { width: 44px; height: 44px; border-radius: 50%; object-fit: cover; border: 3px solid var(--panel); box-shadow: var(--shadow-soft); display: block; margin: 0 auto; }
  .condo-pin-photo-empty { display: flex; align-items: center; justify-content: center; background: var(--panel-2); fill: var(--muted); }
  .condo-pin-badge { position: absolute; top: -8px; left: 50%; transform: translateX(-50%); color: #fff; font-size: 10px; font-weight: 700; padding: 2px 7px; border-radius: 10px; white-space: nowrap; box-shadow: var(--shadow-soft); }

  /* cartão de prédio (popup) */
  .condo-popup-photo { width: 100%; height: 130px; object-fit: cover; display: block; background: var(--panel-2); }
  .condo-popup-photo-empty { display: flex; align-items: center; justify-content: center; fill: var(--muted); }
  .condo-popup-body { padding: 12px 14px; }
  .condo-popup-name { font-size: 15px; display: block; }
  .condo-popup-addr { font-size: 12px; color: var(--muted); margin-top: 4px; }
```

- [ ] **Step 3: JS — `appState.mapMode`**

Old:
```js
const appState = { properties: [], condos: [], condosById: {}, addMode: false, editingPropertyId: null };
```
New:
```js
const appState = { properties: [], condos: [], condosById: {}, addMode: false, editingPropertyId: null, mapMode: 'unidade' };
```

- [ ] **Step 4: JS — `condoMarkersLayer` em `initMap`**

Old:
```js
let map, markersLayer;
const markersById = {};

function initMap() {
  map = L.map('map', { zoomControl: false }).setView(MAP_CENTER, MAP_ZOOM);
  L.control.zoom({ position: 'bottomright' }).addTo(map);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19
  }).addTo(map);
  const satelliteLayer = L.tileLayer.wms('https://geofloripa.pmf.sc.gov.br/geoserver/Geoportal/ows', {
    layers: '201601MUN',
    format: 'image/png',
    transparent: true,
    version: '1.3.0',
    attribution: 'Ortomosaico 2016 &copy; IPUF/PMF Florianópolis',
    maxZoom: 19
  });
  L.control.layers(null, { 'Satélite (2016)': satelliteLayer }).addTo(map);
  markersLayer = L.layerGroup().addTo(map);
}
```
New:
```js
let map, markersLayer, condoMarkersLayer;
const markersById = {};
const condoMarkersById = {};

function initMap() {
  map = L.map('map', { zoomControl: false }).setView(MAP_CENTER, MAP_ZOOM);
  L.control.zoom({ position: 'bottomright' }).addTo(map);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19
  }).addTo(map);
  const satelliteLayer = L.tileLayer.wms('https://geofloripa.pmf.sc.gov.br/geoserver/Geoportal/ows', {
    layers: '201601MUN',
    format: 'image/png',
    transparent: true,
    version: '1.3.0',
    attribution: 'Ortomosaico 2016 &copy; IPUF/PMF Florianópolis',
    maxZoom: 19
  });
  L.control.layers(null, { 'Satélite (2016)': satelliteLayer }).addTo(map);
  markersLayer = L.layerGroup().addTo(map);
  condoMarkersLayer = L.layerGroup();
}
```

- [ ] **Step 5: JS — funções de marcador de condomínio (popup mínimo) + self-check**

Old:
```js
function closeDetailView() {
```
New:
```js
function condoFillPercent(condoId, properties) {
  const linked = properties.filter(p => p.condoId === condoId);
  if (!linked.length) return null;
  const filled = linked.filter(p => p.status === 'vendido' || p.status === 'alugado').length;
  return Math.round((filled / linked.length) * 100);
}

function fillBadgeColor(pct) {
  if (pct < 34) return 'var(--warn-fg)';
  if (pct < 67) return 'var(--accent-2)';
  return 'var(--ok-fg)';
}

function condoMatchesFilters(condo, filters) {
  if (filters.neighborhood && !((condo.address && condo.address.neighborhood) || '').toLowerCase().includes(filters.neighborhood.toLowerCase())) return false;
  if (filters.amenities.length) {
    for (const amenity of filters.amenities) if (!condo[amenity]) return false;
  }
  if (filters.query) {
    const q = filters.query.toLowerCase();
    const addr = condo.address || {};
    const haystack = [condo.name, addr.street, addr.neighborhood].filter(Boolean).join(' ').toLowerCase();
    if (!haystack.includes(q)) return false;
  }
  return true;
}

function buildCondoMarkerIcon(condo, pct) {
  const photoUrl = condo.photos && condo.photos.length ? URL.createObjectURL(condo.photos[0]) : null;
  const badgeColor = fillBadgeColor(pct);
  const photoHtml = photoUrl
    ? `<img src="${photoUrl}" class="condo-pin-photo">`
    : `<div class="condo-pin-photo condo-pin-photo-empty">${ICONS.building}</div>`;
  const html = `<div class="condo-pin">
      <span class="condo-pin-badge" style="background:${badgeColor};">${pct}%</span>
      ${photoHtml}
    </div>`;
  return L.divIcon({ className: 'condo-pin-wrap', html, iconSize: [52, 64], iconAnchor: [26, 60], popupAnchor: [0, -56] });
}

function buildCondoPopupHtml(condo) {
  const addr = condo.address || {};
  const photoUrl = condo.photos && condo.photos.length ? URL.createObjectURL(condo.photos[0]) : null;
  return `
    <div class="condo-popup" data-condo-id="${condo.id}">
      ${photoUrl ? `<img src="${photoUrl}" class="condo-popup-photo">` : `<div class="condo-popup-photo condo-popup-photo-empty">${ICONS.building}</div>`}
      <div class="condo-popup-body">
        <strong class="condo-popup-name">${escapeHtml(condo.name)}</strong>
        <div class="condo-popup-addr">${escapeHtml(addr.street || '')} ${escapeHtml(addr.number || '')} — ${escapeHtml(addr.neighborhood || '')}</div>
      </div>
    </div>`;
}

function upsertCondoMarker(condo) {
  const pct = condoFillPercent(condo.id, appState.properties);
  if (pct === null || !Array.isArray(condo.coordinates)) { removeCondoMarker(condo.id); return; }
  const existing = condoMarkersById[condo.id];
  if (existing) condoMarkersLayer.removeLayer(existing);
  const marker = L.marker(condo.coordinates, { icon: buildCondoMarkerIcon(condo, pct) });
  marker.bindPopup(() => buildCondoPopupHtml(condo));
  marker.addTo(condoMarkersLayer);
  condoMarkersById[condo.id] = marker;
}

function removeCondoMarker(id) {
  const marker = condoMarkersById[id];
  if (marker) { condoMarkersLayer.removeLayer(marker); delete condoMarkersById[id]; }
}

function renderCondoMarkers() {
  appState.condos.forEach(condo => {
    if (!condoMatchesFilters(condo, lastFilters)) { removeCondoMarker(condo.id); return; }
    upsertCondoMarker(condo);
  });
}

(function selfCheckCondoMarkers() {
  const props = [
    { condoId: 'c1', status: 'vendido' }, { condoId: 'c1', status: 'disponivel' },
    { condoId: 'c2', status: 'reservado' }
  ];
  console.assert(condoFillPercent('c1', props) === 50, 'ASSERT FAIL: condoFillPercent deveria calcular 50%');
  console.assert(condoFillPercent('c2', props) === 0, 'ASSERT FAIL: condoFillPercent deveria calcular 0% sem vendido/alugado');
  console.assert(condoFillPercent('c3', props) === null, 'ASSERT FAIL: condoFillPercent deveria retornar null sem unidades vinculadas');
  console.assert(fillBadgeColor(10) === 'var(--warn-fg)', 'ASSERT FAIL: fillBadgeColor baixo deveria ser warn');
  console.assert(fillBadgeColor(50) === 'var(--accent-2)', 'ASSERT FAIL: fillBadgeColor médio deveria ser accent-2');
  console.assert(fillBadgeColor(90) === 'var(--ok-fg)', 'ASSERT FAIL: fillBadgeColor alto deveria ser ok');
  const condoNoFilter = { name: 'Ed. Teste', address: { neighborhood: 'Centro' }, pool: true };
  console.assert(condoMatchesFilters(condoNoFilter, defaultFilters()) === true, 'ASSERT FAIL: condoMatchesFilters deveria aceitar sem filtros ativos');
  console.assert(condoMatchesFilters(condoNoFilter, { ...defaultFilters(), neighborhood: 'centro' }) === true, 'ASSERT FAIL: condoMatchesFilters neighborhood deveria ser case-insensitive e parcial');
  console.assert(condoMatchesFilters(condoNoFilter, { ...defaultFilters(), neighborhood: 'norte' }) === false, 'ASSERT FAIL: condoMatchesFilters deveria rejeitar bairro diferente');
  console.assert(condoMatchesFilters(condoNoFilter, { ...defaultFilters(), amenities: ['pool'] }) === true, 'ASSERT FAIL: condoMatchesFilters deveria aceitar amenidade presente');
  console.assert(condoMatchesFilters(condoNoFilter, { ...defaultFilters(), amenities: ['gym'] }) === false, 'ASSERT FAIL: condoMatchesFilters deveria rejeitar amenidade ausente');
  console.log('[self-check] Condomínio no mapa OK');
})();

function closeDetailView() {
```

Nota: `escapeHtml` e `defaultFilters` são `function` declarations
definidas mais abaixo no mesmo `<script>` — hoisting cobre isso, não
precisam ser movidas. `lastFilters` é criado no Step 7.

- [ ] **Step 6: JS — `apply()` do Alpine passa os filtros no evento**

Old:
```js
    apply() {
      const filtered = appState.properties.filter(p => matchesFilters(p, appState.condosById, this.filters));
      window.dispatchEvent(new CustomEvent('filters-applied', { detail: { filtered } }));
    },
```
New:
```js
    apply() {
      const filtered = appState.properties.filter(p => matchesFilters(p, appState.condosById, this.filters));
      window.dispatchEvent(new CustomEvent('filters-applied', { detail: { filtered, filters: this.filters } }));
    },
```

- [ ] **Step 7: JS — `lastFilters` + listener de `filters-applied` atualiza pins de condomínio**

Old:
```js
window.addEventListener('filters-applied', (e) => {
  $('resultCount').textContent = e.detail.filtered.length;
  const filteredIds = new Set(e.detail.filtered.map(p => p.id));
  Object.entries(markersById).forEach(([id, marker]) => {
    const shouldShow = filteredIds.has(id);
    const isShown = markersLayer.hasLayer(marker);
    if (shouldShow && !isShown) marker.addTo(markersLayer);
    if (!shouldShow && isShown) markersLayer.removeLayer(marker);
  });
});
```
New:
```js
let lastFilters = defaultFilters();

window.addEventListener('filters-applied', (e) => {
  $('resultCount').textContent = e.detail.filtered.length;
  const filteredIds = new Set(e.detail.filtered.map(p => p.id));
  Object.entries(markersById).forEach(([id, marker]) => {
    const shouldShow = filteredIds.has(id);
    const isShown = markersLayer.hasLayer(marker);
    if (shouldShow && !isShown) marker.addTo(markersLayer);
    if (!shouldShow && isShown) markersLayer.removeLayer(marker);
  });
  lastFilters = e.detail.filters;
  if (appState.mapMode === 'condominio') renderCondoMarkers();
});
```

- [ ] **Step 8: JS — listeners do HUD (toggle de modo + atalho de filtro)**

Old:
```js
  $('btnToggleList').addEventListener('click', () => {
    const hidden = $('listPanel').classList.toggle('hidden');
    $('btnToggleList').textContent = hidden ? 'Mostrar lista' : 'Ocultar lista';
    map.invalidateSize();
  });

  $('tabDashboard').addEventListener('click', () => {
```
New:
```js
  $('btnToggleList').addEventListener('click', () => {
    const hidden = $('listPanel').classList.toggle('hidden');
    $('btnToggleList').textContent = hidden ? 'Mostrar lista' : 'Ocultar lista';
    map.invalidateSize();
  });

  $('mapModeToggle').addEventListener('click', (e) => {
    const btn = e.target.closest('button[data-mode]');
    if (!btn || appState.mapMode === btn.dataset.mode) return;
    appState.mapMode = btn.dataset.mode;
    $('mapModeToggle').querySelectorAll('button').forEach(b => b.classList.toggle('active', b === btn));
    if (appState.mapMode === 'condominio') {
      markersLayer.remove();
      renderCondoMarkers();
      condoMarkersLayer.addTo(map);
    } else {
      condoMarkersLayer.remove();
      markersLayer.addTo(map);
    }
  });

  $('btnHudFilter').addEventListener('click', () => $('btnCollapseSidebar').click());

  $('tabDashboard').addEventListener('click', () => {
```

- [ ] **Step 9: Teste manual — toggle troca camada, pin aparece com foto e %**

Via Browser pane: usando o condomínio criado na Task 1 (com endereço e
foto), vincular 2 imóveis a ele (`f_condoId`) com status diferentes (1
`vendido`, 1 `disponivel`) e salvar. Clicar "Condomínio" no HUD:
confirmar que os pins de unidade somem do mapa
(`map.hasLayer(markersLayer) === false`) e que aparece 1 pin de
condomínio com a foto e badge "50%" (cor `var(--accent-2)`, já que 50
cai na faixa média). Clicar o pin: popup mostra nome, endereço e foto.
Clicar "Unidade" de novo: confirma que volta ao comportamento normal
(`map.hasLayer(markersLayer) === true`, `map.hasLayer(condoMarkersLayer)
=== false`). Digitar um bairro que não bate no filtro da sidebar e
confirmar que o pin do condomínio some do modo Condomínio; limpar o
filtro e confirmar que volta.

- [ ] **Step 10: Rodar self-check no console**

`read_console_messages()` — `[self-check] Condomínio no mapa OK`, e os
self-checks anteriores, sem `ASSERT FAIL`.

- [ ] **Step 11: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: HUD de modo Unidade/Condomínio e pins de condomínio no mapa"
```

---

### Task 3: Cartão de prédio completo (galeria, descrição, amenidades, unidades agenciadas, Agenciar)

**Files:**
- Modify: `mapa-imoveis/index.html` (CSS; JS: `buildCondoPopupHtml`
  reescrita, novas `condoGalleryStep`, `registerAgencyRequest`)

**Interfaces:**
- Consumes: `buildCondoPopupHtml` (Task 2) — reescrita completa.
  `AMENITY_LABELS`, `AMENITY_CHECK_ICON`, `priceLabelFor`,
  `UNIT_TYPE_LABELS`, `openPropertyDetail`, `escapeHtml`, `dbPut` (já
  existentes).
- Produces: `condoGalleryStep(btn, delta)`, `registerAgencyRequest(condoId)`.

- [ ] **Step 1: CSS — galeria, seções, lista de unidades, botão Agenciar**

Old:
```css
  .condo-popup-addr { font-size: 12px; color: var(--muted); margin-top: 4px; }
```
New:
```css
  .condo-popup-addr { font-size: 12px; color: var(--muted); margin-top: 4px; }
  .condo-popup-gallery { position: relative; }
  .condo-popup-gallery-prev, .condo-popup-gallery-next { position: absolute; top: 50%; transform: translateY(-50%); width: 24px; height: 24px; border-radius: 50%; background: rgba(20,24,50,.5); color: #fff; border: 0; cursor: pointer; font-size: 14px; line-height: 1; }
  .condo-popup-gallery-prev { left: 8px; }
  .condo-popup-gallery-next { right: 8px; }
  .condo-popup-section { margin-top: 10px; }
  .condo-popup-section h4 { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: .4px; color: var(--muted); margin: 0 0 6px; }
  .condo-popup-section p { font-size: 12px; color: var(--text); margin: 0; }
  .condo-popup-amenities { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
  .condo-popup-amenities span { display: flex; align-items: center; gap: 6px; font-size: 11px; color: var(--text); fill: none; stroke: var(--accent); }
  .condo-popup-units { display: flex; flex-direction: column; gap: 6px; }
  .condo-popup-unit { background: var(--panel-2); border: 1px solid var(--line); border-radius: 8px; padding: 6px 10px; cursor: pointer; display: flex; justify-content: space-between; align-items: center; font-size: 12px; }
  .condo-popup-unit:hover { border-color: var(--accent); }
  .condo-popup-unit span { color: var(--muted); }
  .condo-popup-empty { font-size: 12px; color: var(--muted); margin: 0; }
  .condo-popup-agenciar { width: 100%; margin-top: 12px; background: var(--accent); color: #fff; border: 0; padding: 9px; border-radius: 8px; font-weight: 700; cursor: pointer; font-size: 13px; }
```

- [ ] **Step 2: JS — `buildCondoPopupHtml` completa + `condoGalleryStep` + `registerAgencyRequest`**

Old:
```js
function buildCondoPopupHtml(condo) {
  const addr = condo.address || {};
  const photoUrl = condo.photos && condo.photos.length ? URL.createObjectURL(condo.photos[0]) : null;
  return `
    <div class="condo-popup" data-condo-id="${condo.id}">
      ${photoUrl ? `<img src="${photoUrl}" class="condo-popup-photo">` : `<div class="condo-popup-photo condo-popup-photo-empty">${ICONS.building}</div>`}
      <div class="condo-popup-body">
        <strong class="condo-popup-name">${escapeHtml(condo.name)}</strong>
        <div class="condo-popup-addr">${escapeHtml(addr.street || '')} ${escapeHtml(addr.number || '')} — ${escapeHtml(addr.neighborhood || '')}</div>
      </div>
    </div>`;
}

function upsertCondoMarker(condo) {
```
New:
```js
function buildCondoPopupHtml(condo) {
  const addr = condo.address || {};
  const photoUrls = (condo.photos || []).map(b => URL.createObjectURL(b));
  const activeAmenities = Object.keys(AMENITY_LABELS).filter(k => condo[k]);
  const units = appState.properties.filter(p => p.condoId === condo.id);
  const galleryHtml = photoUrls.length
    ? `<div class="condo-popup-gallery" data-idx="0">
        <img src="${photoUrls[0]}" class="condo-popup-photo">
        ${photoUrls.length > 1 ? `
          <button type="button" class="condo-popup-gallery-prev" onclick="condoGalleryStep(this, -1)">‹</button>
          <button type="button" class="condo-popup-gallery-next" onclick="condoGalleryStep(this, 1)">›</button>` : ''}
      </div>`
    : `<div class="condo-popup-photo condo-popup-photo-empty">${ICONS.building}</div>`;
  const amenitiesHtml = activeAmenities.length
    ? `<div class="condo-popup-section"><h4>Condomínio</h4><div class="condo-popup-amenities">${activeAmenities.map(k => `<span>${AMENITY_CHECK_ICON}${AMENITY_LABELS[k]}</span>`).join('')}</div></div>`
    : '';
  const unitsHtml = `<div class="condo-popup-section"><h4>Unidades agenciadas</h4>${
    units.length
      ? `<div class="condo-popup-units">${units.map(u => `
          <div class="condo-popup-unit" onclick="openPropertyDetail('${u.id}')">
            <strong>${priceLabelFor(u)}</strong>
            <span>${UNIT_TYPE_LABELS[u.unitType] || u.unitType}${u.rooms ? ' · ' + u.rooms + 'q' : ''}</span>
          </div>`).join('')}</div>`
      : `<p class="condo-popup-empty">Nenhuma unidade agenciada neste prédio.</p>`
  }</div>`;
  return `
    <div class="condo-popup" data-condo-id="${condo.id}">
      ${galleryHtml}
      <div class="condo-popup-body">
        <strong class="condo-popup-name">${escapeHtml(condo.name)}</strong>
        <div class="condo-popup-addr">${escapeHtml(addr.street || '')} ${escapeHtml(addr.number || '')} — ${escapeHtml(addr.neighborhood || '')}</div>
        ${condo.description ? `<div class="condo-popup-section"><h4>Sobre o prédio</h4><p>${escapeHtml(condo.description)}</p></div>` : ''}
        ${amenitiesHtml}
        ${unitsHtml}
        <button type="button" class="condo-popup-agenciar" onclick="registerAgencyRequest('${condo.id}')">Agenciar</button>
      </div>
    </div>`;
}

function condoGalleryStep(btn, delta) {
  const wrap = btn.closest('.condo-popup-gallery');
  const popup = btn.closest('.condo-popup');
  const condo = appState.condosById[popup.dataset.condoId];
  const photoUrls = (condo.photos || []).map(b => URL.createObjectURL(b));
  let idx = Number(wrap.dataset.idx);
  idx = (idx + delta + photoUrls.length) % photoUrls.length;
  wrap.dataset.idx = idx;
  wrap.querySelector('img').src = photoUrls[idx];
}

async function registerAgencyRequest(condoId) {
  const condo = appState.condosById[condoId];
  if (!condo) return;
  condo.agencyRequestedAt = Date.now();
  try {
    await dbPut('condos', condo);
    alert('Agenciamento registrado para ' + condo.name + '.');
  } catch (err) {
    console.error(err);
    alert('Não foi possível registrar o agenciamento.');
  }
}

function upsertCondoMarker(condo) {
```

- [ ] **Step 3: Teste manual — popup completo**

Via Browser pane, no condomínio de teste (2 unidades vinculadas, 1 foto,
descrição preenchida, com `pool`/`elevator` marcados): clicar o pin no
modo Condomínio, confirmar que o popup mostra "Sobre o prédio" com a
descrição, seção "Condomínio" só com Piscina e Elevador (não mostra as
amenidades desmarcadas), seção "Unidades agenciadas" com as 2 unidades.
Clicar numa unidade da lista: confirma que abre `#detailView` com os
dados certos daquela unidade (`$('detailView').dataset.id` bate).
Voltar, reabrir o popup, clicar "Agenciar": confirma `alert` e, via
`dbGet('condos', id)`, que `agencyRequestedAt` foi gravado. Adicionar
uma 2ª foto ao condomínio (editar não existe — criar um condomínio novo
de teste com 2 fotos) e confirmar que as setas `‹`/`›` trocam a imagem
exibida ciclicamente. Testar um condomínio sem nenhuma unidade
vinculada: popup mostra "Nenhuma unidade agenciada neste prédio." (mas
lembrando que esse condomínio não teria pin no mapa — testar chamando
`buildCondoPopupHtml` direto via `javascript_tool` para essa checagem
pontual). Payload `<img src=x onerror="window.__xss=true">` no nome e na
descrição do condomínio: confirma que `window.__xss` não vira `true` e
que o popup mostra o texto literal.

- [ ] **Step 4: Rodar self-check no console**

`read_console_messages()` — todos os self-checks anteriores, sem
`ASSERT FAIL`.

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: cartão de prédio completo — galeria, descrição, amenidades, unidades agenciadas e agenciar"
```

---

## Self-Review

**Cobertura do spec:**
- Modelo de dados do condomínio (endereço/coordenadas/fotos/descrição) →
  Task 1. ✅
- HUD flutuante (toggle Unidade/Condomínio + botão Filtrar) → Task 2. ✅
- Pin de condomínio (foto + badge de %, faixas de cor documentadas) →
  Task 2. ✅
- Filtro de bairro/amenidades afetando condomínios no modo Condomínio →
  Task 2. ✅
- Cartão de prédio completo (galeria, sobre o prédio, condomínio,
  unidades agenciadas linkando pra `#detailView`, Agenciar local) →
  Task 3. ✅
- Segurança (`escapeHtml` em nome/descrição/endereço do condomínio) →
  Task 2 (popup mínimo) e Task 3 (popup completo). ✅
- Fora de escopo (POI real, webhook ClickUp, popup de unidade dedicado,
  edição de condomínio, "Últimas vendas") → nenhuma task toca nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo.

**Consistência de nomes:** `buildCondoPopupHtml` é reescrita 1 vez (Task
3 Old é o New exato da Task 2). `geocodeAddress()` perde o parâmetro
implícito e vira `geocodeAddress(prefix)` numa única mudança (Task 1),
todos os call sites atualizados no mesmo Step. `renderPhotoThumbnails()`
generalizada uma única vez (Task 1 Step 3), os 2 call sites que a usavam
sem argumento (`openPropertyForm`, listener de `f_photos`) atualizados
para os wrappers novos no mesmo Step/Step seguinte — conferido que não
sobra nenhuma chamada à assinatura antiga. `lastFilters` declarado uma
vez (Task 2 Step 7), lido em `renderCondoMarkers` (Task 2 Step 5) sem
redeclarar — a função é definida ANTES da declaração de `lastFilters`
no arquivo, mas como é `function` hoisted e só é chamada em tempo de
evento (depois do parse completo), não há problema de ordem.

# Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Camada de pontos de amenidades reais (mercado, padaria,
escola, academia, hospital, farmácia, restaurante, parque) no mapa,
buscados via Overpass API (OpenStreetMap), com legenda flutuante que
liga/desliga cada categoria.

**Architecture:** Só `mapa-imoveis/index.html`. Task 1 entrega a busca
Overpass + renderização dos pontos (zoom-gated, com cache por área) —
testável via console/zoom, sem UI de controle ainda (tudo visível por
padrão). Task 2 entrega a legenda flutuante que liga a UI de toggle às
camadas que a Task 1 já criou.

**Tech Stack:** Mesmo stack, sem dependência nova. `fetch` nativo pro
Overpass, `L.circleMarker`/`L.layerGroup` nativos do Leaflet (já
carregado). Verificação via Browser pane + `console.assert`.

## Global Constraints

- Arquivo único: `mapa-imoveis/index.html`. Sem arquivo novo, sem
  dependência nova.
- Endpoint Overpass: `https://overpass-api.de/api/interpreter` (o
  oficial). Task 1 originalmente usou o espelho
  `overpass.kumi.systems` — testado antes do plano, parecia funcionar —
  mas na verificação ao vivo da implementação ele se mostrou instável
  (fetch travando). Trocado pro oficial em commit de fix após a Task 1,
  confirmado com 123 pontos reais em Florianópolis.
- Zoom mínimo pra buscar/mostrar POIs: 14 (constante `POI_MIN_ZOOM`).
- `AMENITY_POI_CATEGORIES` é a única fonte de verdade das 8 categorias
  — query, classificação e legenda leem dessa mesma lista, sem duplicar.
- Nome de POI (`tags.name`, dado externo do OSM) que vira tooltip passa
  por `escapeHtml()` (já existe no arquivo).
- Falha de rede do Overpass: `console.error` e não mexe nos pontos já
  renderizados — sem banner, sem retry automático (decisão do spec).
- IDs de elementos existentes não podem mudar.

---

### Task 1: Busca Overpass + pontos de amenidade no mapa (zoom-gated, com cache)

**Files:**
- Modify: `mapa-imoveis/index.html` (JS: novo `AMENITY_POI_CATEGORIES`
  na seção CONFIG; novas funções na seção MAP; listener de
  `moveend`/`zoomend` no bloco de `DOMContentLoaded` que já chama
  `initMap()`)

**Interfaces:**
- Produces: `AMENITY_POI_CATEGORIES` (array), `buildPoiQuery(bbox)`,
  `categoryForTags(tags)`, `initPoiLayers()`, `fetchAndRenderPois()`,
  `clearPoiLayers()`, `renderPoiElements(elements)`,
  `poiLayersByCategory` (objeto `{chave: L.layerGroup}`), `poiEnabled`
  (objeto `{chave: boolean}`, todas `true` por padrão — consumido pela
  Task 2).
- Consumes: `map` (já existe, de `initMap`), `escapeHtml` (já existe).

- [ ] **Step 1: JS — `AMENITY_POI_CATEGORIES` na seção CONFIG**

Old:
```js
const AMENITY_CHECK_ICON = '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6L9 17l-5-5"/></svg>';
const AMENITY_LABELS = {
  pool: 'Piscina', gym: 'Academia', partyRoom: 'Salão de festas', playground: 'Playground',
  petArea: 'Área pet', security24h: 'Portaria 24h', elevator: 'Elevador', gatedCommunity: 'Condomínio fechado'
};
```
New:
```js
const AMENITY_CHECK_ICON = '<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6L9 17l-5-5"/></svg>';
const AMENITY_LABELS = {
  pool: 'Piscina', gym: 'Academia', partyRoom: 'Salão de festas', playground: 'Playground',
  petArea: 'Área pet', security24h: 'Portaria 24h', elevator: 'Elevador', gatedCommunity: 'Condomínio fechado'
};

/* categorias de POI (amenidades reais no mapa, via Overpass/OSM) — fase 8 */
const AMENITY_POI_CATEGORIES = [
  { key: 'mercados', label: 'Mercados', color: '#4ade80', osmKey: 'shop', osmValue: 'supermarket' },
  { key: 'padarias', label: 'Padarias', color: '#facc15', osmKey: 'shop', osmValue: 'bakery' },
  { key: 'escolas', label: 'Escolas', color: '#4b5fee', osmKey: 'amenity', osmValue: 'school' },
  { key: 'academias', label: 'Academias', color: '#f97316', osmKey: 'leisure', osmValue: 'fitness_centre' },
  { key: 'hospitais', label: 'Hospitais', color: '#ef4444', osmKey: 'amenity', osmValue: 'hospital' },
  { key: 'farmacias', label: 'Farmácias', color: '#22c55e', osmKey: 'amenity', osmValue: 'pharmacy' },
  { key: 'restaurantes', label: 'Restaurantes', color: '#a855f7', osmKey: 'amenity', osmValue: 'restaurant' },
  { key: 'parques', label: 'Parques', color: '#14b8a6', osmKey: 'leisure', osmValue: 'park' }
];
const POI_ENDPOINT = 'https://overpass.kumi.systems/api/interpreter';
const POI_MIN_ZOOM = 14;
```

- [ ] **Step 2: JS — funções de query/classificação/busca/render, logo após `initMap`**

Old:
```js
  L.control.layers(null, { 'Satélite (2016)': satelliteLayer }).addTo(map);
  markersLayer = L.layerGroup().addTo(map);
  condoMarkersLayer = L.layerGroup();
}

window.addEventListener('DOMContentLoaded', () => {
  initMap();
  // ponytail: Leaflet mede o container no momento do L.map(); se o layout
  // flex/vh ainda não assentou nesse instante, o mapa fica com escala/
  // proporção erradas até um resize manual. invalidateSize corrige.
  setTimeout(() => map.invalidateSize(), 50);
  window.addEventListener('resize', () => map.invalidateSize());
});
```
New:
```js
  L.control.layers(null, { 'Satélite (2016)': satelliteLayer }).addTo(map);
  markersLayer = L.layerGroup().addTo(map);
  condoMarkersLayer = L.layerGroup();
}

const poiLayersByCategory = {};
const poiEnabled = Object.fromEntries(AMENITY_POI_CATEGORIES.map(c => [c.key, true]));
let lastPoiBounds = null;
let poiFetchTimer = null;

function initPoiLayers() {
  AMENITY_POI_CATEGORIES.forEach(c => { poiLayersByCategory[c.key] = L.layerGroup().addTo(map); });
}

function buildPoiQuery(bbox) {
  const clauses = AMENITY_POI_CATEGORIES.map(c =>
    `node["${c.osmKey}"="${c.osmValue}"](${bbox});way["${c.osmKey}"="${c.osmValue}"](${bbox});`
  ).join('');
  return `[out:json][timeout:25];(${clauses});out center tags;`;
}

function categoryForTags(tags) {
  return AMENITY_POI_CATEGORIES.find(c => tags && tags[c.osmKey] === c.osmValue) || null;
}

function clearPoiLayers() {
  AMENITY_POI_CATEGORIES.forEach(c => poiLayersByCategory[c.key].clearLayers());
}

function renderPoiElements(elements) {
  clearPoiLayers();
  elements.forEach(el => {
    const cat = categoryForTags(el.tags);
    if (!cat) return;
    const lat = el.lat ?? (el.center && el.center.lat);
    const lon = el.lon ?? (el.center && el.center.lon);
    if (lat == null || lon == null) return;
    const marker = L.circleMarker([lat, lon], { radius: 6, color: '#fff', weight: 2, fillColor: cat.color, fillOpacity: 1 });
    if (el.tags.name) marker.bindTooltip(escapeHtml(el.tags.name));
    marker.addTo(poiLayersByCategory[cat.key]);
  });
}

async function fetchAndRenderPois() {
  if (map.getZoom() < POI_MIN_ZOOM) { clearPoiLayers(); return; }
  const bounds = map.getBounds();
  if (lastPoiBounds && lastPoiBounds.contains(bounds)) return;
  const padded = bounds.pad(0.3);
  lastPoiBounds = padded;
  const bbox = `${padded.getSouth()},${padded.getWest()},${padded.getNorth()},${padded.getEast()}`;
  try {
    const res = await fetch(POI_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: 'data=' + encodeURIComponent(buildPoiQuery(bbox))
    });
    if (!res.ok) throw new Error('Overpass HTTP ' + res.status);
    const data = await res.json();
    renderPoiElements(data.elements || []);
  } catch (err) {
    console.error('[POI] falha ao buscar amenidades', err);
  }
}

(function selfCheckPoi() {
  console.assert(AMENITY_POI_CATEGORIES.length === 8, 'ASSERT FAIL: deveria ter 8 categorias de amenidade');
  console.assert(new Set(AMENITY_POI_CATEGORIES.map(c => c.key)).size === 8, 'ASSERT FAIL: chaves de categoria deveriam ser únicas');
  const q = buildPoiQuery('1,2,3,4');
  console.assert(q.includes('node["shop"="supermarket"](1,2,3,4)'), 'ASSERT FAIL: buildPoiQuery deveria montar cláusula de mercado');
  console.assert(q.includes('way["leisure"="park"](1,2,3,4)'), 'ASSERT FAIL: buildPoiQuery deveria incluir way de parque');
  console.assert(categoryForTags({ amenity: 'pharmacy' }).key === 'farmacias', 'ASSERT FAIL: categoryForTags deveria achar farmácia');
  console.assert(categoryForTags({ shop: 'bakery' }).key === 'padarias', 'ASSERT FAIL: categoryForTags deveria achar padaria');
  console.assert(categoryForTags({ amenity: 'bogus' }) === null, 'ASSERT FAIL: categoryForTags deveria retornar null pra tag desconhecida');
  console.assert(categoryForTags(null) === null, 'ASSERT FAIL: categoryForTags deveria aceitar tags undefined sem lançar erro');
  console.log('[self-check] Amenidades (POI) OK');
})();

window.addEventListener('DOMContentLoaded', () => {
  initMap();
  initPoiLayers();
  map.on('moveend zoomend', () => {
    clearTimeout(poiFetchTimer);
    poiFetchTimer = setTimeout(fetchAndRenderPois, 600);
  });
  // ponytail: Leaflet mede o container no momento do L.map(); se o layout
  // flex/vh ainda não assentou nesse instante, o mapa fica com escala/
  // proporção erradas até um resize manual. invalidateSize corrige.
  setTimeout(() => map.invalidateSize(), 50);
  window.addEventListener('resize', () => map.invalidateSize());
});
```

- [ ] **Step 3: Teste manual — zoom-gate, busca real, cache, erro de rede**

Via Browser pane (`preview_start({name: "mapa-imoveis"})`): confirmar
que no zoom inicial (13) nenhum ponto de amenidade aparece
(`Object.values(poiLayersByCategory).every(l => l.getLayers().length
=== 0)`). Dar zoom pra 15+ numa área real (ex: `map.setView([-27.595,
-48.548], 16)`) e aguardar o debounce (~700ms): confirmar via
`read_network_requests`/`javascript_tool` que uma requisição POST pro
Overpass foi feita e que `poiLayersByCategory` tem pelo menos 1
categoria com pontos (dado real de Florianópolis deve ter farmácia,
mercado etc. na área central). Conferir que cada marcador tem a cor
certa pra sua categoria (`marker.options.fillColor === cat.color`).
Mover o mapa um pouco (`map.panBy([5, 5])`) dentro da mesma área:
confirmar que NÃO dispara nova requisição (bounds contidos no cache).
Mover bem longe (`map.setView([-23.55, -46.63], 16)` — São Paulo):
confirmar que dispara nova busca depois do debounce. Simular falha
trocando `POI_ENDPOINT` temporariamente pra uma URL inválida via
`javascript_tool`, forçar `fetchAndRenderPois()`, confirmar que loga
erro no console e não lança exceção não tratada.

- [ ] **Step 4: Rodar self-check no console**

`read_console_messages()` — `[self-check] Amenidades (POI) OK` e os
self-checks anteriores, sem `ASSERT FAIL`.

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: busca de amenidades reais (Overpass/OSM) e pontos no mapa, com zoom mínimo e cache por área"
```

---

### Task 2: Legenda flutuante com toggle por categoria

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML de `#mapWrap`; CSS; JS:
  `renderAmenityLegend`, `togglePoiCategory`, listener de clique)

**Interfaces:**
- Consumes: `AMENITY_POI_CATEGORIES`, `poiEnabled`,
  `poiLayersByCategory` (Task 1).
- Produces: `renderAmenityLegend()`, `togglePoiCategory(key)`.

- [ ] **Step 1: HTML — container da legenda dentro de `#mapWrap`**

Old:
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
    <div id="amenityLegend"></div>
    </div>
```

- [ ] **Step 2: CSS — legenda flutuante no canto inferior esquerdo**

Old:
```css
  #mapHud #btnHudFilter { background: var(--accent); color: #fff; border: 0; padding: 7px 16px; border-radius: 20px; font-weight: 700; font-size: 12px; cursor: pointer; }
```
New:
```css
  #mapHud #btnHudFilter { background: var(--accent); color: #fff; border: 0; padding: 7px 16px; border-radius: 20px; font-weight: 700; font-size: 12px; cursor: pointer; }
  #amenityLegend { position: absolute; left: 14px; bottom: 14px; z-index: 1000; display: flex; flex-direction: column; gap: 2px; background: var(--panel); border-radius: 12px; padding: 8px; box-shadow: var(--shadow-soft); max-height: 220px; overflow-y: auto; }
  .amenity-legend-item { display: flex; align-items: center; gap: 6px; background: transparent; border: 0; color: var(--text); font-size: 11px; padding: 4px 6px; border-radius: 6px; cursor: pointer; text-align: left; }
  .amenity-legend-item:hover { background: var(--panel-2); }
  .amenity-legend-item.off { color: var(--muted); opacity: .5; }
  .amenity-legend-dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }
```

- [ ] **Step 3: JS — `renderAmenityLegend` + `togglePoiCategory`, logo após `fetchAndRenderPois`**

Old (arquivo real, pós-fixes da Task 1 — endpoint oficial e
`lastPoiBounds` gravado só após sucesso, não mais `overpass.kumi.systems`
nem antes do fetch como no primeiro rascunho deste plano):
```js
async function fetchAndRenderPois() {
  if (map.getZoom() < POI_MIN_ZOOM) { clearPoiLayers(); return; }
  const bounds = map.getBounds();
  if (lastPoiBounds && lastPoiBounds.contains(bounds)) return;
  const padded = bounds.pad(0.3);
  const bbox = `${padded.getSouth()},${padded.getWest()},${padded.getNorth()},${padded.getEast()}`;
  try {
    const res = await fetch(POI_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: 'data=' + encodeURIComponent(buildPoiQuery(bbox))
    });
    if (!res.ok) throw new Error('Overpass HTTP ' + res.status);
    const data = await res.json();
    renderPoiElements(data.elements || []);
    lastPoiBounds = padded;
  } catch (err) {
    console.error('[POI] falha ao buscar amenidades', err);
  }
}
```
New:
```js
async function fetchAndRenderPois() {
  if (map.getZoom() < POI_MIN_ZOOM) { clearPoiLayers(); return; }
  const bounds = map.getBounds();
  if (lastPoiBounds && lastPoiBounds.contains(bounds)) return;
  const padded = bounds.pad(0.3);
  const bbox = `${padded.getSouth()},${padded.getWest()},${padded.getNorth()},${padded.getEast()}`;
  try {
    const res = await fetch(POI_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: 'data=' + encodeURIComponent(buildPoiQuery(bbox))
    });
    if (!res.ok) throw new Error('Overpass HTTP ' + res.status);
    const data = await res.json();
    renderPoiElements(data.elements || []);
    lastPoiBounds = padded;
  } catch (err) {
    console.error('[POI] falha ao buscar amenidades', err);
  }
}

function renderAmenityLegend() {
  $('amenityLegend').innerHTML = AMENITY_POI_CATEGORIES.map(c => `
    <button type="button" class="amenity-legend-item${poiEnabled[c.key] ? '' : ' off'}" data-key="${c.key}">
      <span class="amenity-legend-dot" style="background:${c.color};"></span>${c.label}
    </button>`).join('');
}

function togglePoiCategory(key) {
  poiEnabled[key] = !poiEnabled[key];
  const layer = poiLayersByCategory[key];
  if (poiEnabled[key]) layer.addTo(map); else layer.remove();
}
```

- [ ] **Step 4: JS — render inicial da legenda + listener de clique, no `DOMContentLoaded` que já chama `initPoiLayers`**

Old:
```js
window.addEventListener('DOMContentLoaded', () => {
  initMap();
  initPoiLayers();
  map.on('moveend zoomend', () => {
    clearTimeout(poiFetchTimer);
    poiFetchTimer = setTimeout(fetchAndRenderPois, 600);
  });
```
New:
```js
window.addEventListener('DOMContentLoaded', () => {
  initMap();
  initPoiLayers();
  renderAmenityLegend();
  $('amenityLegend').addEventListener('click', (e) => {
    const btn = e.target.closest('button[data-key]');
    if (!btn) return;
    togglePoiCategory(btn.dataset.key);
    btn.classList.toggle('off', !poiEnabled[btn.dataset.key]);
  });
  map.on('moveend zoomend', () => {
    clearTimeout(poiFetchTimer);
    poiFetchTimer = setTimeout(fetchAndRenderPois, 600);
  });
```

- [ ] **Step 5: Teste manual — legenda, toggle, sem nova busca ao reativar**

Via Browser pane: confirmar que `#amenityLegend` renderiza 8 itens
(`$('amenityLegend').children.length === 8`), todos sem a classe `off`
no início. Dar zoom até aparecerem pontos reais (igual Task 1). Clicar
numa categoria com pontos visíveis (ex: Farmácias): confirmar que
`map.hasLayer(poiLayersByCategory.farmacias) === false` e que o botão
ganhou a classe `off`; os pontos das outras categorias continuam no
mapa. Clicar de novo: confirmar que volta (`hasLayer === true`, sem
classe `off`) SEM disparar nova requisição de rede (checar
`read_network_requests` antes/depois do 2º clique — contagem de
chamada ao Overpass não muda). Trocar pro modo Condomínio (HUD da fase
6) e confirmar que os pontos de amenidade continuam visíveis,
independente do toggle Unidade/Condomínio.

- [ ] **Step 6: Rodar self-check no console**

`read_console_messages()` — todos os self-checks anteriores, sem
`ASSERT FAIL`.

- [ ] **Step 7: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: legenda de amenidades com toggle por categoria"
```

---

## Self-Review

**Cobertura do spec:**
- Fonte Overpass/OSM, endpoint testado → Task 1. ✅
- 8 categorias, tags certas, parques via node+way com `out center` →
  Task 1 Step 1-2. ✅
- Zoom mínimo 14, cache por área, debounce → Task 1. ✅
- Cores próprias (não usa tokens de UI) → Task 1 Step 1. ✅
- `L.circleMarker` (nativo, leve), tooltip com nome escapado → Task 1
  Step 2. ✅
- Falha de rede não quebra nem limpa pontos existentes → Task 1 Step 2
  (`catch` só loga). ✅
- Legenda flutuante, 8 itens, toggle sem refetch, ligada por padrão →
  Task 2. ✅
- Funciona nos 2 modos do mapa (Unidade/Condomínio) → Task 2 Step 5
  (teste explícito). ✅
- Fora de escopo (clique/popup no ponto, config de zoom pela UI, cache
  entre sessões, failover de provedor) → nenhuma task toca nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo.

**Consistência de nomes:** `AMENITY_POI_CATEGORIES` declarada uma vez
(Task 1 Step 1), lida por `buildPoiQuery`/`categoryForTags`/
`initPoiLayers`/`renderAmenityLegend` sem duplicar a lista de
categorias em nenhum lugar. `poiEnabled`/`poiLayersByCategory`
declaradas uma vez (Task 1 Step 2), a Task 2 só lê e muta
(`togglePoiCategory`), não redeclara. `fetchAndRenderPois` é reescrita
uma vez no Step 3 da Task 2 (Old é o New exato da Task 1), só pra
inserir as 2 funções novas logo depois — corpo da função em si não
muda entre as tasks.

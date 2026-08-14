# Mapa de Imóveis (uso interno) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ferramenta interna de um único arquivo HTML onde corretores cadastram imóveis manualmente via pin num mapa (Leaflet + OpenStreetMap, sem custo), com filtros por características do imóvel e do condomínio como núcleo da UX, persistência 100% local em IndexedDB, e backup via exportar/importar JSON.

**Architecture:** Arquivo único `mapa-imoveis/index.html`, sem build step. CSS com custom properties (dark theme, mesmo padrão do `corretor-gv/calculadora-proposta.html`). JS vanilla para mapa, formulário, IndexedDB e fotos; Alpine.js (via CDN) isolado só no painel de filtros, que é a parte com estado mais cruzado (filtro → contagem → mapa → lista simultaneamente).

**Tech Stack:** HTML/CSS/JS vanilla, Leaflet 1.x (CDN, tiles OpenStreetMap), Alpine.js 3.x (CDN, escopo restrito ao painel de filtros), IndexedDB nativo (sem wrapper).

## Global Constraints

- Sem backend, sem login, sem sincronização entre máquinas — uso local individual (spec: "Fora do escopo").
- Sem provedor de mapa pago — só Leaflet + tiles OpenStreetMap, sem chave de API.
- Entrega final em `mapa-imoveis/index.html`, nenhuma outra alteração em arquivos existentes do repositório.
- Coordenadas sempre armazenadas como `[lat, lng]` (ordem do Leaflet) — não confundir com a ordem `[lng, lat]` usada por APIs GeoJSON/Mappo.
- Sem framework de teste — verificação via `console.assert` embutido no próprio arquivo (self-check no load) para lógica pura, e verificação manual no navegador para UI/DOM/IndexedDB.
- MVP: sem clustering de pins, sem multi-usuário, sem autenticação.

---

## File Structure

Um único arquivo, `mapa-imoveis/index.html`, criado na Task 1 com seções âncora (`/* === SECTION: X === */`) dentro de um único bloco `<script>`. Tasks seguintes inserem código dentro dessas seções ou modificam trechos específicos já escritos — cada task referencia a âncora ou linha exata onde inserir/alterar.

Seções do `<script>`, na ordem em que aparecem no arquivo:
1. `CONFIG` — constantes (cores de status, tipos de imóvel, centro do mapa)
2. `INDEXEDDB` — camada de persistência (dbOpen/dbPut/dbGet/dbGetAll/dbDelete)
3. `LOGIC` — funções puras (formatCurrency, validateProperty, matchesFilters)
4. `STATE` — `appState` (cache em memória de properties/condos)
5. `MAP` — inicialização do Leaflet, marcadores (upsert/remove/popup)
6. `FORM` — abrir/fechar/ler/salvar formulário de cadastro + condomínio inline
7. `PHOTOS` — captura, redimensionamento e miniaturas de fotos
8. `FILTERS` — componente Alpine `filterApp()` + sincronização com mapa
9. `LIST` — lista lateral sincronizada com o filtro
10. `BACKUP` — exportar/importar `.json`

---

### Task 1: Estrutura base — HTML, CSS e mapa Leaflet vazio

**Files:**
- Create: `mapa-imoveis/index.html`

**Interfaces:**
- Produces: elemento `#map` no DOM; variáveis globais `map` (instância Leaflet) e `markersLayer` (L.LayerGroup), preenchidas por `initMap()`; constantes `MAP_CENTER`, `MAP_ZOOM`, `STATUS_COLORS`, `STATUS_LABELS`, `UNIT_TYPES`, `UNIT_TYPE_LABELS`, `STATUS_OPTIONS`; seções âncora comentadas para as tasks seguintes.

- [ ] **Step 1: Criar o arquivo com HTML/CSS/JS skeleton completo**

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Mapa de Imóveis — Uso Interno</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1/dist/leaflet.css">
<style>
  :root {
    --bg: #0f1720; --panel: #17212b; --panel-2: #1e2c38; --line: #2a3a49;
    --text: #e6edf3; --muted: #8b9bab; --accent: #3ea6ff;
    --ok-bg: #10361f; --ok-fg: #4ade80; --ok-line: #1f6b3a;
    --warn-bg: #3a1717; --warn-fg: #ff6b6b; --warn-line: #7a2a2a;
    --radius: 10px;
    --mono: ui-monospace, "SF Mono", "Cascadia Code", Consolas, monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  }
  * { box-sizing: border-box; }
  html, body { height: 100%; margin: 0; }
  body { font-family: var(--sans); background: var(--bg); color: var(--text); font-size: 15px; line-height: 1.4; display: flex; flex-direction: column; }
  header {
    display: flex; align-items: center; gap: 16px; flex-wrap: wrap;
    padding: 12px 20px; background: var(--panel); border-bottom: 1px solid var(--line);
    position: sticky; top: 0; z-index: 30;
  }
  header h1 { font-size: 16px; margin: 0; font-weight: 600; flex: 1; min-width: 160px; }
  header button {
    background: var(--accent); color: #04121f; border: 0; padding: 8px 14px;
    border-radius: 8px; font-weight: 600; cursor: pointer; font-size: 13px;
  }
  header button.ghost { background: transparent; color: var(--muted); border: 1px solid var(--line); }
  header button.active { background: var(--ok-fg); }
  .app { flex: 1; display: flex; min-height: 0; }
  #sidebar {
    width: 320px; flex-shrink: 0; background: var(--panel); border-right: 1px solid var(--line);
    display: flex; flex-direction: column; overflow: hidden;
  }
  #filterPanel { padding: 14px; border-bottom: 1px solid var(--line); overflow-y: auto; max-height: 55%; }
  #filterPanel h2, #listPanel h2 { font-size: 12px; text-transform: uppercase; letter-spacing: .5px; color: var(--muted); margin: 0 0 10px; }
  #listPanel { flex: 1; overflow-y: auto; padding: 14px; }
  #mapWrap { flex: 1; position: relative; }
  #map { position: absolute; inset: 0; }
  #propertyForm {
    position: fixed; top: 0; right: 0; height: 100%; width: 380px; max-width: 100vw;
    background: var(--panel); border-left: 1px solid var(--line); z-index: 40;
    transform: translateX(100%); transition: transform .2s ease; overflow-y: auto; padding: 18px;
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
</style>
</head>
<body>
<header>
  <h1>Mapa de Imóveis</h1>
  <button id="btnAddProperty" type="button">+ Adicionar imóvel</button>
  <button id="btnExport" class="ghost" type="button">Exportar</button>
  <button id="btnImport" class="ghost" type="button">Importar</button>
  <input id="importFileInput" type="file" accept="application/json" style="display:none">
</header>
<div class="app">
  <aside id="sidebar">
    <div id="filterPanel"><h2>Filtros</h2></div>
    <div id="listPanel"><h2>Imóveis (<span id="resultCount">0</span>)</h2><div id="listItems"></div></div>
  </aside>
  <div id="mapWrap"><div id="map"></div></div>
</div>
<div id="propertyForm">
  <h2 id="formTitle">Novo imóvel</h2>
  <form id="propertyFormEl"></form>
</div>

<script src="https://unpkg.com/leaflet@1/dist/leaflet.js"></script>
<script defer src="https://unpkg.com/alpinejs@3/dist/cdn.min.js"></script>
<script>
/* === SECTION: CONFIG === */
const MAP_CENTER = [-27.5966, -48.6084];
const MAP_ZOOM = 13;
const STATUS_COLORS = { disponivel: '#4ade80', reservado: '#facc15', vendido: '#ff6b6b', alugado: '#ff6b6b' };
const STATUS_LABELS = { disponivel: 'Disponível', reservado: 'Reservado', vendido: 'Vendido', alugado: 'Alugado' };
const UNIT_TYPES = ['casa','apartamento','comercial','terreno','cobertura','sobrado'];
const UNIT_TYPE_LABELS = { casa:'Casa', apartamento:'Apartamento', comercial:'Comercial', terreno:'Terreno', cobertura:'Cobertura', sobrado:'Sobrado' };
const STATUS_OPTIONS = ['disponivel','reservado','vendido','alugado'];

/* === SECTION: INDEXEDDB === */

/* === SECTION: LOGIC === */

/* === SECTION: STATE === */
const appState = { properties: [], condos: [], condosById: {}, addMode: false, editingPropertyId: null };

/* === SECTION: MAP === */
let map, markersLayer;
const markersById = {};

function initMap() {
  map = L.map('map').setView(MAP_CENTER, MAP_ZOOM);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19
  }).addTo(map);
  markersLayer = L.layerGroup().addTo(map);
}

window.addEventListener('DOMContentLoaded', initMap);

/* === SECTION: FORM === */

/* === SECTION: PHOTOS === */

/* === SECTION: FILTERS === */

/* === SECTION: LIST === */

/* === SECTION: BACKUP === */
</script>
</body>
</html>
```

- [ ] **Step 2: Verificar no navegador**

Abra `mapa-imoveis/index.html` diretamente no navegador (duplo clique ou `file:///caminho/absoluto/mapa-imoveis/index.html`).

Expected: header "Mapa de Imóveis" com os 3 botões visíveis; mapa preenche o resto da tela, centrado em Florianópolis continental, com tiles do OpenStreetMap carregando; sidebar esquerda vazia (só títulos "Filtros" e "Imóveis (0)"); painel de formulário fora da tela à direita (não visível). Console do DevTools (F12) sem erros vermelhos.

- [ ] **Step 3: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: estrutura base do mapa de imóveis (HTML/CSS/Leaflet)"
```

---

### Task 2: Camada de persistência IndexedDB

**Files:**
- Modify: `mapa-imoveis/index.html` (dentro de `/* === SECTION: INDEXEDDB === */`)

**Interfaces:**
- Consumes: nenhuma (base da camada de dados).
- Produces: `dbOpen()`, `dbPut(storeName, record)`, `dbGet(storeName, id)`, `dbGetAll(storeName)`, `dbDelete(storeName, id)` — todas retornam Promise. Stores: `'properties'` e `'condos'`, ambas com `keyPath: 'id'`.

- [ ] **Step 1: Escrever o self-check (vai falhar — funções ainda não existem)**

Insira, logo após `/* === SECTION: INDEXEDDB === */`:

```js
/* === SECTION: INDEXEDDB === */
const DB_NAME = 'mapaImoveisDB';
const DB_VERSION = 1;
let dbInstance = null;

(async function selfCheckDb() {
  try {
    const testRecord = { id: '__selfcheck__', value: 42 };
    await dbPut('properties', testRecord);
    const read = await dbGet('properties', '__selfcheck__');
    console.assert(read && read.value === 42, 'ASSERT FAIL: dbPut/dbGet roundtrip');
    await dbDelete('properties', '__selfcheck__');
    const afterDelete = await dbGet('properties', '__selfcheck__');
    console.assert(afterDelete === null, 'ASSERT FAIL: dbDelete não removeu o registro');
    console.log('[self-check] IndexedDB OK');
  } catch (err) {
    console.error('[self-check] IndexedDB FAILED', err);
  }
})();
```

- [ ] **Step 2: Rodar e confirmar que falha**

Abra `mapa-imoveis/index.html` no navegador, veja o console (F12).
Expected: erro tipo `dbPut is not defined` (ou `ReferenceError`) — as funções ainda não existem.

- [ ] **Step 3: Implementar as funções**

Insira **antes** do bloco `selfCheckDb` (entre `let dbInstance = null;` e `(async function selfCheckDb() {`):

```js
function dbOpen() {
  return new Promise((resolve, reject) => {
    if (dbInstance) return resolve(dbInstance);
    if (!window.indexedDB) return reject(new Error('IndexedDB não suportado neste navegador.'));
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = (e) => {
      const db = e.target.result;
      if (!db.objectStoreNames.contains('properties')) db.createObjectStore('properties', { keyPath: 'id' });
      if (!db.objectStoreNames.contains('condos')) db.createObjectStore('condos', { keyPath: 'id' });
    };
    req.onsuccess = (e) => { dbInstance = e.target.result; resolve(dbInstance); };
    req.onerror = (e) => reject(e.target.error);
  });
}

async function dbPut(storeName, record) {
  const db = await dbOpen();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(storeName, 'readwrite');
    tx.objectStore(storeName).put(record);
    tx.oncomplete = () => resolve(record);
    tx.onerror = (e) => reject(e.target.error);
  });
}

async function dbGet(storeName, id) {
  const db = await dbOpen();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(storeName, 'readonly');
    const req = tx.objectStore(storeName).get(id);
    req.onsuccess = () => resolve(req.result || null);
    req.onerror = (e) => reject(e.target.error);
  });
}

async function dbGetAll(storeName) {
  const db = await dbOpen();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(storeName, 'readonly');
    const req = tx.objectStore(storeName).getAll();
    req.onsuccess = () => resolve(req.result || []);
    req.onerror = (e) => reject(e.target.error);
  });
}

async function dbDelete(storeName, id) {
  const db = await dbOpen();
  return new Promise((resolve, reject) => {
    const tx = db.transaction(storeName, 'readwrite');
    tx.objectStore(storeName).delete(id);
    tx.oncomplete = () => resolve();
    tx.onerror = (e) => reject(e.target.error);
  });
}
```

- [ ] **Step 4: Rodar e confirmar que passa**

Recarregue `mapa-imoveis/index.html` no navegador, veja o console.
Expected: `[self-check] IndexedDB OK`, nenhum `ASSERT FAIL`.

- [ ] **Step 5: Verificar especificamente via `file://` (não só servidor local)**

Como a entrega final é um arquivo aberto direto (duplo clique), confirme que o self-check acima passou abrindo o arquivo via `file:///...` (não `http://localhost`). Alguns navegadores restringem IndexedDB em origens `file://` — se o self-check falhar só nesse modo, anote o navegador testado e sinalize antes de prosseguir (Chrome e Firefox atuais suportam; é a validação que importa aqui).

- [ ] **Step 6: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: camada de persistência IndexedDB (properties/condos)"
```

---

### Task 3: Lógica pura — formatação, validação e matching de filtro

**Files:**
- Modify: `mapa-imoveis/index.html` (dentro de `/* === SECTION: LOGIC === */`)

**Interfaces:**
- Consumes: nenhuma (funções puras, sem I/O).
- Produces: `formatCurrency(value)`, `validateProperty(property)` (retorna array de strings de erro, vazio se válido), `matchesFilters(property, condosById, filters)` (retorna bool). Usadas por `FORM` (Task 4, validação) e `FILTERS` (Task 7, matching).

- [ ] **Step 1: Escrever o self-check (vai falhar — funções ainda não existem)**

Insira, logo após `/* === SECTION: LOGIC === */`:

```js
/* === SECTION: LOGIC === */
(function selfCheckLogic() {
  console.assert(formatCurrency(1200000) === 'R$ 1.200.000,00', 'ASSERT FAIL: formatCurrency básico');
  console.assert(formatCurrency(null) === 'R$ 0,00', 'ASSERT FAIL: formatCurrency null');

  const okProp = { unitType: 'casa', dealType: 'venda', salePrice: 500000, address: { street: 'Rua X' }, coordinates: [-27.6, -48.6] };
  console.assert(validateProperty(okProp).length === 0, 'ASSERT FAIL: validateProperty deveria aceitar imóvel válido');
  const badProp = { unitType: '', dealType: '', salePrice: 0, address: {}, coordinates: null };
  console.assert(validateProperty(badProp).length === 6, 'ASSERT FAIL: validateProperty deveria acusar 6 erros, achou ' + validateProperty(badProp).length);

  const baseFilters = { dealType: 'ambos', unitTypes: [], status: [], priceMin: null, priceMax: null, roomsMin: null, suitesMin: null, bathroomsMin: null, parkingMin: null, areaMin: null, areaMax: null, furnished: 'qualquer', petsAllowed: 'qualquer', neighborhood: '', agentResponsible: '', amenities: [] };
  const propA = { dealType: 'venda', unitType: 'casa', status: 'disponivel', salePrice: 500000, rentPrice: 0, rooms: 3, suites: 1, bathrooms: 2, parkingSpaces: 2, constructedArea: 120, furnished: 'não', petsAllowed: true, address: { neighborhood: 'Capoeiras' }, agentResponsible: 'Ana', condoId: null };
  console.assert(matchesFilters(propA, {}, baseFilters) === true, 'ASSERT FAIL: matchesFilters deveria aceitar sem filtros ativos');
  console.assert(matchesFilters(propA, {}, { ...baseFilters, unitTypes: ['apartamento'] }) === false, 'ASSERT FAIL: matchesFilters deveria rejeitar tipo diferente');
  console.assert(matchesFilters(propA, {}, { ...baseFilters, priceMin: 600000 }) === false, 'ASSERT FAIL: matchesFilters deveria rejeitar preço abaixo do mínimo');
  console.assert(matchesFilters(propA, {}, { ...baseFilters, roomsMin: 2 }) === true, 'ASSERT FAIL: matchesFilters deveria aceitar quartos suficientes');

  const condosById = { condo1: { pool: true, gym: false } };
  const propB = { ...propA, condoId: 'condo1' };
  console.assert(matchesFilters(propB, condosById, { ...baseFilters, amenities: ['pool'] }) === true, 'ASSERT FAIL: matchesFilters deveria aceitar amenidade presente');
  console.assert(matchesFilters(propB, condosById, { ...baseFilters, amenities: ['gym'] }) === false, 'ASSERT FAIL: matchesFilters deveria rejeitar amenidade ausente');

  console.log('[self-check] Lógica de negócio OK');
})();
```

- [ ] **Step 2: Rodar e confirmar que falha**

Abra `mapa-imoveis/index.html` no navegador, veja o console.
Expected: `ReferenceError: formatCurrency is not defined` (ou similar) — funções ainda não existem.

- [ ] **Step 3: Implementar as funções**

Insira **antes** do bloco `selfCheckLogic`:

```js
function formatCurrency(value) {
  if (value == null || isNaN(value)) return 'R$ 0,00';
  return 'R$ ' + Number(value).toLocaleString('pt-BR', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}

function validateProperty(property) {
  const errors = [];
  if (!property.unitType) errors.push('Tipo de imóvel é obrigatório.');
  if (!property.dealType) errors.push('Negócio (venda/aluguel) é obrigatório.');
  if (property.dealType !== 'aluguel' && !(property.salePrice > 0)) errors.push('Preço de venda é obrigatório.');
  if (property.dealType !== 'venda' && !(property.rentPrice > 0)) errors.push('Preço de aluguel é obrigatório.');
  if (!property.address || !property.address.street) errors.push('Endereço é obrigatório.');
  if (!Array.isArray(property.coordinates) || property.coordinates.length !== 2) errors.push('Coordenadas são obrigatórias (clique no mapa).');
  return errors;
}

function matchesFilters(property, condosById, filters) {
  if (filters.dealType !== 'ambos' && property.dealType !== 'venda_e_aluguel' && property.dealType !== filters.dealType) return false;
  if (filters.unitTypes.length && !filters.unitTypes.includes(property.unitType)) return false;
  if (filters.status.length && !filters.status.includes(property.status)) return false;
  const price = filters.dealType === 'aluguel' ? property.rentPrice : property.salePrice;
  if (filters.priceMin != null && price < filters.priceMin) return false;
  if (filters.priceMax != null && price > filters.priceMax) return false;
  if (filters.roomsMin != null && (property.rooms || 0) < filters.roomsMin) return false;
  if (filters.suitesMin != null && (property.suites || 0) < filters.suitesMin) return false;
  if (filters.bathroomsMin != null && (property.bathrooms || 0) < filters.bathroomsMin) return false;
  if (filters.parkingMin != null && (property.parkingSpaces || 0) < filters.parkingMin) return false;
  if (filters.areaMin != null && (property.constructedArea || 0) < filters.areaMin) return false;
  if (filters.areaMax != null && (property.constructedArea || 0) > filters.areaMax) return false;
  if (filters.furnished !== 'qualquer' && property.furnished !== filters.furnished) return false;
  if (filters.petsAllowed !== 'qualquer' && String(property.petsAllowed) !== filters.petsAllowed) return false;
  if (filters.neighborhood && !(property.address.neighborhood || '').toLowerCase().includes(filters.neighborhood.toLowerCase())) return false;
  if (filters.agentResponsible && property.agentResponsible !== filters.agentResponsible) return false;
  if (filters.amenities.length) {
    const condo = property.condoId ? condosById[property.condoId] : null;
    if (!condo) return false;
    for (const amenity of filters.amenities) if (!condo[amenity]) return false;
  }
  return true;
}
```

- [ ] **Step 4: Rodar e confirmar que passa**

Recarregue `mapa-imoveis/index.html`, veja o console.
Expected: `[self-check] Lógica de negócio OK`, nenhum `ASSERT FAIL`.

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: lógica pura de validação e matching de filtro"
```

---

### Task 4: Formulário de cadastro (imóvel + condomínio inline)

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML do `<form id="propertyFormEl">` vazio criado na Task 1; `/* === SECTION: FORM === */`)

**Interfaces:**
- Consumes: `dbPut`, `validateProperty` (Task 2/3); `appState` (Task 1); `map` (Task 1, para o listener de clique).
- Produces: `openPropertyForm(coords, existingProperty)`, `closePropertyForm()`, `readPropertyForm()`, `savePropertyForm(event)`, `refreshCondoSelect()`. Consumido por Task 5 (popup "Editar" chama `openPropertyForm`) e Task 6 (fotos, que estende `openPropertyForm`/`readPropertyForm`).

- [ ] **Step 1: Preencher o HTML do formulário**

Substitua `<form id="propertyFormEl"></form>` (criado na Task 1) por:

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

  <div id="formErrors"></div>

  <div class="btn-row">
    <button type="submit">Salvar</button>
    <button type="button" id="btnCancelForm" class="ghost">Cancelar</button>
  </div>
</form>
```

- [ ] **Step 2: Implementar a lógica do formulário**

Insira dentro de `/* === SECTION: FORM === */`:

```js
/* === SECTION: FORM === */
function openPropertyForm(coords, existingProperty) {
  appState.editingPropertyId = existingProperty ? existingProperty.id : null;
  document.getElementById('formTitle').textContent = existingProperty ? 'Editar imóvel' : 'Novo imóvel';
  const p = existingProperty || {
    address: {}, status: 'disponivel', dealType: 'venda', unitType: 'casa',
    furnished: 'não', petsAllowed: false, coordinates: coords
  };
  document.getElementById('f_id').value = p.id || '';
  document.getElementById('f_lat').value = p.coordinates[0];
  document.getElementById('f_lng').value = p.coordinates[1];
  document.getElementById('f_label').value = p.label || '';
  document.getElementById('f_status').value = p.status;
  document.getElementById('f_dealType').value = p.dealType;
  document.getElementById('f_unitType').value = p.unitType;
  document.getElementById('f_salePrice').value = p.salePrice || '';
  document.getElementById('f_rentPrice').value = p.rentPrice || '';
  document.getElementById('f_constructedArea').value = p.constructedArea || '';
  document.getElementById('f_landArea').value = p.landArea || '';
  document.getElementById('f_rooms').value = p.rooms || '';
  document.getElementById('f_suites').value = p.suites || '';
  document.getElementById('f_bathrooms').value = p.bathrooms || '';
  document.getElementById('f_parkingSpaces').value = p.parkingSpaces || '';
  document.getElementById('f_floor').value = p.floor || '';
  document.getElementById('f_furnished').value = p.furnished || 'não';
  document.getElementById('f_petsAllowed').value = String(!!p.petsAllowed);
  document.getElementById('f_street').value = (p.address && p.address.street) || '';
  document.getElementById('f_number').value = (p.address && p.address.number) || '';
  document.getElementById('f_neighborhood').value = (p.address && p.address.neighborhood) || '';
  document.getElementById('f_city').value = (p.address && p.address.city) || 'Florianópolis';
  document.getElementById('f_complement').value = (p.address && p.address.complement) || '';
  document.getElementById('f_agentResponsible').value = p.agentResponsible || '';
  document.getElementById('f_condoId').value = p.condoId || '';
  document.getElementById('f_notes').value = p.notes || '';
  document.getElementById('newCondoFields').style.display = 'none';
  document.getElementById('formErrors').classList.remove('show');
  document.getElementById('propertyForm').classList.add('open');
}

function closePropertyForm() {
  document.getElementById('propertyForm').classList.remove('open');
  appState.addMode = false;
  appState.editingPropertyId = null;
  document.getElementById('btnAddProperty').classList.remove('active');
}

function readPropertyForm() {
  const existing = appState.editingPropertyId ? appState.properties.find(p => p.id === appState.editingPropertyId) : null;
  return {
    id: document.getElementById('f_id').value || 'p_' + Date.now() + '_' + Math.random().toString(36).slice(2, 8),
    label: document.getElementById('f_label').value.trim(),
    status: document.getElementById('f_status').value,
    dealType: document.getElementById('f_dealType').value,
    unitType: document.getElementById('f_unitType').value,
    salePrice: Number(document.getElementById('f_salePrice').value) || 0,
    rentPrice: Number(document.getElementById('f_rentPrice').value) || 0,
    constructedArea: Number(document.getElementById('f_constructedArea').value) || 0,
    landArea: Number(document.getElementById('f_landArea').value) || 0,
    rooms: Number(document.getElementById('f_rooms').value) || 0,
    suites: Number(document.getElementById('f_suites').value) || 0,
    bathrooms: Number(document.getElementById('f_bathrooms').value) || 0,
    parkingSpaces: Number(document.getElementById('f_parkingSpaces').value) || 0,
    floor: document.getElementById('f_floor').value ? Number(document.getElementById('f_floor').value) : null,
    furnished: document.getElementById('f_furnished').value,
    petsAllowed: document.getElementById('f_petsAllowed').value === 'true',
    address: {
      street: document.getElementById('f_street').value.trim(),
      number: document.getElementById('f_number').value.trim(),
      neighborhood: document.getElementById('f_neighborhood').value.trim(),
      city: document.getElementById('f_city').value.trim(),
      complement: document.getElementById('f_complement').value.trim()
    },
    coordinates: [Number(document.getElementById('f_lat').value), Number(document.getElementById('f_lng').value)],
    agentResponsible: document.getElementById('f_agentResponsible').value.trim(),
    condoId: document.getElementById('f_condoId').value || null,
    notes: document.getElementById('f_notes').value.trim(),
    photos: existing && existing.photos ? existing.photos : [],
    createdAt: existing ? existing.createdAt : Date.now(),
    updatedAt: Date.now()
  };
}

async function handleCondoCreateIfNeeded() {
  const nameField = document.getElementById('c_name');
  if (document.getElementById('newCondoFields').style.display === 'none' || !nameField.value.trim()) return null;
  const condo = {
    id: 'c_' + Date.now() + '_' + Math.random().toString(36).slice(2, 8),
    name: nameField.value.trim(),
    pool: document.getElementById('c_pool').checked,
    gym: document.getElementById('c_gym').checked,
    partyRoom: document.getElementById('c_partyRoom').checked,
    playground: document.getElementById('c_playground').checked,
    petArea: document.getElementById('c_petArea').checked,
    security24h: document.getElementById('c_security24h').checked,
    elevator: document.getElementById('c_elevator').checked,
    gatedCommunity: document.getElementById('c_gatedCommunity').checked,
    monthlyFee: Number(document.getElementById('c_monthlyFee').value) || 0
  };
  await dbPut('condos', condo);
  appState.condos.push(condo);
  appState.condosById[condo.id] = condo;
  refreshCondoSelect();
  return condo.id;
}

function refreshCondoSelect() {
  const select = document.getElementById('f_condoId');
  const current = select.value;
  select.innerHTML = '<option value="">Nenhum</option>' +
    appState.condos.map(c => `<option value="${c.id}">${c.name}</option>`).join('');
  select.value = current;
}

async function savePropertyForm(event) {
  event.preventDefault();
  const newCondoId = await handleCondoCreateIfNeeded();
  if (newCondoId) document.getElementById('f_condoId').value = newCondoId;
  const property = readPropertyForm();
  const errors = validateProperty(property);
  const errorsEl = document.getElementById('formErrors');
  if (errors.length) {
    errorsEl.innerHTML = errors.map(e => '• ' + e).join('<br>');
    errorsEl.classList.add('show');
    return;
  }
  errorsEl.classList.remove('show');
  await dbPut('properties', property);
  const idx = appState.properties.findIndex(p => p.id === property.id);
  if (idx >= 0) appState.properties[idx] = property; else appState.properties.push(property);
  window.dispatchEvent(new CustomEvent('properties-updated'));
  closePropertyForm();
}

window.addEventListener('DOMContentLoaded', () => {
  document.getElementById('propertyFormEl').addEventListener('submit', savePropertyForm);
  document.getElementById('btnCancelForm').addEventListener('click', closePropertyForm);
  document.getElementById('btnNewCondo').addEventListener('click', () => {
    const el = document.getElementById('newCondoFields');
    el.style.display = el.style.display === 'none' ? 'block' : 'none';
  });
  document.getElementById('btnAddProperty').addEventListener('click', () => {
    appState.addMode = !appState.addMode;
    document.getElementById('btnAddProperty').classList.toggle('active', appState.addMode);
  });
  map.on('click', (e) => {
    if (!appState.addMode) return;
    openPropertyForm([e.latlng.lat, e.latlng.lng], null);
    appState.addMode = false;
    document.getElementById('btnAddProperty').classList.remove('active');
  });
});
```

> Nota: `savePropertyForm` ainda não desenha o pin no mapa nem persiste visualmente — isso entra na Task 5 (`upsertMarker`). Por enquanto, salvar só grava no IndexedDB e no cache em memória.

- [ ] **Step 3: Verificar manualmente no navegador**

Abra `mapa-imoveis/index.html`. Clique em "+ Adicionar imóvel" (botão fica destacado), clique em qualquer ponto do mapa.
Expected: painel lateral desliza da direita com "Novo imóvel", campo de coordenadas oculto preenchido. Preencha Tipo=Casa, Negócio=Venda, Preço de venda=500000, Rua="Rua Teste". Clique "Salvar".
Expected: painel fecha sem erros no console. Recarregue a página, abra o DevTools console e rode `dbGetAll('properties').then(console.log)` — deve listar o imóvel salvo.

Tente salvar sem preencher nada (clique "+ Adicionar imóvel" → clique no mapa → "Salvar" direto).
Expected: painel mostra lista de erros em vermelho, não fecha.

- [ ] **Step 4: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: formulário de cadastro de imóvel com condomínio inline"
```

---

### Task 5: Pins no mapa (renderizar, editar, excluir, arrastar)

**Files:**
- Modify: `mapa-imoveis/index.html` (append ao final de `/* === SECTION: MAP === */`, e um ajuste em `savePropertyForm` da Task 4)

**Interfaces:**
- Consumes: `appState`, `markersLayer`, `dbGetAll`, `dbDelete`, `dbPut` (Task 2), `openPropertyForm` (Task 4), `STATUS_COLORS`, `STATUS_LABELS`, `UNIT_TYPE_LABELS`, `formatCurrency` (Task 3).
- Produces: `upsertMarker(property)`, `removeMarker(id)`, `deleteProperty(id)`, `loadAllProperties()`. Consumido por `FORM` (Task 4, precisa chamar `upsertMarker` após salvar), `FILTERS` (Task 7, esconde/mostra via `markersById`), `BACKUP` (Task 9, recarrega via `loadAllProperties`).

- [ ] **Step 1: Adicionar renderização de marcadores**

Insira ao final de `/* === SECTION: MAP === */` (depois de `window.addEventListener('DOMContentLoaded', initMap);`):

```js
function buildMarkerIcon(status) {
  const color = STATUS_COLORS[status] || '#999';
  return L.divIcon({
    className: '',
    html: `<div style="width:16px;height:16px;border-radius:50%;background:${color};border:2px solid #04121f;"></div>`,
    iconSize: [16, 16],
    iconAnchor: [8, 8]
  });
}

function buildPopupHtml(property) {
  const price = property.dealType === 'aluguel' ? property.rentPrice : property.salePrice;
  const priceLabel = property.dealType === 'aluguel' ? formatCurrency(price) + '/mês' : formatCurrency(price);
  const addr = property.address || {};
  return `
    <div style="min-width:200px;">
      <strong>${priceLabel}</strong><br>
      ${UNIT_TYPE_LABELS[property.unitType] || property.unitType} · ${STATUS_LABELS[property.status] || property.status}<br>
      ${addr.street || ''} ${addr.number || ''} - ${addr.neighborhood || ''}<br>
      <div style="margin-top:8px; display:flex; gap:6px;">
        <button type="button" onclick="openPropertyForm(null, appState.properties.find(p => p.id === '${property.id}'))">Editar</button>
        <button type="button" onclick="deleteProperty('${property.id}')">Excluir</button>
      </div>
    </div>`;
}

function upsertMarker(property) {
  const existing = markersById[property.id];
  if (existing) markersLayer.removeLayer(existing);
  const marker = L.marker(property.coordinates, { draggable: true, icon: buildMarkerIcon(property.status) });
  marker.bindPopup(() => buildPopupHtml(property));
  marker.on('dragend', async () => {
    const pos = marker.getLatLng();
    property.coordinates = [pos.lat, pos.lng];
    await dbPut('properties', property);
  });
  marker.addTo(markersLayer);
  markersById[property.id] = marker;
}

function removeMarker(id) {
  const marker = markersById[id];
  if (marker) { markersLayer.removeLayer(marker); delete markersById[id]; }
}

async function deleteProperty(id) {
  if (!confirm('Excluir este imóvel?')) return;
  await dbDelete('properties', id);
  appState.properties = appState.properties.filter(p => p.id !== id);
  removeMarker(id);
  window.dispatchEvent(new CustomEvent('properties-updated'));
}

async function loadAllProperties() {
  appState.condos = await dbGetAll('condos');
  appState.condosById = Object.fromEntries(appState.condos.map(c => [c.id, c]));
  appState.properties = await dbGetAll('properties');
  appState.properties.forEach(upsertMarker);
  refreshCondoSelect();
  window.dispatchEvent(new CustomEvent('properties-updated'));
}

window.addEventListener('DOMContentLoaded', loadAllProperties);
```

- [ ] **Step 2: Ligar `savePropertyForm` (Task 4) ao desenho do pin**

Em `savePropertyForm`, dentro de `/* === SECTION: FORM === */`, localize a linha:

```js
  if (idx >= 0) appState.properties[idx] = property; else appState.properties.push(property);
```

e adicione logo abaixo:

```js
  upsertMarker(property);
```

- [ ] **Step 3: Verificar manualmente no navegador**

Recarregue `mapa-imoveis/index.html`.
Expected: o imóvel salvo na Task 4 já aparece como pin verde (status "disponivel") no local clicado.

Clique no pin → popup mostra preço, tipo, status, endereço e botões "Editar"/"Excluir".
Clique "Editar" → formulário abre pré-preenchido com os dados salvos → altere o Status para "Vendido" → Salvar.
Expected: pin muda de cor pra vermelho.

Arraste o pin pra outro ponto do mapa. Recarregue a página.
Expected: pin aparece na nova posição (persistiu).

Clique "Excluir" → confirme.
Expected: pin some do mapa; recarregando a página, continua sumido.

- [ ] **Step 4: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: renderização de pins com editar/excluir/arrastar"
```

---

### Task 6: Fotos do imóvel

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML do form, `/* === SECTION: PHOTOS === */`, e ajustes em `openPropertyForm`/`readPropertyForm` da Task 4)

**Interfaces:**
- Consumes: `appState.editingPropertyId` (Task 4).
- Produces: variável `formPhotos` (array de `Blob` em memória enquanto o formulário está aberto), `resizeImageToBlob(file, maxWidth)`, `renderPhotoThumbnails()`. Integra com `readPropertyForm` (Task 4), que passa a usar `formPhotos` como `photos` do registro salvo.

- [ ] **Step 1: Adicionar campo de fotos no HTML do formulário**

No `<form id="propertyFormEl">` (Task 4), insira logo antes de `<div id="formErrors"></div>`:

```html
  <label for="f_photos">Fotos</label>
  <input id="f_photos" type="file" accept="image/*" multiple>
  <div id="photoThumbs" style="margin-top:8px;"></div>
```

- [ ] **Step 2: Implementar captura, redimensionamento e miniaturas**

Insira dentro de `/* === SECTION: PHOTOS === */`:

```js
/* === SECTION: PHOTOS === */
let formPhotos = [];

function resizeImageToBlob(file, maxWidth) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    const reader = new FileReader();
    reader.onload = () => { img.src = reader.result; };
    reader.onerror = reject;
    img.onload = () => {
      const scale = Math.min(1, maxWidth / img.width);
      const canvas = document.createElement('canvas');
      canvas.width = Math.round(img.width * scale);
      canvas.height = Math.round(img.height * scale);
      const ctx = canvas.getContext('2d');
      ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
      canvas.toBlob((blob) => blob ? resolve(blob) : reject(new Error('Falha ao gerar imagem')), 'image/jpeg', 0.85);
    };
    img.onerror = reject;
    reader.readAsDataURL(file);
  });
}

function renderPhotoThumbnails() {
  const wrap = document.getElementById('photoThumbs');
  wrap.innerHTML = '';
  formPhotos.forEach((blob, idx) => {
    const url = URL.createObjectURL(blob);
    const thumb = document.createElement('div');
    thumb.style.cssText = 'position:relative;display:inline-block;margin:4px;';
    thumb.innerHTML = `<img src="${url}" style="width:64px;height:64px;object-fit:cover;border-radius:6px;">
      <button type="button" data-idx="${idx}" style="position:absolute;top:-6px;right:-6px;width:18px;height:18px;border-radius:50%;background:var(--warn-fg);color:#04121f;border:0;cursor:pointer;font-size:11px;line-height:1;">×</button>`;
    wrap.appendChild(thumb);
  });
  wrap.querySelectorAll('button[data-idx]').forEach(btn => {
    btn.addEventListener('click', () => {
      formPhotos.splice(Number(btn.dataset.idx), 1);
      renderPhotoThumbnails();
    });
  });
}

window.addEventListener('DOMContentLoaded', () => {
  document.getElementById('f_photos').addEventListener('change', async (e) => {
    for (const file of e.target.files) {
      const blob = await resizeImageToBlob(file, 1600);
      formPhotos.push(blob);
    }
    renderPhotoThumbnails();
    e.target.value = '';
  });
});
```

- [ ] **Step 3: Ligar `formPhotos` ao abrir/ler o formulário (Task 4)**

Em `openPropertyForm` (`/* === SECTION: FORM === */`), localize a linha:

```js
  document.getElementById('propertyForm').classList.add('open');
```

e adicione **antes** dela:

```js
  formPhotos = p.photos ? p.photos.slice() : [];
  renderPhotoThumbnails();
```

Em `readPropertyForm`, localize a linha:

```js
    photos: existing && existing.photos ? existing.photos : [],
```

e substitua por:

```js
    photos: formPhotos.slice(),
```

- [ ] **Step 4: Verificar manualmente no navegador**

Recarregue `mapa-imoveis/index.html`, abra "Editar" num imóvel existente, anexe 2 fotos pelo campo de arquivo.
Expected: 2 miniaturas quadradas aparecem abaixo do campo. Clique no "×" de uma — some. Salve.
Recarregue a página, abra "Editar" no mesmo imóvel novamente.
Expected: a foto restante ainda aparece (persistiu no IndexedDB como Blob).

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: captura, redimensionamento e persistência de fotos"
```

---

### Task 7: Painel de filtros (Alpine.js) sincronizado com o mapa

**Files:**
- Modify: `mapa-imoveis/index.html` (`<style>` da Task 1, `<div id="filterPanel">` da Task 1, `/* === SECTION: FILTERS === */`)

**Interfaces:**
- Consumes: `matchesFilters` (Task 3), `appState.properties`/`appState.condosById` (Task 5), `markersById`/`markersLayer` (Task 1/5), `STATUS_OPTIONS`/`UNIT_TYPES`/`UNIT_TYPE_LABELS`/`STATUS_LABELS` (Task 1).
- Produces: componente Alpine `filterApp()`; evento customizado `filters-applied` (detail: `{ filtered: Property[] }`), consumido por `LIST` (Task 8).

- [ ] **Step 1: CSS adicional pro painel de filtro**

No bloco `<style>` (Task 1), insira **antes** de `</style>`:

```css
  #filterPanel label, #listPanel label { display:block; font-size:12px; color:var(--muted); margin:8px 0 4px; text-transform:none; }
  #filterPanel select, #filterPanel input[type=text], #filterPanel input[type=number] {
    width:100%; background:var(--panel-2); border:1px solid var(--line); color:var(--text);
    padding:6px 8px; border-radius:6px; font-size:13px;
  }
  #filterPanel label.checkbox-label { display:block; font-size:13px; font-weight:normal; color:var(--text); margin:2px 0; }
  .list-item:hover { background: var(--panel-2); }
```

- [ ] **Step 2: HTML do painel de filtros com Alpine**

Substitua `<div id="filterPanel"><h2>Filtros</h2></div>` (Task 1) por:

```html
    <div id="filterPanel" x-data="filterApp()" x-init="init()">
      <h2>Filtros</h2>

      <label>Negócio</label>
      <select x-model="filters.dealType" @change="apply()">
        <option value="ambos">Ambos</option>
        <option value="venda">Venda</option>
        <option value="aluguel">Aluguel</option>
      </select>

      <label>Tipo de imóvel</label>
      <template x-for="ut in unitTypeOptions" :key="ut.value">
        <label class="checkbox-label">
          <input type="checkbox" :value="ut.value" x-model="filters.unitTypes" @change="apply()"> <span x-text="ut.label"></span>
        </label>
      </template>

      <label>Status</label>
      <template x-for="st in statusOptions" :key="st.value">
        <label class="checkbox-label">
          <input type="checkbox" :value="st.value" x-model="filters.status" @change="apply()"> <span x-text="st.label"></span>
        </label>
      </template>

      <label>Preço mín / máx</label>
      <div class="row">
        <input type="number" x-model.number="filters.priceMin" @input="apply()" placeholder="mín">
        <input type="number" x-model.number="filters.priceMax" @input="apply()" placeholder="máx">
      </div>

      <label>Quartos mín</label>
      <input type="number" x-model.number="filters.roomsMin" @input="apply()">
      <label>Suítes mín</label>
      <input type="number" x-model.number="filters.suitesMin" @input="apply()">
      <label>Banheiros mín</label>
      <input type="number" x-model.number="filters.bathroomsMin" @input="apply()">
      <label>Vagas mín</label>
      <input type="number" x-model.number="filters.parkingMin" @input="apply()">

      <label>Área mín / máx (m²)</label>
      <div class="row">
        <input type="number" x-model.number="filters.areaMin" @input="apply()" placeholder="mín">
        <input type="number" x-model.number="filters.areaMax" @input="apply()" placeholder="máx">
      </div>

      <label>Mobiliado</label>
      <select x-model="filters.furnished" @change="apply()">
        <option value="qualquer">Qualquer</option>
        <option value="sim">Sim</option>
        <option value="não">Não</option>
        <option value="parcial">Parcial</option>
      </select>

      <label>Aceita pet</label>
      <select x-model="filters.petsAllowed" @change="apply()">
        <option value="qualquer">Qualquer</option>
        <option value="true">Sim</option>
        <option value="false">Não</option>
      </select>

      <label>Bairro</label>
      <input type="text" x-model="filters.neighborhood" @input="apply()">

      <label>Corretor</label>
      <input type="text" x-model="filters.agentResponsible" @input="apply()">

      <label>Amenidades do condomínio</label>
      <template x-for="am in amenityOptions" :key="am.value">
        <label class="checkbox-label">
          <input type="checkbox" :value="am.value" x-model="filters.amenities" @change="apply()"> <span x-text="am.label"></span>
        </label>
      </template>
    </div>
```

- [ ] **Step 3: Implementar `filterApp()` e sincronização com o mapa**

Insira dentro de `/* === SECTION: FILTERS === */`:

```js
/* === SECTION: FILTERS === */
function filterApp() {
  return {
    filters: {
      dealType: 'ambos', unitTypes: [], status: STATUS_OPTIONS.slice(),
      priceMin: null, priceMax: null,
      roomsMin: null, suitesMin: null, bathroomsMin: null, parkingMin: null,
      areaMin: null, areaMax: null,
      furnished: 'qualquer', petsAllowed: 'qualquer',
      neighborhood: '', agentResponsible: '', amenities: []
    },
    unitTypeOptions: UNIT_TYPES.map(v => ({ value: v, label: UNIT_TYPE_LABELS[v] })),
    statusOptions: STATUS_OPTIONS.map(v => ({ value: v, label: STATUS_LABELS[v] })),
    amenityOptions: [
      { value: 'pool', label: 'Piscina' }, { value: 'gym', label: 'Academia' },
      { value: 'partyRoom', label: 'Salão de festas' }, { value: 'playground', label: 'Playground' },
      { value: 'petArea', label: 'Área pet' }, { value: 'security24h', label: 'Portaria 24h' },
      { value: 'elevator', label: 'Elevador' }, { value: 'gatedCommunity', label: 'Condomínio fechado' }
    ],
    init() {
      window.addEventListener('properties-updated', () => this.apply());
      this.apply();
    },
    apply() {
      const filtered = appState.properties.filter(p => matchesFilters(p, appState.condosById, this.filters));
      window.dispatchEvent(new CustomEvent('filters-applied', { detail: { filtered } }));
    }
  };
}

window.addEventListener('filters-applied', (e) => {
  document.getElementById('resultCount').textContent = e.detail.filtered.length;
  const filteredIds = new Set(e.detail.filtered.map(p => p.id));
  Object.entries(markersById).forEach(([id, marker]) => {
    const shouldShow = filteredIds.has(id);
    const isShown = markersLayer.hasLayer(marker);
    if (shouldShow && !isShown) marker.addTo(markersLayer);
    if (!shouldShow && isShown) markersLayer.removeLayer(marker);
  });
});
```

- [ ] **Step 4: Verificar manualmente no navegador**

Recarregue `mapa-imoveis/index.html` com pelo menos 2 imóveis cadastrados com tipos diferentes (ex: um "casa", um "apartamento").
Expected: painel de filtros aparece na sidebar com todos os controles; "Imóveis (2)" no topo da lista.

Marque só "Apartamento" em Tipo de imóvel.
Expected: contador muda pra "Imóveis (1)"; o pin do imóvel tipo "casa" some do mapa; o pin do tipo "apartamento" continua visível.

Desmarque tudo (volta a nenhum filtro de tipo ativo).
Expected: os dois pins voltam a aparecer.

- [ ] **Step 5: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: painel de filtros (Alpine.js) sincronizado com o mapa"
```

---

### Task 8: Lista lateral sincronizada com o filtro

**Files:**
- Modify: `mapa-imoveis/index.html` (`/* === SECTION: LIST === */`)

**Interfaces:**
- Consumes: evento `filters-applied` (Task 7), `markersById`/`map` (Task 1/5), `formatCurrency` (Task 3), `UNIT_TYPE_LABELS` (Task 1).
- Produces: `renderList(properties)`.

- [ ] **Step 1: Implementar a lista**

Insira dentro de `/* === SECTION: LIST === */`:

```js
/* === SECTION: LIST === */
function renderList(properties) {
  const wrap = document.getElementById('listItems');
  wrap.innerHTML = properties.map(p => {
    const price = p.dealType === 'aluguel' ? p.rentPrice : p.salePrice;
    const addr = p.address || {};
    return `<div class="list-item" data-id="${p.id}" style="padding:8px;border-bottom:1px solid var(--line);cursor:pointer;">
      <div style="font-weight:600;">${formatCurrency(price)}</div>
      <div style="font-size:12px;color:var(--muted);">${UNIT_TYPE_LABELS[p.unitType] || p.unitType} · ${addr.neighborhood || ''}</div>
    </div>`;
  }).join('');
  wrap.querySelectorAll('.list-item').forEach(el => {
    el.addEventListener('click', () => {
      const id = el.dataset.id;
      const marker = markersById[id];
      if (!marker) return;
      map.setView(marker.getLatLng(), 16);
      marker.openPopup();
    });
  });
}

window.addEventListener('filters-applied', (e) => renderList(e.detail.filtered));
```

- [ ] **Step 2: Verificar manualmente no navegador**

Recarregue `mapa-imoveis/index.html` com pelo menos 2 imóveis.
Expected: lista lateral abaixo dos filtros mostra um item por imóvel (preço + tipo + bairro), atualizando junto com os filtros da Task 7.

Clique num item da lista.
Expected: mapa centraliza e dá zoom no pin correspondente, popup abre automaticamente.

- [ ] **Step 3: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: lista lateral sincronizada com filtro e clique-para-centralizar"
```

---

### Task 9: Backup (exportar/importar) e aviso de navegador incompatível

**Files:**
- Modify: `mapa-imoveis/index.html` (`/* === SECTION: BACKUP === */`, ajuste no listener de `loadAllProperties` da Task 5)

**Interfaces:**
- Consumes: `dbGetAll`, `dbPut` (Task 2), `loadAllProperties` (Task 5).
- Produces: `exportBackup()`, `importBackup(file)`.

- [ ] **Step 1: Implementar exportar/importar**

Insira dentro de `/* === SECTION: BACKUP === */`:

```js
/* === SECTION: BACKUP === */
function blobToBase64(blob) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result);
    reader.onerror = reject;
    reader.readAsDataURL(blob);
  });
}

function base64ToBlob(dataUrl) {
  return fetch(dataUrl).then(r => r.blob());
}

async function exportBackup() {
  const properties = await dbGetAll('properties');
  const condos = await dbGetAll('condos');
  const propertiesOut = [];
  for (const p of properties) {
    const photos = [];
    for (const blob of (p.photos || [])) photos.push(await blobToBase64(blob));
    propertiesOut.push({ ...p, photos });
  }
  const payload = { version: 1, exportedAt: new Date().toISOString(), properties: propertiesOut, condos };
  const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'mapa-imoveis-backup-' + new Date().toISOString().slice(0, 10) + '.json';
  a.click();
  URL.revokeObjectURL(url);
}

async function importBackup(file) {
  const text = await file.text();
  const payload = JSON.parse(text);
  for (const condo of (payload.condos || [])) await dbPut('condos', condo);
  for (const p of (payload.properties || [])) {
    const photos = [];
    for (const dataUrl of (p.photos || [])) photos.push(await base64ToBlob(dataUrl));
    await dbPut('properties', { ...p, photos });
  }
  await loadAllProperties();
  alert('Importação concluída.');
}

window.addEventListener('DOMContentLoaded', () => {
  document.getElementById('btnExport').addEventListener('click', exportBackup);
  document.getElementById('btnImport').addEventListener('click', () => document.getElementById('importFileInput').click());
  document.getElementById('importFileInput').addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (file) importBackup(file);
    e.target.value = '';
  });
});
```

- [ ] **Step 2: Aviso de navegador sem IndexedDB**

Em `/* === SECTION: MAP === */` (Task 5), localize:

```js
window.addEventListener('DOMContentLoaded', loadAllProperties);
```

e substitua por:

```js
window.addEventListener('DOMContentLoaded', () => {
  loadAllProperties().catch(err => {
    console.error(err);
    const banner = document.createElement('div');
    banner.textContent = 'Este navegador não suporta o armazenamento necessário (IndexedDB). Use um navegador atualizado (Chrome, Firefox, Edge).';
    banner.style.cssText = 'background:var(--warn-bg);color:var(--warn-fg);border-bottom:1px solid var(--warn-line);padding:10px 20px;text-align:center;';
    document.body.prepend(banner);
  });
});
```

- [ ] **Step 3: Verificar manualmente no navegador**

Com pelo menos 1 imóvel (com foto) cadastrado, clique "Exportar".
Expected: baixa um arquivo `mapa-imoveis-backup-AAAA-MM-DD.json` com os dados (abra num editor de texto pra conferir que `properties` e `condos` estão preenchidos e a foto virou uma string base64 longa).

Exclua todos os imóveis pela UI (ou abra o DevTools console e rode `indexedDB.deleteDatabase('mapaImoveisDB')` e recarregue). Clique "Importar", selecione o `.json` exportado.
Expected: alerta "Importação concluída."; pins reaparecem no mapa com a mesma foto.

- [ ] **Step 4: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: exportar/importar backup JSON e aviso de IndexedDB indisponível"
```

---

### Task 10: Verificação end-to-end do MVP

**Files:**
- Nenhum arquivo novo — apenas verificação manual cobrindo o fluxo completo do spec.

**Interfaces:**
- Consumes: aplicação completa (Tasks 1-9).
- Produces: nenhuma — task de validação final.

- [ ] **Step 1: Rodar todos os self-checks juntos**

Abra `mapa-imoveis/index.html` (via `file://`, o modo real de uso), abra o console (F12).
Expected: `[self-check] IndexedDB OK` e `[self-check] Lógica de negócio OK`, sem nenhum `ASSERT FAIL` nem erro vermelho.

- [ ] **Step 2: Fluxo completo — cadastro, condomínio, foto, edição**

1. "+ Adicionar imóvel" → clique num ponto do mapa → preencha um imóvel tipo "apartamento", negócio "venda", crie um condomínio novo com Piscina e Portaria 24h marcadas, anexe 1 foto → Salvar.
2. Repita cadastrando um segundo imóvel tipo "casa", negócio "aluguel", sem condomínio, num ponto diferente do mapa.

Expected: 2 pins no mapa, cores conforme status "disponivel" (verde) em ambos.

- [ ] **Step 3: Fluxo completo — filtro por amenidade de condomínio**

No painel de filtros, marque a amenidade "Piscina".
Expected: só o pin do imóvel vinculado ao condomínio com piscina permanece visível; lista lateral mostra 1 item; contador "Imóveis (1)".

Desmarque a amenidade.
Expected: os 2 pins voltam.

- [ ] **Step 4: Fluxo completo — backup**

Exporte o backup (`.json`), confira que os 2 imóveis e o condomínio estão no arquivo. Apague o IndexedDB (`indexedDB.deleteDatabase('mapaImoveisDB')` no console + recarregar), importe o `.json`.
Expected: os 2 pins e o condomínio voltam exatamente como antes, incluindo a foto.

- [ ] **Step 5: Commit final (se algo precisou de ajuste)**

Se qualquer verificação acima falhar, corrija o arquivo e:

```bash
git add mapa-imoveis/index.html
git commit -m "fix: ajustes de verificação end-to-end do MVP"
```

Se tudo passou de primeira, não há o que commitar nesta task.

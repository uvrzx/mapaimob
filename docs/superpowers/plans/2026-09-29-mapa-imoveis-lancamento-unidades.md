# Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adiciona a opção "É lançamento?" no imóvel com total/disponível de
unidades, editável no formulário e ajustável a qualquer momento via +/- no
popup do pin e no card da lista, sem precisar abrir o formulário.

**Architecture:** Continua em `mapa-imoveis/index.html` (arquivo único).
Task 1 cobre o "caminho completo" (dado + formulário + validação); Task 2
cobre o "ajuste rápido" (popup/lista + função de mutação). Task 2 depende
dos campos que a Task 1 introduz no objeto `property`, mas as duas são
verificáveis e revisáveis em separado.

**Tech Stack:** Mesmo stack de sempre — HTML/CSS/JS vanilla + Alpine.js +
Leaflet, sem framework de teste. Verificação via Browser pane
(`preview_start({name: "mapa-imoveis"})`) + `console.assert`.

## Global Constraints

- Arquivo único: `mapa-imoveis/index.html`. Sem novos arquivos/dependências.
- Sem migração de schema do IndexedDB — `isLaunch`/`totalUnits`/
  `availableUnits` são só campos novos no objeto `property`, a store
  `properties` não tem colunas fixas.
- `isLaunch`/`totalUnits`/`availableUnits` só relevantes quando
  `isLaunch === true`; imóveis normais ficam com `false`/`0`/`0` e não
  aparecem em nenhuma UI.
- Toda lógica não-trivial nova ganha `console.assert` de self-check.
- IDs de campos existentes não podem mudar.
- IndexedDB não funciona em `file://` — testar via `preview_start`.

---

### Task 1: Dado + formulário + validação de lançamento

**Files:**
- Modify: `mapa-imoveis/index.html` (HTML do form, `validateProperty`,
  `selfCheckLogic`, `readPropertyForm`, `openPropertyForm`, listener JS)

**Interfaces:**
- Produces: `property.isLaunch` (bool), `property.totalUnits` (number),
  `property.availableUnits` (number) — únicos nomes usados por qualquer
  código que ler/escrever essas propriedades (Task 2 depende deles).
- Consumes: nenhuma interface de outra task.

- [ ] **Step 1: Adicionar checkbox + campos de unidade no HTML do form, seção Básico**

Old:
```html
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
```
New:
```html
      <label for="f_unitType">Tipo de imóvel</label>
      <select id="f_unitType">
        <option value="casa">Casa</option>
        <option value="apartamento">Apartamento</option>
        <option value="comercial">Comercial</option>
        <option value="terreno">Terreno</option>
        <option value="cobertura">Cobertura</option>
        <option value="sobrado">Sobrado</option>
      </select>

      <label><input type="checkbox" id="f_isLaunch"> É lançamento?</label>
      <div id="launchUnitsFields" style="display:none;">
        <div class="row">
          <div>
            <label for="f_totalUnits">Total de unidades</label>
            <input id="f_totalUnits" type="number" min="0" step="1">
          </div>
          <div>
            <label for="f_availableUnits">Disponíveis agora</label>
            <input id="f_availableUnits" type="number" min="0" step="1">
          </div>
        </div>
      </div>
    </div>
```

- [ ] **Step 2: `validateProperty` ganha as 2 regras de lançamento**

Old:
```js
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
```
New:
```js
function validateProperty(property) {
  const errors = [];
  if (!property.unitType) errors.push('Tipo de imóvel é obrigatório.');
  if (!property.dealType) errors.push('Negócio (venda/aluguel) é obrigatório.');
  if (property.dealType !== 'aluguel' && !(property.salePrice > 0)) errors.push('Preço de venda é obrigatório.');
  if (property.dealType !== 'venda' && !(property.rentPrice > 0)) errors.push('Preço de aluguel é obrigatório.');
  if (!property.address || !property.address.street) errors.push('Endereço é obrigatório.');
  if (!Array.isArray(property.coordinates) || property.coordinates.length !== 2) errors.push('Coordenadas são obrigatórias (clique no mapa).');
  if (property.isLaunch && !(property.totalUnits > 0)) errors.push('Lançamento precisa de total de unidades maior que zero.');
  if (property.isLaunch && (property.availableUnits < 0 || property.availableUnits > property.totalUnits)) errors.push('Unidades disponíveis deve estar entre 0 e o total.');
  return errors;
}
```

- [ ] **Step 3: Self-check das novas regras (`selfCheckLogic`, logo após o assert de `badProp`)**

Old:
```js
  const okProp = { unitType: 'casa', dealType: 'venda', salePrice: 500000, address: { street: 'Rua X' }, coordinates: [-27.6, -48.6] };
  console.assert(validateProperty(okProp).length === 0, 'ASSERT FAIL: validateProperty deveria aceitar imóvel válido');
  const badProp = { unitType: '', dealType: '', salePrice: 0, address: {}, coordinates: null };
  console.assert(validateProperty(badProp).length === 6, 'ASSERT FAIL: validateProperty deveria acusar 6 erros, achou ' + validateProperty(badProp).length);
```
New:
```js
  const okProp = { unitType: 'casa', dealType: 'venda', salePrice: 500000, address: { street: 'Rua X' }, coordinates: [-27.6, -48.6] };
  console.assert(validateProperty(okProp).length === 0, 'ASSERT FAIL: validateProperty deveria aceitar imóvel válido');
  const badProp = { unitType: '', dealType: '', salePrice: 0, address: {}, coordinates: null };
  console.assert(validateProperty(badProp).length === 6, 'ASSERT FAIL: validateProperty deveria acusar 6 erros, achou ' + validateProperty(badProp).length);

  const launchPropOk = { ...okProp, isLaunch: true, totalUnits: 40, availableUnits: 40 };
  console.assert(validateProperty(launchPropOk).length === 0, 'ASSERT FAIL: validateProperty deveria aceitar lançamento válido');
  const launchPropBadTotal = { ...okProp, isLaunch: true, totalUnits: 0, availableUnits: 0 };
  console.assert(validateProperty(launchPropBadTotal).length === 1, 'ASSERT FAIL: validateProperty deveria rejeitar lançamento sem total de unidades');
  const launchPropBadAvailable = { ...okProp, isLaunch: true, totalUnits: 10, availableUnits: 15 };
  console.assert(validateProperty(launchPropBadAvailable).length === 1, 'ASSERT FAIL: validateProperty deveria rejeitar disponíveis maior que o total');
```

Nota: essa mudança NÃO altera a contagem de 6 erros do `badProp` — ele não
define `isLaunch`, então as 2 regras novas ficam `undefined && ...` = falsy
e não somam erro.

- [ ] **Step 4: `readPropertyForm` lê os 3 campos novos**

Old:
```js
    petsAllowed: $('f_petsAllowed').value === 'true',
    address: {
```
New:
```js
    petsAllowed: $('f_petsAllowed').value === 'true',
    isLaunch: $('f_isLaunch').checked,
    totalUnits: Number($('f_totalUnits').value) || 0,
    availableUnits: Number($('f_availableUnits').value) || 0,
    address: {
```

- [ ] **Step 5: `openPropertyForm` preenche os 3 campos novos ao abrir/editar**

Old:
```js
  $('f_furnished').value = p.furnished || 'não';
  $('f_petsAllowed').value = String(!!p.petsAllowed);
  $('f_street').value = (p.address && p.address.street) || '';
```
New:
```js
  $('f_furnished').value = p.furnished || 'não';
  $('f_petsAllowed').value = String(!!p.petsAllowed);
  $('f_isLaunch').checked = !!p.isLaunch;
  $('f_totalUnits').value = p.totalUnits || '';
  $('f_availableUnits').value = p.availableUnits || '';
  $('launchUnitsFields').style.display = p.isLaunch ? 'block' : 'none';
  $('f_street').value = (p.address && p.address.street) || '';
```

- [ ] **Step 6: listener de show/hide do bloco de unidades (junto dos outros listeners de formulário, no `DOMContentLoaded` que registra `btnNewCondo`)**

Old:
```js
  $('btnNewCondo').addEventListener('click', () => {
    const el = $('newCondoFields');
    el.style.display = el.style.display === 'none' ? 'block' : 'none';
  });
```
New:
```js
  $('btnNewCondo').addEventListener('click', () => {
    const el = $('newCondoFields');
    el.style.display = el.style.display === 'none' ? 'block' : 'none';
  });
  $('f_isLaunch').addEventListener('change', () => {
    $('launchUnitsFields').style.display = $('f_isLaunch').checked ? 'block' : 'none';
  });
```

- [ ] **Step 7: Teste manual — criar um lançamento pelo formulário**

Via Browser pane com `preview_start({name: "mapa-imoveis"})`: abrir "+
Adicionar imóvel", via `javascript_tool` marcar `$('f_isLaunch').checked =
true` e disparar `change`, confirmar que `#launchUnitsFields` fica visível
(`getComputedStyle(...).display !== 'none'`), preencher
`f_totalUnits=40`/`f_availableUnits=40` + campos obrigatórios (rua,
lat/lng, preço), submeter, e checar
`appState.properties.find(...).isLaunch === true` e `totalUnits === 40`.
Depois testar o caso de erro: `availableUnits=50` (maior que o total),
confirmar que `#formErrors` mostra "Unidades disponíveis deve estar entre
0 e o total.".

- [ ] **Step 8: Rodar self-check no console**

Via `read_console_messages()`: esperar `[self-check] Lógica de negócio OK`
sem nenhum `ASSERT FAIL`.

- [ ] **Step 9: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: opção de lançamento com total/disponível de unidades no formulário"
```

---

### Task 2: Ajuste rápido (+/-) no popup e no card da lista

**Files:**
- Modify: `mapa-imoveis/index.html` (CSS, `buildPopupHtml`, `renderList`,
  função nova `adjustAvailableUnits`)

**Interfaces:**
- Consumes: `property.isLaunch`/`totalUnits`/`availableUnits` (Task 1),
  `markersById` (já existe, `MAP` section), `dbPut` (já existe,
  `INDEXEDDB` section), evento `properties-updated` (já existe, disparado
  por `filterApp().apply()`'s listener em `LIST`/`FILTERS` sections).
- Produces: `adjustAvailableUnits(id, delta)` (função global, usada via
  `onclick` inline no HTML gerado, mesmo padrão de `deleteProperty(id)` e
  `openPropertyForm(...)` já usados em `buildPopupHtml`).

- [ ] **Step 1: CSS do stepper**

Old:
```css
  .property-popup-actions { display: flex; gap: 8px; margin-top: 12px; }
```
New:
```css
  .unit-stepper { display: flex; align-items: center; gap: 8px; margin-top: 10px; font-size: 12px; color: var(--text); }
  .unit-stepper button { width: 22px; height: 22px; border-radius: 50%; background: var(--panel-2); border: 1px solid var(--line); color: var(--text); cursor: pointer; font-weight: 700; line-height: 1; padding: 0; flex-shrink: 0; }
  .unit-stepper button:hover { border-color: var(--accent); color: var(--accent); }
  .property-popup-actions { display: flex; gap: 8px; margin-top: 12px; }
```

- [ ] **Step 2: `adjustAvailableUnits`, logo após `deleteProperty` (seção MAP)**

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
  if (!confirm('Excluir este imóvel?')) return;
  await dbDelete('properties', id);
  appState.properties = appState.properties.filter(p => p.id !== id);
  removeMarker(id);
  window.dispatchEvent(new CustomEvent('properties-updated'));
}

async function adjustAvailableUnits(id, delta) {
  const property = appState.properties.find(p => p.id === id);
  if (!property || !property.isLaunch) return;
  const next = Math.max(0, Math.min(property.totalUnits, property.availableUnits + delta));
  if (next === property.availableUnits) return;
  const previous = property.availableUnits;
  property.availableUnits = next;
  try {
    await dbPut('properties', property);
  } catch (err) {
    console.error(err);
    property.availableUnits = previous;
    alert('Não foi possível salvar o ajuste de unidades.');
    return;
  }
  const marker = markersById[id];
  if (marker && marker.isPopupOpen()) marker.setPopupContent(buildPopupHtml(property));
  window.dispatchEvent(new CustomEvent('properties-updated'));
}
```

- [ ] **Step 3: `buildPopupHtml` mostra o stepper quando `isLaunch`**

Old:
```js
        ${specs ? `<div class="property-popup-specs">${specs}</div>` : ''}
        <div class="property-popup-actions">
```
New:
```js
        ${specs ? `<div class="property-popup-specs">${specs}</div>` : ''}
        ${property.isLaunch ? `<div class="unit-stepper"><button type="button" onclick="adjustAvailableUnits('${property.id}', -1)">−</button><span>${property.availableUnits} de ${property.totalUnits} disponíveis</span><button type="button" onclick="adjustAvailableUnits('${property.id}', 1)">+</button></div>` : ''}
        <div class="property-popup-actions">
```

- [ ] **Step 4: `renderList` mostra o stepper quando `isLaunch`, com `stopPropagation`**

Old:
```js
        ${specs ? `<div class="list-item-specs">${specs}</div>` : ''}
      </div>
    </div>`;
```
New:
```js
        ${specs ? `<div class="list-item-specs">${specs}</div>` : ''}
        ${p.isLaunch ? `<div class="unit-stepper"><button type="button" onclick="event.stopPropagation(); adjustAvailableUnits('${p.id}', -1)">−</button><span>${p.availableUnits} de ${p.totalUnits}</span><button type="button" onclick="event.stopPropagation(); adjustAvailableUnits('${p.id}', 1)">+</button></div>` : ''}
      </div>
    </div>`;
```

- [ ] **Step 5: Teste manual — ajuste rápido no popup e na lista**

Via Browser pane com o servidor `mapa-imoveis` rodando e o lançamento
criado na Task 1 já salvo: abrir o popup do marker
(`markersById[id].openPopup()`), confirmar que `.unit-stepper` aparece com
"40 de 40 disponíveis". Clicar 3x no botão `−` (via `javascript_tool`
disparando `.click()` no botão dentro do popup), confirmar que o texto do
popup atualiza pra "37 de 40" SEM fechar/reabrir o popup, e que
`appState.properties.find(p => p.id === id).availableUnits === 37`.
Recarregar (`navigate`) e confirmar que o valor persistiu (37, não voltou
pra 40). No card da lista, clicar no botão `+`/`−` do stepper e confirmar
via `read_console_messages`/checagem de `map.getZoom()` que o mapa NÃO
mudou de posição (ou seja, o clique não vazou pro handler do card).

- [ ] **Step 6: Commit**

```bash
git add mapa-imoveis/index.html
git commit -m "feat: ajuste rápido de unidades disponíveis no popup e na lista"
```

---

## Self-Review

**Cobertura do spec:**
- Campos novos no imóvel (`isLaunch`/`totalUnits`/`availableUnits`) →
  Task 1. ✅
- Checkbox + campos no formulário, mostrando/escondendo → Task 1. ✅
- Validação (total > 0, disponível entre 0 e total) → Task 1. ✅
- Ajuste rápido (+/-) no popup → Task 2. ✅
- Ajuste rápido (+/-) no card da lista, sem vazar clique pro card →
  Task 2. ✅
- Fora de escopo (dashboard, automação de status, página de detalhe,
  filtro por lançamento) → nenhuma task toca nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo.

**Consistência de nomes:** `isLaunch`/`totalUnits`/`availableUnits` são os
únicos nomes usados em `validateProperty`, `readPropertyForm`,
`openPropertyForm`, `adjustAvailableUnits`, `buildPopupHtml` e `renderList`
— conferido, sem variação entre tasks. `adjustAvailableUnits(id, delta)`
tem a mesma assinatura nos 2 call sites (popup e lista).

# Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Botão "💰 Simular financiamento" em cada unidade dentro do
cartão de prédio (popup de condomínio) abre a aba Financiamento com o
preço da unidade já preenchido no simulador.

**Architecture:** Toca 2 arquivos: `mapa-imoveis/index.html` (botão +
funções de navegação/URL) e `mapa-imoveis/financiamento.html` (lê o
parâmetro `vi` da URL no load). Task única — escopo pequeno o bastante
pra não precisar quebrar em mais de uma task.

**Tech Stack:** Mesmo stack, sem dependência nova. `URLSearchParams`
(API nativa) pra ler o parâmetro — nada de parsing manual de query
string.

## Global Constraints

- 2 arquivos: `mapa-imoveis/index.html`, `mapa-imoveis/financiamento.html`.
  Sem arquivo novo, sem dependência nova.
- `vi` é sempre numérico na URL — não precisa de `escapeHtml` em lugar
  nenhum deste diff (ver spec, seção Segurança).
- Clicar "💰" numa linha de unidade não pode também disparar o
  `onclick="openPropertyDetail(...)"` da linha (precisa de
  `event.stopPropagation()`).
- IDs de elementos existentes não podem mudar.

---

### Task 1: Botão "💰 Simular financiamento" na unidade do cartão de prédio

**Files:**
- Modify: `mapa-imoveis/index.html` (CSS; JS: `buildCondoPopupHtml`,
  novas `showFinancingTab`, `simulateFinancing`, `simulateFinancingFor`,
  refatora o listener de `tabFinancing`)
- Modify: `mapa-imoveis/financiamento.html` (JS: leitura de `?vi=` no
  load)

**Interfaces:**
- Produces: `showFinancingTab()` (reusada pelo listener de `tabFinancing`
  e por `simulateFinancing`). `simulateFinancing(property)`,
  `simulateFinancingFor(id)` (funções globais, chamadas pelo `onclick`
  do botão novo).
- Consumes: `priceLabelFor`'s mesma regra de preço (`dealType === 'aluguel'
  ? rentPrice : salePrice`), já replicada inline (não precisa importar
  nada, é uma linha).

- [ ] **Step 1: CSS — layout da linha de unidade com botão + info agrupada**

Old:
```css
  .condo-popup-unit { background: var(--panel-2); border: 1px solid var(--line); border-radius: 8px; padding: 6px 10px; cursor: pointer; display: flex; justify-content: space-between; align-items: center; font-size: 12px; }
  .condo-popup-unit:hover { border-color: var(--accent); }
  .condo-popup-unit span { color: var(--muted); }
```
New:
```css
  .condo-popup-unit { background: var(--panel-2); border: 1px solid var(--line); border-radius: 8px; padding: 6px 10px; display: flex; justify-content: space-between; align-items: center; gap: 8px; font-size: 12px; }
  .condo-popup-unit-info { display: flex; flex-direction: column; cursor: pointer; min-width: 0; }
  .condo-popup-unit-info:hover { color: var(--accent); }
  .condo-popup-unit span { color: var(--muted); }
  .condo-popup-unit-sim { background: var(--accent-2); color: var(--text); border: 0; border-radius: 6px; width: 24px; height: 24px; flex-shrink: 0; cursor: pointer; font-size: 12px; line-height: 1; }
```

Nota: `.condo-popup-unit:hover { border-color: var(--accent); }` foi
removido porque a linha inteira deixa de ser um único alvo de clique —
quem reage a hover agora é só `.condo-popup-unit-info` (texto, via
`color`) e o botão (cursor nativo). O fundo/borda da linha fica neutro.

- [ ] **Step 2: HTML — unidade agenciada ganha o botão, texto vira um sub-grupo clicável separado**

Old:
```js
      ? `<div class="condo-popup-units">${units.map(u => `
          <div class="condo-popup-unit" onclick="openPropertyDetail('${u.id}')">
            <strong>${priceLabelFor(u)}</strong>
            <span>${UNIT_TYPE_LABELS[u.unitType] || u.unitType}${u.rooms ? ' · ' + u.rooms + 'q' : ''}</span>
          </div>`).join('')}</div>`
```
New:
```js
      ? `<div class="condo-popup-units">${units.map(u => `
          <div class="condo-popup-unit">
            <div class="condo-popup-unit-info" onclick="openPropertyDetail('${u.id}')">
              <strong>${priceLabelFor(u)}</strong>
              <span>${UNIT_TYPE_LABELS[u.unitType] || u.unitType}${u.rooms ? ' · ' + u.rooms + 'q' : ''}</span>
            </div>
            <button type="button" class="condo-popup-unit-sim" onclick="event.stopPropagation(); simulateFinancingFor('${u.id}')" title="Simular financiamento">💰</button>
          </div>`).join('')}</div>`
```

- [ ] **Step 3: JS — `showFinancingTab`, `simulateFinancing`, `simulateFinancingFor`**

Old:
```js
function registerAgencyRequest(condoId) {
```
New:
```js
function showFinancingTab() {
  $('dashboardView').style.display = 'none';
  $('mapView').style.display = 'none';
  $('financingView').style.display = 'flex';
  $('detailView').style.display = 'none';
  $('tabFinancing').classList.add('active');
  $('tabDashboard').classList.remove('active');
  $('tabMap').classList.remove('active');
}

function simulateFinancing(property) {
  const price = property.dealType === 'aluguel' ? property.rentPrice : property.salePrice;
  $('financingFrame').src = 'financiamento.html' + (price > 0 ? '?vi=' + encodeURIComponent(price) : '');
  showFinancingTab();
}

function simulateFinancingFor(id) {
  const property = appState.properties.find(p => p.id === id);
  if (property) simulateFinancing(property);
}

function registerAgencyRequest(condoId) {
```

(O ponto exato de inserção não importa — é só perto das outras funções
de condomínio. Usei `registerAgencyRequest` como âncora porque é a
função logo antes de `upsertCondoMarker` no arquivo atual, mesma
vizinhança de `buildCondoPopupHtml`.)

- [ ] **Step 4: JS — handler de `tabFinancing` passa a usar `showFinancingTab()`**

Old:
```js
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
```
New:
```js
  $('tabFinancing').addEventListener('click', () => {
    showFinancingTab();
    if (!$('financingFrame').src) $('financingFrame').src = 'financiamento.html';
  });
```

- [ ] **Step 5: `financiamento.html` — lê `?vi=` da URL antes da primeira pintura**

Old:
```js
  document.addEventListener('keydown', e => { if (e.key === 'Escape') present.classList.remove('show'); });

  // primeira pintura
  render();
})();
```
New:
```js
  document.addEventListener('keydown', e => { if (e.key === 'Escape') present.classList.remove('show'); });

  // pré-preenchimento vindo do mapa (?vi=<preço da unidade>) — ver fase 7
  const viParam = parseFloat(new URLSearchParams(location.search).get('vi'));
  if (Number.isFinite(viParam) && viParam > 0) {
    state.VI = viParam;
    const viInput = document.querySelector('[data-field="VI"]');
    if (viInput) viInput.value = viParam.toLocaleString('pt-BR', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
  }

  // primeira pintura
  render();
})();
```

- [ ] **Step 6: Teste manual**

Via Browser pane (`preview_start({name: "mapa-imoveis"})`): criar um
condomínio com 1+ unidade agenciada com `salePrice` preenchido (fluxo
igual ao usado nos testes da fase 6). Abrir o popup do condomínio no
modo Condomínio, clicar "💰" numa unidade: confirmar que a aba vira
Financiamento (`$('tabFinancing').classList.contains('active')` true,
`$('financingView')` visível), e que
`$('financingFrame').contentDocument.querySelector('[data-field="VI"]').value`
bate com o preço da unidade formatado em pt-BR. Confirmar que o
`data-out="VI_top"` do cabeçalho da calculadora também reflete o valor
(prova que `state.VI`, não só o input visual, foi setado — se só o
`value` do input tivesse sido setado sem tocar `state`, o resto da
calculadora ficaria com VI=0 e o resumo não bateria).
Clicar "💰" numa unidade com `dealType: 'aluguel'` (sem `salePrice`,
só `rentPrice`): confirmar que usa `rentPrice`.
Clicar no texto da unidade (não no botão): confirma que ainda abre
`openPropertyDetail` normalmente, sem também ter disparado a simulação.
Simular unidade A, sem fechar nada, simular unidade B: confirmar que o
VI final é o de B.
Abrir a aba Financiamento direto, sem clicar em "💰" antes: confirmar
que carrega `financiamento.html` sem `?vi=`, campo VI vazio, igual ao
comportamento de antes desta fase.
Navegar direto pra `http://localhost:<porta>/financiamento.html?vi=abc`
(parâmetro inválido) e `?vi=-50` (negativo): confirmar que o campo VI
fica vazio/zerado, sem erro `ASSERT` nem exceção no console (este
arquivo não tem self-check formal — conferir ausência de erro via
`read_console_messages()`, com a ressalva já conhecida nesta sessão de
que esse tool às vezes retorna resultado de cache obsoleto — se o
resultado parecer suspeito, recarregar numa aba nova e checar de novo
antes de confiar).

- [ ] **Step 7: Rodar self-check do `index.html` no console**

`read_console_messages()` — todos os self-checks existentes continuam
passando sem `ASSERT FAIL` (este diff não adiciona lógica nova
não-trivial em `index.html` que justifique um assert novo — `showFinancingTab`/
`simulateFinancing` são navegação/DOM, cobertas pelo teste manual do
Step 6, mesmo padrão já usado pras outras funções de navegação do
arquivo como `closeDetailView`).

- [ ] **Step 8: Commit**

```bash
git add mapa-imoveis/index.html mapa-imoveis/financiamento.html
git commit -m "feat: botão de simular financiamento na unidade do cartão de prédio"
```

---

## Self-Review

**Cobertura do spec:**
- Botão "💰" por unidade agenciada, preço certo (venda vs aluguel) →
  Step 2 + Step 3 (`simulateFinancing`). ✅
- Troca de aba reaproveitando a lógica existente (sem duplicar) →
  Step 3 (`showFinancingTab`) + Step 4 (handler da aba usa a mesma
  função). ✅
- `financiamento.html` lê `?vi=` e preenche `state.VI` E o input visual
  (não só um dos dois) → Step 5. ✅
- Clique no botão não propaga pro `openPropertyDetail` da linha →
  Step 2 (`event.stopPropagation()`). ✅
- Fora de escopo (link por condomínio inteiro, botão em `#detailView`/
  popup de unidade avulsa, campo de cliente/endereço no simulador,
  preservar simulação em andamento) → nenhum step toca nisso. ✅

**Placeholders:** nenhum — todo Old/New é código completo.

**Consistência de nomes:** `showFinancingTab()` definida uma vez
(Step 3), usada em 2 lugares (`simulateFinancing`, Step 3; handler de
`tabFinancing`, Step 4) — mesmo texto exato nos 2 call sites, sem
duplicar a lógica de troca de view. `simulateFinancingFor` é só um
wrapper fino de busca-por-id em cima de `simulateFinancing`, para o
`onclick` do HTML não precisar inline de `appState.properties.find(...)`
(mesmo padrão já usado em `registerAgencyRequest(condoId)` recebendo só
o id, não o objeto).

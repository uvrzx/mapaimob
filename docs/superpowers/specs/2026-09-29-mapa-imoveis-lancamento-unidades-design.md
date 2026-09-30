# Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2)

## Contexto

Fase 2 de 4 do redesign combinado com o usuário. Fase 1 (design system: tema
claro, ícones por tipo, painéis recolhíveis, formulário em seções) já está
mergeada em `master` (`43dc6ca..5da0ae6`). Esta fase adiciona a opção de
marcar um imóvel como "lançamento" com um número de unidades disponíveis que
pode ser ajustado a qualquer momento, sem precisar abrir o formulário
inteiro — pedido original do usuário: *"coloque uma opção na hora da adição
dos apartamentos se for lançamento e se sim quantas unidades temos a
disposição para vender com opções de aumentar e diminuir esse valor de
unidades a qualquer instante"*.

Decisões já tomadas com o usuário (brainstorming desta fase):
- Ajuste rápido (+/-) aparece no **popup do pin e no card da lista**, além
  do formulário — não só no formulário.
- "Lançamento" fica disponível pra **qualquer tipo de imóvel**, não só
  apartamento/cobertura.
- Guarda **total de unidades** (fixo, definido na criação) + **unidades
  disponíveis** (ajustável), pra poder mostrar "12 de 40" e porque a fase 4
  (dashboard) vai precisar do total pra calcular % vendido.

## Dados novos no imóvel

3 campos novos no objeto `property` (mesma store `properties` do IndexedDB,
sem migração de schema — `dbOpen`'s `onupgradeneeded` não precisa mudar,
são só campos novos em registros novos/editados, IndexedDB não tem schema
rígido de coluna):

- `isLaunch: boolean` — default `false`.
- `totalUnits: number` — total de unidades do lançamento, fixo (editável no
  formulário se precisar corrigir, mas não tem stepper próprio).
- `availableUnits: number` — unidades disponíveis agora, `0 ≤ availableUnits
  ≤ totalUnits`, ajustável via +/- a qualquer momento.

Só têm sentido quando `isLaunch === true`; em imóveis normais ficam
`false`/`0`/`0` e não aparecem em lugar nenhum da UI.

## Formulário (seção "Básico")

Depois do select `f_unitType`, adiciona:
- Checkbox `f_isLaunch` ("É lançamento?").
- Bloco `#launchUnitsFields` (escondido por padrão, `display:none`) com 2
  inputs numéricos lado a lado: `f_totalUnits` (Total de unidades) e
  `f_availableUnits` (Disponíveis agora). Aparece quando o checkbox é
  marcado, some quando desmarcado (JS simples de show/hide, mesmo padrão já
  usado por `#newCondoFields`/`btnNewCondo`).

Nenhum stepper (+/-) dentro do formulário — são inputs numéricos normais,
iguais aos outros campos do form. O stepper rápido é só no popup/card (ver
abaixo), que é o caso de uso de "ajustar a qualquer instante" sem abrir o
formulário inteiro.

## Ajuste rápido no popup e no card da lista

Quando `property.isLaunch`, popup e card da lista mostram uma linha:

```
[ − ]  12 de 40 disponíveis  [ + ]
```

Clicar em `−`/`+` chama uma função nova `adjustAvailableUnits(id, delta)`
que:
1. Acha o imóvel em `appState.properties`.
2. Calcula o novo valor, sempre limitado a `0..totalUnits` (clique em `+`
   quando já está no total, ou em `−` quando já está em 0, não faz nada).
3. Salva no IndexedDB (`dbPut`), com o mesmo padrão de revert-on-failure já
   usado no `dragend` do marker (tenta salvar, se falhar desfaz o valor em
   memória e avisa o usuário).
4. Atualiza a UI: se o popup desse imóvel estiver aberto no momento,
   atualiza o conteúdo dele in-place (Leaflet permite recarregar o conteúdo
   de um popup já aberto); dispara `properties-updated` pra lista se
   re-renderizar (esse evento já existe e já aciona `renderList`).

No card da lista, o clique nos botões `−`/`+` precisa impedir que o clique
"vaze" pro card inteiro (que hoje tem seu próprio `click` pra centralizar o
mapa no imóvel) — usa `event.stopPropagation()` no handler inline.

## Validação

`validateProperty` ganha 2 checagens novas, só quando `isLaunch`:
- `totalUnits > 0` — "Lançamento precisa de total de unidades maior que
  zero."
- `0 ≤ availableUnits ≤ totalUnits` — "Unidades disponíveis deve estar
  entre 0 e o total."

Essas checagens são condicionadas a `property.isLaunch` sendo truthy, então
não mudam o resultado do self-check existente (`badProp` no
`selfCheckLogic()` não define `isLaunch`, continua com exatamente 6 erros).

## Fora de escopo

- Dashboard/relatório de vendas (fase 4) — vai LER `totalUnits`/
  `availableUnits` depois, não é construído aqui.
- Qualquer automação (ex.: `availableUnits === 0` mudar `status` pra
  "vendido" sozinho) — o corretor controla os dois campos manualmente.
- Página de detalhe do imóvel (fase 3).
- Alterar a lógica de filtros existente (`matchesFilters`) — lançamento não
  é um filtro nesta fase.

## Teste manual

Via servidor local (`mapa-imoveis` em `.claude/launch.json`):
- Criar um imóvel marcando "É lançamento?", com total 40 e disponíveis 40 —
  formulário aceita, campos de unidade aparecem/somem ao marcar/desmarcar
  o checkbox.
- Abrir o popup do pin: ver "40 de 40 disponíveis" + botões `−`/`+`.
  Clicar `−` 3 vezes → "37 de 40", popup atualizado sem fechar/reabrir.
- Card da lista mostra o mesmo stepper; clicar nele NÃO deve fazer o mapa
  voar pro imóvel (stopPropagation funcionando).
- Recarregar a página (F5) — valor ajustado persiste (leu do IndexedDB).
- Tentar salvar um lançamento com `availableUnits > totalUnits` no
  formulário → erro de validação aparece.
- Self-checks (`console.assert`) continuam passando sem alteração.

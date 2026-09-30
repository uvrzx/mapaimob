# Mapa de Imóveis — Dashboard (Fase 4)

## Contexto

Última das 4 fases combinadas. Fases 1-3 (design system, lançamento/
unidades, página de detalhe) já mergeadas em `master`. Referência: imagem
3 ("Orbix Studio" dashboard) — adaptada pros nossos dados reais, sem
inventar métrica que não existe (sem "views", sem meta de vendas, sem
avaliação). Decisões do brainstorming: gráfico em SVG feito à mão (sem lib
nova), sem mini-mapa (não duplicar o Leaflet da aba Mapa), tabela sem
coluna Views.

## Dashboard vira a aba padrão

Header ganha 3 abas: **Dashboard** (nova, ativa por padrão), Mapa,
Financiamento. `#mapView` passa a começar escondido (`display:none`
inline), `#dashboardView` (nova) começa visível. Os handlers de clique nas
3 abas passam a esconder as outras 3 views (dashboard/mapa/financiamento/
detalhe) e mostrar só a escolhida — mesmo padrão já usado, só com mais uma
opção.

**Efeito colateral que precisa de fix**: a página de detalhe (fase 3)
hoje sempre volta pro Mapa (`btnDetailBack` e o "Excluir" do rodapé têm
essa lógica hardcoded). Como agora dá pra abrir o detalhe a partir do
Dashboard também (imóvel em destaque, tabela de recentes), o "← Voltar"
precisa lembrar de onde veio. `openPropertyDetail` passa a gravar
`dataset.returnTo` ('dashboard' ou 'map'), e um `closeDetailView()` novo
(substituindo a lógica duplicada que já existia em 2 lugares) decide pra
onde voltar.

## Conteúdo (adaptado da imagem 3 pros nossos dados)

- **KPIs**: total de imóveis, disponíveis, reservados, vendidos, alugados
  — contagens reais de `appState.properties`.
- **Balanço de vendas**: seletor Mensal/Trimestral/Semestral/Anual = janela
  móvel de 30/90/180/365 dias a partir de agora (**simplificação
  deliberada**: não temos campo de "data da venda", uso `updatedAt` como
  proxy — o valor só é preciso se o corretor não editar o imóvel por outro
  motivo depois de marcar como vendido/alugado). Mostra total R$ vendido e
  R$ alugado no período, com 2 barras SVG/CSS comparando os valores.
- **Unidades de lançamento**: soma `totalUnits`/`availableUnits` de todos
  os imóveis com `isLaunch`, com barra de progresso (vendidas vs total).
- **Imóvel em destaque**: o de `createdAt` mais recente — foto, preço,
  specs, botão "Ver detalhes".
- **Tabela de recentes**: últimos 10 por `createdAt` — Imóvel/Tipo/
  Corretor/Preço/Status/Ação, linha não é clicável inteira (só o botão
  "Ver", evita reintroduzir `stopPropagation` sem necessidade).

## Segurança: não repetir o XSS da fase 3

A tabela de recentes mostra `label`/`agentResponsible` (texto livre do
corretor). A fase 3 corrigiu um XSS armazenado em `notes` que ia direto
pro `innerHTML`. Pra não reintroduzir a mesma classe de bug no código
novo desta fase, adiciono um `escapeHtml()` (helper de 3 linhas,
`textContent` num `<div>` descartável) e uso nos 2 campos de texto livre
da tabela. **Não** estou corrigindo o `innerHTML` de endereço/apelido que
já existe no popup/lista/detalhe — isso é dívida técnica anterior a esta
fase, fora de escopo aqui (mesma decisão que a revisão da fase 3 já tinha
tomado).

## Fora de escopo

- Mini-mapa (decisão do brainstorming).
- Qualquer campo novo no imóvel.
- Corrigir o `innerHTML` de endereço/apelido pré-existente no popup/lista/
  detalhe (dívida técnica anterior, não desta fase).
- Gráficos de série temporal dia-a-dia (não temos granularidade de data de
  venda pra isso além do proxy de `updatedAt`).

## Teste manual

- Abrir o app: Dashboard é a aba visível, com os 5 KPIs e o resto do
  conteúdo.
- Trocar o período do balanço de vendas: os totais recalculam, e a seleção
  não reseta se outro imóvel for salvo/editado em paralelo.
- Card de lançamento só aparece se existir pelo menos 1 imóvel com
  `isLaunch`.
- Imóvel em destaque é sempre o mais recente (criar um novo, confirmar que
  ele vira o destaque).
- Abrir detalhe a partir do destaque/tabela → "← Voltar" volta pro
  Dashboard (não pro Mapa). Abrir detalhe a partir do popup/lista do mapa
  → "← Voltar" volta pro Mapa.
- Tabela: payload tipo `<img src=x onerror=...>` num apelido ou nome de
  corretor renderiza como texto, sem executar nada.
- Abas Dashboard/Mapa/Financiamento continuam alternando certo, mapa
  ainda dimensiona certo ao entrar na aba Mapa (`invalidateSize`).
- Self-checks continuam passando.

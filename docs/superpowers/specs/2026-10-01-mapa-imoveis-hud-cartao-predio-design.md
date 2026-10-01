# Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6)

## Contexto

Cliente mandou vídeo (WhatsApp, 2:44, áudio transcrito + 55 frames
analisados) mostrando protótipo Figma de 3 páginas ("Hud Mapa", "Cartão
Prédio", "Cartão Unidade") com a visão original do produto. Catalogado em
conversa, com 2 decisões já confirmadas pelo cliente:

- Paleta do cartão de popup segue a paleta clara do app (branco/azul/
  amarelo), não o navy/vermelho do protótipo — protótipo é referência de
  **estrutura**, não de cor.
- Botão "Agenciar" (que no vídeo dispara automação pro ClickUp) fica como
  ação local por enquanto — webhook real fica pra depois.

## Decisão de escopo: o que entra nesta fase vs. o que fica pra depois

O vídeo mostra 3 coisas distintas. Nem tudo tem o mesmo tamanho de
problema técnico:

1. **Toggle Unidade/Condomínio + pin com foto/% + cartão de prédio** — dado
   que já existe no app (properties + condos), só precisa de modelagem
   nova e UI nova. **Entra nesta fase.**
2. **Cartão de unidade** (clicar numa unidade agenciada dentro do cartão
   de prédio) — o cliente disse no vídeo "isso daqui eu não fiz ainda".
   O conteúdo que ele descreve (specs da unidade, dormitórios, vagas)
   **já existe** na página de detalhe (`#detailView`, fase 3). Reaproveito
   em vez de construir um popup novo do zero (ponytail regra 2: já existe
   no código, usa). **Entra nesta fase, sem componente novo.**
3. **Legenda de amenidades como toggle de camada no mapa** (mercado,
   farmácia, hospital etc. aparecendo/sumindo do mapa) — isso exige uma
   fonte de dados de POI real que o app não tem hoje (teria que vir de
   Overpass API ou similar, é integração nova e maior). **Fora desta
   fase**, vira fase própria depois.
4. **Webhook ClickUp no botão Agenciar** — decisão do cliente, fica pra
   depois.

## Modelo de dados: condomínio vira "prédio" no mapa

Hoje `condo` é só um conjunto de amenidades + nome + taxa, ligado a
`property.condoId`, sem posição própria no mapa. Pra virar o "prédio"
clicável do vídeo, `condo` ganha:

- `address` (`street`, `number`, `neighborhood`, `city`) — igual ao
  formato já usado em `property.address`.
- `coordinates` (`[lat, lng]` ou `null`) — posição do pin no mapa,
  preenchida via geocodificação (reaproveita o mesmo fluxo Nominatim já
  usado no formulário de imóvel).
- `description` (texto livre) — "sobre o prédio" do vídeo (construtora,
  ano etc.). Um campo só, texto livre — não separo em
  `builderName`/`yearBuilt` estruturados, o cliente escreve como quiser
  (ponytail: não estruturar o que não precisa).
- `photos` (array de Blob, mesmo padrão de `property.photos`).

Continua criado só pelo formulário de imóvel (botão "+ Novo condomínio"),
sem tela de edição própria — fora de escopo editar condomínio depois de
criado (mesma limitação que já existe hoje).

## HUD flutuante sobre o mapa

Barra flutuante centralizada no topo do `#mapWrap` (não é mais um 2º
formulário de filtro — é um controle de modo + atalho):

- Segmented control **Unidade | Condomínio** — alterna `appState.mapMode`.
  Controla **qual camada de marcador aparece no mapa**: unidades
  (comportamento atual, inalterado) ou prédios/condomínios (novo).
- Botão **Filtrar** — atalho que abre a sidebar de filtros já existente
  (não é um formulário novo, só chama o toggle do `btnCollapseSidebar`).

A sidebar de filtros continua sendo uma só. No modo Condomínio, nem todo
filtro faz sentido pro nível de prédio (quartos/banheiros/preço são da
unidade) — o vídeo confirma isso ("filtrando, só aparece no mapa os
residenciais que batem com esse filtro... não tem nada a ver com
unidade"). Os critérios que **são** de nível de prédio já existem no
filtro atual — bairro e amenidades — e são os únicos aplicados a
condomínios no modo Condomínio. Os demais critérios (preço, quartos etc.)
são ignorados nesse modo, sem UI condicional nova.

## Pin de condomínio

Substitui o pin SVG padrão nesse modo por: foto do prédio (primeira foto)
em círculo + badge colorido no topo com o percentual de preenchimento.

**Preenchimento** = % de unidades daquele condomínio com status
`vendido` ou `alugado`, sobre o total de unidades vinculadas
(`property.condoId`). Condomínio sem nenhuma unidade vinculada não
aparece no mapa (não tem o que mostrar).

**Cor do badge** — simplificação deliberada em 3 faixas (terços):
abaixo de 34% vermelho (`--warn-fg`), 34–66% amarelo (`--accent-2`), 67%+
verde (`--ok-fg`). Não é uma regra que o cliente validou explicitamente
no vídeo — é a leitura mais direta do "vermelho baixo / amarelo médio"
que ele mostrou. Ajustável depois se o cliente quiser outra faixa.

## Cartão de prédio (popup ao clicar o pin)

Mesma estrutura do protótipo, paleta clara do app:

- Galeria de fotos (carrossel simples, setas prev/next).
- Nome do prédio + endereço.
- "Sobre o prédio" (texto livre, se preenchido).
- "Condomínio" — lista das amenidades ativas (reaproveita
  `AMENITY_LABELS`/`AMENITY_CHECK_ICON`, mesmo padrão já usado na página
  de detalhe).
- "Unidades agenciadas" — lista das `property` vinculadas a esse
  `condoId`. Clicar numa unidade abre a página de detalhe já existente
  (`openPropertyDetail`), não um popup novo (ver decisão de escopo acima).
  Sem unidades vinculadas, mostra mensagem vazia.
- Botão "Agenciar" — grava `condo.agencyRequestedAt` localmente
  (IndexedDB) e confirma com um alert. Sem chamada externa.

## Segurança

`condo.name`, `condo.description` e os campos de `condo.address` são
texto livre do usuário renderizado em HTML novo — passam por
`escapeHtml()` (já existe, criado na fase 4) no popup de prédio. Mesmo
cuidado que já existe pra tabela de recentes do dashboard.

## Fora de escopo

- Legenda/toggle de amenidades reais no mapa (mercado, farmácia,
  hospital) — precisa de fonte de POI externa, vira fase própria.
- Webhook ClickUp no botão "Agenciar" — decisão do cliente, fica pra
  depois.
- Popup/cartão dedicado pra unidade — reaproveita `#detailView` existente.
- Edição de condomínio depois de criado (mesma limitação de hoje).
- Seção "Últimas vendas" mencionada no vídeo pro cartão de unidade — o
  cliente não mostrou o conteúdo (nem ele sabia ainda), e como reaproveito
  `#detailView` em vez de um cartão novo, não force uma seção que não foi
  especificada. Pode virar um acréscimo em `#detailView` depois, se o
  cliente pedir.

## Teste manual

- Criar condomínio novo (pelo form de imóvel) preenchendo endereço, foto
  e descrição; confirmar que salva certo (`dbGet('condos', id)` tem os
  campos).
- Vincular 2-3 imóveis a esse condomínio com status variados (vendido,
  disponível); trocar o toggle do HUD pra "Condomínio"; confirmar que o
  pin aparece com a foto e o % certo, e que os pins de unidade somem do
  mapa.
- Clicar o pin do condomínio: popup mostra galeria (se mais de 1 foto, as
  setas funcionam), nome, endereço, "sobre o prédio" (se preenchido),
  amenidades ativas, lista de unidades agenciadas.
- Clicar numa unidade agenciada dentro do popup: abre a página de
  detalhe certa.
- Condomínio sem unidade vinculada: não aparece no mapa.
- Clicar "Agenciar": alert de confirmação, `condo.agencyRequestedAt`
  gravado no IndexedDB.
- Payload tipo `<img src=x onerror=...>` no nome/descrição do condomínio
  renderiza como texto no popup, sem executar nada.
- Trocar o toggle de volta pra "Unidade": comportamento do mapa volta ao
  que já existia antes desta fase, sem regressão.
- Filtro por bairro/amenidades no modo Condomínio afeta quais prédios
  aparecem; filtro de preço/quartos (inexistente nesse nível) é
  ignorado sem quebrar nada.
- Self-checks (`[self-check] Lógica de negócio OK`, `[self-check]
  IndexedDB OK`, `[self-check] Condomínio no mapa OK`) continuam
  passando sem `ASSERT FAIL`.

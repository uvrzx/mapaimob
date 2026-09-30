# Mapa de Imóveis — Design System Overhaul (Fase 1)

## Contexto

`mapa-imoveis/index.html` está no tema escuro preto/azul/amarelo (commit `a9908b6`).
O usuário trouxe 3 imagens de referência para uma reforma maior, dividida em 4 fases:

1. **Design system** (esta fase) — retema claro, sidebar + cartão flutuante do mapa
   ganham controle de esconder/mostrar, painel de adicionar imóvel reorganizado,
   ícones do pin diferenciados por tipo de imóvel.
2. Lançamento / inventário de unidades (fase futura).
3. Página de detalhe do imóvel (fase futura, referência: imagem 2 estilo Airbnb).
4. Dashboard como aba inicial (fase futura, referência: imagem 3).

Esta fase cobre **apenas** o item 1. Fases 2-4 têm specs próprios quando chegarem.

## Referência visual (imagem 1 — "uphome")

Dashboard de busca de imóvel: fundo cinza claro, cards brancos, texto escuro,
accent azul/indigo (botão "Search", preço em destaque), sidebar de filtros à
esquerda com checkboxes, mapa claro à direita com pins pequenos e um card de
detalhe flutuante sobre o pin selecionado. Fonte geométrica sans (tipo Inter).

## Tokens de tema

Substituem totalmente o `:root` atual (sem dark mode, decisão do usuário):

```css
--bg: #F5F6FA;        /* fundo da página */
--panel: #FFFFFF;     /* cards, sidebar, painéis */
--panel-2: #F0F1F6;   /* inputs, chips inativos */
--line: #E6E8F0;      /* bordas */
--text: #14161F;      /* texto principal */
--muted: #8A8F9C;     /* texto secundário / labels */
--accent: #4B5FEE;    /* azul/indigo — botões primários, preço, seleção, CTA */
/* --accent-2 (amarelo) é removido: a referência é mono-accent azul/indigo.
   Todo elemento que hoje usa --accent-2 (CTA "+ Adicionar imóvel", header
   button ativo) passa a usar --accent. */
--ok-bg/--ok-fg/--ok-line e --warn-*: recalibrar para tons pastel claros
             (fundo bem claro, texto saturado) em vez dos tons escuros atuais,
             mantendo os mesmos nomes de variável (usados por status/erros).
--radius: 16px (era 10px);
--sans: 'Inter', system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
```

Sombras: usar `0 8px 24px rgba(20,24,50,.06)` (soft, baseado em preto com baixa
opacidade) em vez das sombras pretas mais duras atuais (`rgba(0,0,0,.4)` etc).

Fonte: carregar Inter via Google Fonts `<link>` (mesma origem já permitida —
já carregamos CDN externo pra Leaflet/Alpine).

## Sidebar + cartão flutuante do mapa

- Sidebar: mantém a mecânica de collapse já existente (`#sidebar.collapsed`,
  `#btnCollapseSidebar`) — só reskin de cor/tipografia/sombra.
- **Novo:** `#mapFilterCard` ganha um botão de recolher equivalente (mesmo
  padrão visual do botão da sidebar: círculo com seta, grudado na borda do
  card), escondendo o card inteiro (`display:none` ou `translateX` para fora
  da view) e mostrando um botão pequeno para reabrir.
- Chips/seg-groups/steppers continuam como interação (decisão já tomada em
  sessão anterior de sair de checkbox/select pra chip) — só troca de cor.

## Painel de adicionar/editar imóvel (`#propertyForm`)

Reorganizar os campos existentes (nenhum campo novo nesta fase) em seções com
mini-headers, na ordem atual:

1. **Básico** — apelido, status, negócio, tipo de imóvel
2. **Preço e área** — preços, áreas construída/lote
3. **Características** — quartos, suítes, banheiros, vagas, andar, mobiliado, pet
4. **Endereço** — rua, número, bairro, cidade, complemento, botão geocodificar
5. **Condomínio** — corretor, select de condomínio, campos de novo condomínio
6. **Fotos e observações** — notes, upload de fotos

Botão Salvar/Cancelar fica **sticky no rodapé** do painel (não precisa rolar
até o fim pra salvar). Reskin de inputs/labels/selects para o tema claro.

## Ícones do pin por tipo de imóvel

Hoje `buildMarkerIcon(status)` sempre embute `ICONS.house` no pin, não importa
o `unitType` do imóvel. Passa a ser `buildMarkerIcon(status, unitType)` e
escolher o ícone certo do mapa `typeIcons` (já existe em `filterApp()`, será
promovido para constante global reaproveitável por `buildMarkerIcon` e pelos
popups/cards/lista, que hoje usam sempre `ICONS.house` como fallback).

Ícones novos a desenhar (mesmo estilo filled, sem `fill` explícito no SVG,
herdando do `<g fill>` do pai — padrão já estabelecido em `ICONS`):

- `sobrado`: casa de dois pavimentos (silhueta com duas fileiras de janela).
- `cobertura`: prédio com terraço/varanda no topo (diferente de `building`).

Resultado: os 6 tipos (casa, apartamento, comercial, terreno, sobrado,
cobertura) ficam todos visualmente distintos no pin, no popup, no card da
lista e no grid de tipo do formulário/filtro.

## Fora de escopo

- Tiles do mapa (OSM) — não mexe, é dado de terceiro.
- Qualquer campo novo no formulário (lançamento/unidades — fase 2).
- Página de detalhe do imóvel (fase 3).
- Dashboard (fase 4).
- Dark mode / toggle de tema.

## Teste manual

Após implementado, verificar via servidor local (`mapa-imoveis` em
`.claude/launch.json`):
- Tema claro aplicado em toda a shell (header, sidebar, cartão do mapa,
  popup, lista, formulário), sem resíduo de cor escura.
- Sidebar e cartão flutuante do mapa escondem/mostram independentemente.
- Pin de cada um dos 6 tipos renderiza ícone diferente (checar visualmente
  ou via snapshot do SVG gerado por `buildMarkerIcon`).
- Formulário abre com seções visíveis e botão Salvar sempre visível ao rolar.
- Self-checks de `console.assert` (IndexedDB + lógica) continuam passando.

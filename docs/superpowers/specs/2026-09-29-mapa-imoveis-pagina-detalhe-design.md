# Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3)

## Contexto

Fase 3 de 4. Fases 1 (design system) e 2 (lançamento/unidades) já mergeadas
em `master`. Referência visual: imagem 2 (página de detalhe estilo
Airbnb — título, localização, preço, botão de ação, specs com ícone,
descrição, comodidades, galeria de fotos à direita). Não temos avaliação/
review de hóspedes (não é short-term rental) nem disponibilidade por data —
essas partes da referência são adaptadas ou removidas.

Decisões do brainstorming desta fase:
- Gatilho: foto/preço do card da lista abre o detalhe direto (com
  `stopPropagation`, sem afetar o clique no resto do card que já voa o mapa);
  popup ganha um 3º botão "Ver detalhes" ao lado de Editar/Excluir.
- Sem seção de reviews. No lugar do "Check Availability" da referência,
  botão "Editar imóvel" no mesmo destaque visual.
- Sem router/backend: 3ª "view" no mesmo padrão de `#mapView`/
  `#financingView` (show/hide via `style.display`), com botão "← Voltar".

## Layout (baseado na imagem 2, adaptado)

Duas colunas (`.detail-grid`, grid `1.1fr 1fr`):

**Esquerda:**
- Título (apelido do imóvel, ou "Tipo em Bairro" se não tiver apelido).
- Endereço completo.
- Linha de preço + badge de status + botão "Editar imóvel".
- Specs (quartos, suítes, banheiros, vagas, área) com os ícones já usados
  no popup/lista.
- Descrição (`notes`), truncada com "Mostrar mais/menos" se longa (>220
  caracteres).
- Comodidades do condomínio (se `condoId` setado): lista com ícone de
  check genérico + nome de cada amenidade marcada — **simplificação
  deliberada**: 1 ícone de check reaproveitado pra todas as amenidades, em
  vez de 8 ícones únicos (piscina/academia/etc.) como a referência sugere;
  ganho visual marginal não justifica desenhar 8 SVGs novos nesta fase.
- Se `isLaunch`: mesmo stepper +/- já usado no popup/lista.
- Rodapé: link "Excluir imóvel" (ghost, baixo destaque).

**Direita:** galeria — 1ª foto grande (hero), até 2 fotos seguintes menores
lado a lado abaixo. Sem foto: mostra o ícone do tipo de imóvel (mesmo
placeholder já usado no popup/lista quando não há foto).

## Navegação

`#detailView`, sibling de `#financingView`, escondido por padrão. Abrir
chama `openPropertyDetail(id)` que popula os campos e troca os `display`
(esconde `#mapView`, mostra `#detailView`). "← Voltar" faz o inverso. As
abas Mapa/Financiamento do header continuam existindo; clicar em "Mapa"
enquanto no detalhe também fecha o detalhe (robustez, não é o fluxo
principal).

## Reuso de código

- `priceLabelFor(property)` — extrai a lógica de preço hoje duplicada em
  `buildPopupHtml` e `renderList` (2 cópias idênticas) pra uma função só,
  reaproveitada também pela página de detalhe (3º consumidor).
- `AMENITY_LABELS` — promove o array `amenityOptions` de dentro do
  `filterApp()` (Alpine) pra uma constante global, do mesmo jeito que
  `TYPE_ICONS` foi promovido na fase 1 — a página de detalhe é código
  vanilla, não tem acesso ao escopo do componente Alpine.
- `deleteProperty` passa a retornar `true`/`false` (deletou ou cancelou) —
  mudança compatível com o único outro caller (popup, que ignora o
  retorno) — necessário pra saber se deve voltar pro mapa depois de
  excluir pela página de detalhe.

## Fora de escopo

- Dashboard (fase 4).
- Qualquer dado novo no imóvel (fotos, notas, endereço — tudo já existe).
- Tornar a página de detalhe uma URL compartilhável (sem backend, sem
  necessidade nesta fase — ferramenta interna, uso é sempre a partir do
  mapa já aberto).
- Redesenhar o popup/lista além de adicionar os gatilhos de abrir o
  detalhe.

## Teste manual

- Clicar na foto/preço de um card da lista abre o detalhe; clicar no
  resto do card continua voando o mapa (sem abrir o detalhe por engano).
- Botão "Ver detalhes" no popup abre a mesma página.
- Detalhe mostra: título, endereço, preço, status, specs, fotos (hero +
  miniaturas), descrição com toggle se longa, comodidades (se tiver
  condomínio com amenidades), stepper de unidades (se lançamento).
- "Editar imóvel" abre o formulário de sempre, pré-preenchido.
- "Excluir imóvel" confirma, exclui, e volta pro mapa automaticamente.
- "← Voltar" sem excluir volta pro mapa sem perder nada.
- Self-checks continuam passando.

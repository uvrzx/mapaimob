# Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8)

## Contexto

Item que ficou explicitamente fora de escopo na fase 6 ("Fora de escopo:
legenda/toggle de amenidades reais no mapa... precisa de fonte de POI
externa, vira fase própria"). É o último pedaço do vídeo de referência
original que faltava: legenda no canto do mapa com mercado/padaria/
escola/academia/hospital/farmácia/restaurante/parque, cada um com
ícone colorido e toggle pra ligar/desligar aquele tipo no mapa.

## Fonte de dados — confirmada com o usuário

Overpass API (OpenStreetMap) — gratuita, sem chave, mesma fonte que o
app já usa pro mapa base e pra geocodificação (Nominatim). Endpoint:
`https://overpass-api.de/api/interpreter` (o oficial). No teste inicial
que precedeu o plano ele deu um 504 transitório e o espelho
`overpass.kumi.systems` respondeu certo — mas na implementação real
esse espelho se mostrou instável (fetch travando/timeout), enquanto o
endpoint oficial respondeu rápido e certo (123 pontos reais em
Florianópolis, 8 categorias). Trocado de volta pro oficial depois desse
teste ao vivo.

**Limitação assumida**: cobertura depende de quão mapeada a área está
no OpenStreetMap — boa em centros urbanos, pode faltar em áreas menores
ou rurais. Não é bug, é a fonte de dados.

## Categorias e tags OSM

| Legenda | Tag OSM |
|---|---|
| Mercados | `shop=supermarket` |
| Padarias | `shop=bakery` |
| Escolas | `amenity=school` |
| Academias | `leisure=fitness_centre` |
| Hospitais | `amenity=hospital` |
| Farmácias | `amenity=pharmacy` |
| Restaurantes | `amenity=restaurant` |
| Parques | `leisure=park` |

Uma única lista (`AMENITY_POI_CATEGORIES`) no JS guia tudo: a query
Overpass, a classificação do resultado por categoria, e a legenda
renderizada — sem duplicar a lista de categorias em 3 lugares.

Parques no OSM costumam ser mapeados como área (`way`), não só ponto
(`node`) — a query busca os dois tipos e usa `out center` pra pegar um
ponto representativo de áreas. Sem isso, "Parques" ficaria quase vazio
na prática (seria só os raros parques mapeados como ponto).

## Zoom mínimo e cache por área

Só busca/mostra POIs com zoom ≥ 14 (zona de bairro) — zoom inicial do
app é 13, então a legenda começa sem pontos até o usuário dar zoom.
Isso evita consulta de área gigante (cidade inteira) no Overpass, que
timeout/sobrecarrega o servidor público e devolveria uma quantidade de
pontos sem utilidade visual.

Cache simples por área: guarda os bounds da última busca (com uma
margem de 30%) e só busca de novo quando o usuário sai dessa área —
evita bater no Overpass a cada pixel de pan, debounce de 600ms no
`moveend`/`zoomend`.

## Cores — paleta própria, fora do design system de UI

8 categorias precisam de 8 cores distintas pra diferenciação visual —
os 4 tokens de cor da UI (`--accent`, `--accent-2`, `--ok-fg`,
`--warn-fg`) não dão conta disso e não fazem sentido semântico aqui
(não é status de imóvel, é categoria de POI). Uso 8 hex fixos,
documentados como paleta própria no `design.md`, sem significado de
estado — só diferenciação.

## Renderização

Pontos pequenos (`L.circleMarker`, nativo do Leaflet, leve — nada de
ícone SVG customizado como os pins de imóvel/condomínio, que são
visualmente mais pesados e não fazem sentido pra uma camada de
contexto/referência). Tooltip com o nome do local no hover (dado vem
do OSM, passa por `escapeHtml` antes de virar tooltip — mesma cautela
de sempre com texto de fonte externa). Sem popup/clique — são pontos
de referência visual, não têm card nem ação (igual ao vídeo, que só
mostra "pensa que isso aqui é um restaurante", sem interação além do
toggle da legenda).

## Legenda

Painel flutuante no canto inferior esquerdo do mapa (não colide com
zoom control, que é inferior direito, nem com o HUD do topo). Lista das
8 categorias, ponto colorido + nome, clicável — clicar liga/desliga só
aquela categoria no mapa (não refaz a busca, só mostra/esconde a camada
já carregada). Todas ligadas por padrão.

Funciona nos dois modos do mapa (Unidade e Condomínio) — é uma camada
de contexto independente do toggle de fase 6, sem acoplamento entre os
dois sistemas.

## Erro de rede

Se o Overpass falhar (timeout, 504, sem internet): loga no console e
não mexe nos pontos que já estavam no mapa — sem banner de erro, sem
retry automático. É uma camada auxiliar/decorativa, não crítica pro
uso do app (diferente de salvar um imóvel, por exemplo).

## Fora de escopo

- Mais categorias além das 8 do vídeo.
- Clique/popup nos pontos de amenidade (só tooltip no hover).
- Configurar o raio/zoom mínimo pela UI (fixo no código, 14).
- Cache entre sessões (localStorage/IndexedDB) — só em memória durante
  a sessão atual.
- Fallback pra um segundo provedor de POI se o Overpass cair — se isso
  virar problema recorrente, é decisão de trocar o endpoint (uma
  constante), não de built-in failover automático.

## Teste manual

- Zoom abaixo de 14: legenda aparece mas nenhum ponto no mapa (sem
  chamada de rede ainda, ou chamada que não renderiza nada).
- Zoom pra 15+ numa área urbana: pontos aparecem, cores batendo com a
  legenda.
- Hover num ponto: tooltip com o nome do lugar (quando o OSM tem
  `name` na tag).
- Clicar numa categoria na legenda (ex: Farmácias): só os pontos de
  farmácia somem do mapa, o resto continua.
- Clicar de novo: voltam, sem nova chamada de rede (usa os dados já
  buscados).
- Pan pequeno dentro da mesma área: não dispara nova busca (cache).
- Pan grande pra fora da área coberta: dispara nova busca depois do
  debounce.
- Trocar entre modo Unidade e Condomínio (HUD da fase 6): pontos de
  amenidade continuam no mapa sem interferência.
- Simular falha de rede (endpoint errado temporariamente, ou offline):
  console mostra o erro, app não trava, pontos antigos continuam
  visíveis.
- Nome de POI com caracteres HTML (se existir no dado real ou em teste
  sintético): tooltip mostra texto literal, sem executar nada.

# Mapa de Imóveis (uso interno) — Design

## Contexto

Ferramenta interna pra imobiliária visualizar imóveis num mapa. **Não é um agregador/puxador de anúncios** — os próprios corretores cadastram cada imóvel manualmente, colocando um pin no local exato no mapa. O uso principal é achar rápido "qual imóvel bate com esse cliente" filtrando por características do imóvel e do condomínio, à medida que a base cresce.

Uma busca no site da Mappo (concorrente) foi usada só como referência de quais campos um anúncio de imóvel costuma ter (preço, área, quartos, banheiros, vagas, tipo, endereço, corretor, fotos) — nenhum dado de lá é usado, é só inspiração de schema.

Padrão de código de referência: `corretor-gv/calculadora-proposta.html` — HTML único, CSS com custom properties (dark theme: `--bg`, `--panel`, `--accent`...), sem Tailwind, sem build. Este projeto segue o mesmo padrão visual/estrutural, com uma exceção pontual (ver Arquitetura).

**Este é um MVP.** Escopo enxuto de propósito — cresce depois conforme uso real.

## Escopo

### Dentro do escopo
- Mapa (Leaflet + tiles OpenStreetMap, sem chave de API, sem provedor pago) centrado em Florianópolis continental (São José/Estreito/Biguaçu), pan/zoom livre.
- Cadastro de imóvel via clique no mapa → pin no local exato → formulário lateral.
- Pin arrastável pra reposicionar sem recriar cadastro.
- Pin colorido por status (verde=disponível, amarelo=reservado, vermelho=vendido/alugado).
- Cadastro de condomínio reutilizável (amenidades), vinculável a vários imóveis do mesmo prédio.
- Fotos por imóvel, redimensionadas no cliente antes de salvar (max ~1600px largura).
- Painel de filtros sempre visível, com contagem de resultados ao vivo, sincronizado com mapa e lista lateral.
- Persistência 100% local via IndexedDB nativo (sem backend, sem login — uso individual/uma máquina por enquanto).
- Exportar/importar `.json` de backup (proteção contra perda de dados do navegador).
- Auto-check de sanidade da lógica de filtro (assert simples no load, sem framework de teste).

### Fora do escopo (por ora)
- Multi-usuário / backend compartilhado / sincronização entre máquinas.
- Autenticação e permissões.
- Qualquer scraping/importação automática de anúncios externos.
- Clustering de pins (adicionar depois só se a densidade virar problema visual).
- Provedor de mapa pago (Google Maps).

## Arquitetura

Arquivo único `mapa-imoveis/index.html`, sem build.

- **CSS**: custom properties dark theme, mesmo padrão do `corretor-gv` (`--bg`, `--panel`, `--panel-2`, `--line`, `--text`, `--muted`, `--accent`, cores de status ok/warn reaproveitadas pro esquema disponível/vendido).
- **Mapa**: Leaflet.js via CDN + tiles OpenStreetMap. Única dependência externa "pesada" — sem alternativa nativa viável pra mapa interativo com tiles.
- **Estado geral / CRUD / fotos / IndexedDB**: JS vanilla, mesmo padrão do `corretor-gv` e do `rpg-ficha` — variáveis de estado simples + função `render()` que atualiza o DOM. IndexedDB acessado direto pela API nativa (sem wrapper), com um pequeno módulo de funções `dbGet/dbPut/dbDelete/dbGetAll` isolando os callbacks.
- **Painel de filtros + lista lateral**: Alpine.js via CDN, escopo isolado só nessa área (é a parte com mais estado cruzado — filtros afetam contagem, lista e visibilidade de pins ao mesmo tempo; reatividade declarativa evita sincronização manual espalhada). O restante do arquivo (mapa, formulário de cadastro, fotos) permanece vanilla.
- **Sem roteamento, sem build step, sem npm.**

## Modelo de dados

**Imóvel** (`store: properties`):
```
id, label (apelido opcional),
status: disponível | reservado | vendido | alugado,
dealType: venda | aluguel | venda_e_aluguel,
unitType: casa | apartamento | comercial | terreno | cobertura | sobrado,
salePrice, rentPrice,
constructedArea, landArea,
rooms, suites, bathrooms, parkingSpaces, floor,
furnished: sim | não | parcial,
petsAllowed: bool,
address: { street, number, neighborhood, city, complement },
coordinates: [lat, lng],
agentResponsible, notes,
condoId (opcional),
photos: [Blob],
createdAt, updatedAt
```

**Condomínio** (`store: condos`, opcional, reutilizável entre imóveis):
```
id, name,
pool, gym, partyRoom, playground, petArea, security24h, elevator, gatedCommunity,
monthlyFee
```

Ao cadastrar um imóvel, o corretor escolhe um condomínio já existente (autocomplete) ou cria um novo ali mesmo — evita redigitar amenidades pra cada unidade do mesmo prédio.

## Filtros (núcleo da UX)

Painel lateral fixo, contagem "N imóveis encontrados" atualiza a cada mudança. Categorias combinam em **E**; multi-select dentro de uma categoria combina em **OU**:

- Negócio: Venda / Aluguel / Ambos
- Tipo de imóvel: multi-select (casa, apartamento, comercial, terreno, cobertura, sobrado)
- Faixa de preço (usa `salePrice` ou `rentPrice` conforme negócio selecionado)
- Quartos / suítes / banheiros / vagas: mínimo
- Faixa de área construída
- Status: multi-select, todos ativos por padrão
- Mobiliado / Aceita pet: qualquer/sim/não
- Bairro: busca por texto sobre endereços cadastrados
- Corretor responsável: select
- Amenidades do condomínio: checkboxes (piscina, academia, salão de festas, playground, área pet, portaria 24h, elevador, condomínio fechado)

## Fluxo de interação

1. Corretor abre o arquivo local → mapa carrega centrado em Florianópolis continental, pins existentes (do IndexedDB) aparecem coloridos por status.
2. Painel de filtro à esquerda/lateral, lista de resultados sincronizada, mapa mostra só os pins que batem com o filtro atual.
3. **Cadastrar**: clica "+ Adicionar imóvel" → cursor indica modo de colocação → clique no mapa cria o pin → formulário abre com coordenadas preenchidas → preenche campos obrigatórios (tipo, negócio, preço, endereço) → opcionalmente escolhe/cria condomínio → anexa fotos (redimensionadas automaticamente) → salva → pin aparece com a cor do status.
4. **Editar**: clique num pin abre popup resumo com "Editar" (reabre formulário) / "Excluir" / "Ver detalhes". Pin pode ser arrastado pra ajustar posição, salva a nova coordenada automaticamente.
5. **Buscar match pra cliente**: ajusta os filtros → lista e mapa atualizam juntos → clique num item da lista centraliza o mapa nele.
6. **Backup**: botão "Exportar" baixa `.json` com tudo (imóveis + condomínios + fotos em base64); botão "Importar" repovoa o IndexedDB a partir de um arquivo exportado.

## Validação e erros

- Formulário não salva sem: tipo, negócio, preço, endereço, coordenadas — mostra o que falta.
- Fotos redimensionadas via canvas antes de virar Blob, evita IndexedDB inchado.
- Se o navegador não suportar IndexedDB, mostra aviso claro em vez de falhar silenciosamente.
- Self-check da função de matching de filtro roda uma vez no load (`console.assert`), loga erro no console se a lógica quebrar — sem framework de teste, proporcional ao tamanho do projeto.

## Entrega

- Local: `mapa-imoveis/index.html`, pasta nova, isolada dos demais projetos do repositório.
- Nenhuma alteração em arquivos existentes.

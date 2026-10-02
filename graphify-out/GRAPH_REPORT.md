# Graph Report - mapaimob  (2026-10-02)

## Corpus Check
- 20 files · ~49,074 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 162 nodes · 143 edges · 19 communities
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `3e0da3ef`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- File Structure
- Mapa de Imóveis (uso interno) — Design
- Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8)
- Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6)
- Mapa de Imóveis — Design System Overhaul (Fase 1)
- Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2)
- Mapa de Imóveis — Consistência de Design e Layout 3 Colunas
- Design System — Mapa de Imóveis
- Global Constraints
- Global Constraints
- Mapa de Imóveis — Dashboard (Fase 4)
- Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3)
- Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7)
- Global Constraints
- Global Constraints
- Global Constraints
- Global Constraints
- Global Constraints
- Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7) Implementation Plan

## God Nodes (most connected - your core abstractions)
1. `File Structure` - 11 edges
2. `Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8)` - 11 edges
3. `Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6)` - 10 edges
4. `Mapa de Imóveis (uso interno) — Design` - 9 edges
5. `Mapa de Imóveis — Design System Overhaul (Fase 1)` - 9 edges
6. `Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2)` - 8 edges
7. `Mapa de Imóveis — Consistência de Design e Layout 3 Colunas` - 8 edges
8. `Design System — Mapa de Imóveis` - 7 edges
9. `Mapa de Imóveis — Dashboard (Fase 4)` - 7 edges
10. `Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3)` - 7 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Communities (19 total, 0 thin omitted)

### Community 0 - "File Structure"
Cohesion: 0.14
Nodes (13): File Structure, Global Constraints, Mapa de Imóveis (uso interno) Implementation Plan, Task 10: Verificação end-to-end do MVP, Task 1: Estrutura base — HTML, CSS e mapa Leaflet vazio, Task 2: Camada de persistência IndexedDB, Task 3: Lógica pura — formatação, validação e matching de filtro, Task 4: Formulário de cadastro (imóvel + condomínio inline) (+5 more)

### Community 1 - "Mapa de Imóveis (uso interno) — Design"
Cohesion: 0.17
Nodes (11): Arquitetura, Contexto, Dentro do escopo, Entrega, Escopo, Filtros (núcleo da UX), Fluxo de interação, Fora do escopo (por ora) (+3 more)

### Community 2 - "Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8)"
Cohesion: 0.17
Nodes (11): Categorias e tags OSM, Contexto, Cores — paleta própria, fora do design system de UI, Erro de rede, Fonte de dados — confirmada com o usuário, Fora de escopo, Legenda, Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8) (+3 more)

### Community 3 - "Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6)"
Cohesion: 0.18
Nodes (10): Cartão de prédio (popup ao clicar o pin), Contexto, Decisão de escopo: o que entra nesta fase vs. o que fica pra depois, Fora de escopo, HUD flutuante sobre o mapa, Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6), Modelo de dados: condomínio vira "prédio" no mapa, Pin de condomínio (+2 more)

### Community 4 - "Mapa de Imóveis — Design System Overhaul (Fase 1)"
Cohesion: 0.20
Nodes (9): Contexto, Fora de escopo, Mapa de Imóveis — Design System Overhaul (Fase 1), Painel de adicionar/editar imóvel (`#propertyForm`), Referência visual (imagem 1 — "uphome"), Sidebar + cartão flutuante do mapa, Teste manual, Tokens de tema (+1 more)

### Community 5 - "Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2)"
Cohesion: 0.22
Nodes (8): Ajuste rápido no popup e no card da lista, Contexto, Dados novos no imóvel, Fora de escopo, Formulário (seção "Básico"), Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2), Teste manual, Validação

### Community 6 - "Mapa de Imóveis — Consistência de Design e Layout 3 Colunas"
Cohesion: 0.22
Nodes (8): Contexto, Controles do mapa nos cantos, `design.md` — documento de referência do design system, `financiamento.html` retemado pro claro/indigo, Fora de escopo, Layout do mapa em 3 colunas, Mapa de Imóveis — Consistência de Design e Layout 3 Colunas, Teste manual

### Community 7 - "Design System — Mapa de Imóveis"
Cohesion: 0.25
Nodes (7): Cores (`:root`), Design System — Mapa de Imóveis, Estado ativo/toggle (chips, abas, segmented controls), Fonte, Monoespaçada (valores numéricos/técnicos), Radius / Shadow, Regra de skin de botão

### Community 8 - "Global Constraints"
Cohesion: 0.25
Nodes (7): Global Constraints, Mapa de Imóveis — Design System Overhaul (Fase 1) Implementation Plan, Self-Review, Task 1: Retema (tokens CSS, fonte Inter, remoção do amarelo), Task 2: Ícones distintos por tipo de imóvel no pin/popup/lista, Task 3: Cartão flutuante do mapa ganha botão de recolher, Task 4: Formulário de imóvel em seções + rodapé fixo

### Community 9 - "Global Constraints"
Cohesion: 0.25
Nodes (7): Global Constraints, Mapa de Imóveis — Consistência de Design e Layout 3 Colunas — Implementation Plan, Self-Review, Task 1: `design.md` — documento de referência do design system, Task 2: Retemar `financiamento.html` pro claro/indigo, Task 3: Controle de zoom do Leaflet no canto inferior direito, Task 4: Layout do mapa em 3 colunas (lista vertical à esquerda)

### Community 10 - "Mapa de Imóveis — Dashboard (Fase 4)"
Cohesion: 0.25
Nodes (7): Contexto, Conteúdo (adaptado da imagem 3 pros nossos dados), Dashboard vira a aba padrão, Fora de escopo, Mapa de Imóveis — Dashboard (Fase 4), Segurança: não repetir o XSS da fase 3, Teste manual

### Community 11 - "Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3)"
Cohesion: 0.25
Nodes (7): Contexto, Fora de escopo, Layout (baseado na imagem 2, adaptado), Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3), Navegação, Reuso de código, Teste manual

### Community 12 - "Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7)"
Cohesion: 0.25
Nodes (7): Como funciona, Contexto, Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7), O que entra, O que não entra (decisão deliberada, não esquecimento), Segurança, Teste manual

### Community 13 - "Global Constraints"
Cohesion: 0.29
Nodes (6): Global Constraints, Mapa de Imóveis — Dashboard (Fase 4) Implementation Plan, Self-Review, Task 1: Dashboard como aba padrão + KPIs + fix de navegação do detalhe, Task 2: Balanço de vendas por período + unidades de lançamento, Task 3: Imóvel em destaque + tabela de recentes

### Community 14 - "Global Constraints"
Cohesion: 0.29
Nodes (6): Global Constraints, Mapa de Imóveis — HUD do mapa + Cartão de Prédio (Fase 6) Implementation Plan, Self-Review, Task 1: Condomínio ganha endereço, coordenadas, fotos e descrição, Task 2: HUD flutuante (toggle Unidade/Condomínio) + pins de condomínio, Task 3: Cartão de prédio completo (galeria, descrição, amenidades, unidades agenciadas, Agenciar)

### Community 15 - "Global Constraints"
Cohesion: 0.33
Nodes (5): Global Constraints, Mapa de Imóveis — Lançamento / Inventário de Unidades (Fase 2) Implementation Plan, Self-Review, Task 1: Dado + formulário + validação de lançamento, Task 2: Ajuste rápido (+/-) no popup e no card da lista

### Community 16 - "Global Constraints"
Cohesion: 0.33
Nodes (5): Global Constraints, Mapa de Imóveis — Página de Detalhe do Imóvel (Fase 3) Implementation Plan, Self-Review, Task 1: Casca navegável + conteúdo essencial, Task 2: Descrição, comodidades, unidades de lançamento e excluir

### Community 17 - "Global Constraints"
Cohesion: 0.33
Nodes (5): Global Constraints, Mapa de Imóveis — Legenda de amenidades no mapa (Fase 8) Implementation Plan, Self-Review, Task 1: Busca Overpass + pontos de amenidade no mapa (zoom-gated, com cache), Task 2: Legenda flutuante com toggle por categoria

### Community 18 - "Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7) Implementation Plan"
Cohesion: 0.40
Nodes (4): Global Constraints, Mapa de Imóveis — Link Financiamento ↔ Condomínio (Fase 7) Implementation Plan, Self-Review, Task 1: Botão "💰 Simular financiamento" na unidade do cartão de prédio

## Knowledge Gaps
- **114 isolated node(s):** `Fonte`, `Cores (`:root`)`, `Radius / Shadow`, `Monoespaçada (valores numéricos/técnicos)`, `Regra de skin de botão` (+109 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What connects `Fonte`, `Cores (`:root`)`, `Radius / Shadow` to the rest of the system?**
  _114 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `File Structure` be split into smaller, more focused modules?**
  _Cohesion score 0.14285714285714285 - nodes in this community are weakly interconnected._
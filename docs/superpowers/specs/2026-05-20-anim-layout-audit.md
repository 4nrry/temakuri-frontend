# Animações × Layout — Auditoria de Gaps

**Data:** 2026-05-20
**Objetivo:** Mapear quais animações de `/dev/anims` integram no board atual sem mudança de layout, quais pedem ajuste pequeno, e quais exigem redesenho estrutural. Saída orienta a ordem de implementação na integração progressiva.

## Estado atual do board (PlayArea + entorno)

- **Container:** `flex items-start justify-center gap-1.5 sm:gap-3 max-w-2xl`
- **Deck (Monte)** à esquerda. `PileStack` com ofsset visual de 2 camadas (não é stack alto). Width 48px, height 68px. **Escondido em modo duel.**
- **Pile (centro)** flex-1, cartas em linha com `gap-1 sm:gap-1.5`, `min-h-28`. Borda dashed quando vazio, sólida quando ativo.
- **Discard (Descarte)** à direita. `PileStack` idêntico ao deck. **Existe e está visível.**
- **Dividers verticais** `w-px` separam deck/pile e pile/discard. Layout apertado, sem grandes folgas.
- **MyHand:** mobile = scroll-snap horizontal; desktop = wrap centrado; cartas já usam `motion.div layout` (reorder animado).
- **OpponentRow:** card backs fanned (max 8 visíveis, height 28px, overlap `-ml-2`). Não há slot individual.
- **MarketRow:** linha separada acima do PlayArea.
- **Duel:** modal-based (`DuelPassPickModal`), **não há duas pilhas espaciais visíveis**.

Distância deck→pile observada: ~220px (já presente como offset em `drawVariants`).

## Matriz Animação × Layout

Legenda: ✅ = funciona como está · ⚠️ = ajuste pequeno · ❌ = redesenho estrutural

### Standalone / Overlay (independem de layout) — ✅

| Animação | Onde dispara | Notas |
|---|---|---|
| `invalidCardVariants` + `invalidTextVariants` | Carta tremendo após ação ilegal | Aplica em `<CardComponent>` via `useAnimationControls` |
| `SaborPopup` (fullscreen) | Estado de sabor ativo | Já tem `aliveKey` interno, basta passar trigger |
| `saborBannerVariants` + `saborGlowVariants` + `saborTideVariants` | Banner/glow de sabor no card | Aplicar em carta ou banner local |
| `winnerVariants` + `winnerGlowVariants` | Carta vencedora da rodada | Aplica no nó do vencedor |
| `losersVariants` | Cartas perdedoras (in-place dim) | Mesma posição |
| `victoryTitleVariants` + `victoryScoreVariants` | Modal/tela de fim | Overlay no GameOverModal |
| `defeatVariants` + `defeatTextVariants` | Mesma tela | Overlay |
| `chapterTitleVariants` | Transições entre fases | Overlay |
| `tokenRiseVariants` (custom: i) | RoundSummary/recompensa | Overlay numa tela já dedicada |
| `rankBadgeVariants`, `rankParticleVariants`, `rankUpTitleVariants` | Tela de rank-up pós-partida | Tela dedicada |
| `counterPillVariants` | Pill de contador (tokens/sabor) | In-place |
| `lastCardVariants` | Última carta da mão respira | In-place no `<CardComponent>` |
| `avatarBreathVariants` | Avatar do jogador da vez | In-place no avatar |
| `seatAvatarVariants` + `seatNameVariants` | Jogadores entrando/saindo | In-place |
| `comboSiblingVariants` + `nonComboVariants` | Highlight de combo na mão | In-place |
| `hoveredCardVariants` | Hover na mão | In-place |
| `passingHandVariants` + `passAtmosVariants` | Fase "passar" | In-place |
| `beatenCardVariants` | Carta que foi batida no pile | In-place sobre a carta |
| `dragDirectVariants` | Drag/release de carta | In-place |
| `reconnectAtmosVariants` + `sceneDimVariants` | Reconexão | Overlay |
| `sceneOutVariants` + `sceneInVariants` | Transições de cena | Overlay |
| `dealAtmosVariants` | Ambient durante deal | Overlay |
| `dealBalatroImpactVariants` | Pulse de impacto no deal | Overlay |
| `StageTremor` | Tremor em momentos grandes | Container-level |
| `CardImpactEffect` / `CardSlamEffect` (shockwave + glow) | Impacto em qualquer destino | Precisa de **âncora visual** no destino; pile tem |
| `shockwaveVariants` + `doubleShockwaveInner/Outer` | Anel expansivo | Mesmo que acima |

**Subtotal:** ~25 animações entram **sem** mudar layout.

### Ajuste pequeno — ⚠️

| Animação | O que pede | Esforço |
|---|---|---|
| `discardVariants` (flick F03) | Já existe pilha de Descarte à direita ✓. Precisa do **bounding rect** da pilha pra a carta voar até lá. `useRef` + `getBoundingClientRect` ou `layoutId`. | Baixo |
| `discardProductionVariants` | Idem | Baixo |
| `discardForceTrailVariants` | Idem (ghost trail) | Baixo |
| `drawVariants` (suave) | Deck à esquerda ✓. Hand no bottom. Travel diagonal de ~220px existe. Precisa medir destino na mão. | Baixo-Médio |
| `drawForceVariants` (F01) | Idem | Baixo-Médio |
| `playVariants` | MyHand → Pile center. Mesmo padrão de medição. | Baixo-Médio |
| `playForceVariants` (F02) | Idem | Baixo-Médio |
| `dealCalmSweepVariants` | Deck (esquerda) → Hand (bottom). Trajetória diagonal — verificar se a variant assume deck centralizado. | Médio |
| `dealBurstPopVariants` | Idem | Médio |
| `dealCascadeDropVariants` | Idem | Médio |
| `dealParentVariants` + `dealHandVariants` | Orquestrador do deal completo | Médio |
| `tableCardFlipVariants` (custom: i) | Cards no pile flipam ao serem revelados | Pile renderiza em flex row inline — flip in-place funciona. Verificar staggering. | Baixo |
| `flipPokemonHoldVariants` | Variante de flip dramática | Idem | Baixo |
| `idleFloatVariants` | Float idle das cartas na mão | Pode entrar como "respiração" passiva | Baixo |

**Causa raiz comum:** Todas essas animações precisam de **coordenadas concretas** (deck pos, pile pos, discard pos, slot na mão). Hoje o código produção não expõe esses refs. Solução: criar `useBoardLayout()` hook que devolve refs+rects dos pontos-chave do PlayArea, ou usar `layoutId` do Motion entre o "antes" e "depois".

### Redesenho estrutural — ❌

| Animação | Bloqueador | Mudança necessária |
|---|---|---|
| `beatPairForceVariants` (F04 atacante) | Hoje o pile é **flex row plano** — todas as cartas (par atacante + par derrotado) ficam misturadas. Não há divisão visual "meu par vs par derrotado". | Pile precisa **separar atacante e defensor** em duas sub-zonas adjacentes durante o momento de combate, OU usar `data-side="attacker/defender"` + grid de 2 colunas temporário. |
| `beatLoserPushVariants` (F04 derrotado) | Mesmo problema — não há "loser side" identificável visualmente. | Mesma solução. |
| `doubleShockwave*` em contexto de F04 | Sozinho funciona (overlay), mas o **propósito narrativo** depende do beat pair acima. | Vem com a solução de beat pair. |
| Todos os shuffles (`riffle`, `cut`, `pile`, `wash`, `spring`, `cascadeFan`, `overhand`, `tornado`, `dominoFall`, `hindu`) | Deck atual tem só 2 cartas visíveis em offset, num espaço de 48×68px. Não há **palco de shuffle**. Riffle quer 12 cartas com ~120px de folga de cada lado. | Criar **modo "shuffle stage"** — durante shuffle, o deck temporariamente expande pra uma zona central (ou um overlay no pile area), executa o shuffle, e contrai de volta. Idealmente como `<AnimatePresence>` que monta um portal/overlay durante a fase de embaralhamento. |
| Force draw caindo em OpponentRow | Card backs fanned com `-ml-2` overlap, height-cap 28px. Não há slot individual pra carta nova "encaixar". | Decidir: (a) animar opponent draw como uma carta voando do deck e desaparecendo no fan (visualmente abstrata), OU (b) reformular OpponentRow pra ter slots reais de carta com `layout` prop. |

### Áreas a confirmar (sem evidência ainda)

- **Onde renderizam `rankBadge` / `rankParticle` / `tokenRise` em produção?** A audit não vasculhou `RoundSummary`/`GameOverModal`. Provavelmente já é tela dedicada — confirmar antes de plugar.
- **Há momento "shuffling" no backend?** Se sim, a UI tem ~Xms de janela pra tocar shuffle. Se não, shuffle teria que ser puramente cosmético (não-bloqueante).

## Recomendações de ordem

Reagrupei o roteiro de integração com base nessa auditoria:

### Fase 1 — Quick wins (sem mudar layout)
1. **Invalid shake** — `invalidCardVariants` em `<CardComponent>`
2. **SaborPopup** — fullscreen no estado de sabor
3. **Winner glow** — `winnerGlowVariants` na carta vencedora
4. **Last card breathing** — `lastCardVariants` quando mão chega a 1
5. **Combo highlight** — `comboSiblingVariants` ao selecionar combo
6. **Avatar breath** — `avatarBreathVariants` no jogador da vez

### Fase 2 — Hook de coordenadas + animações de viagem (ajuste pequeno)
7. Criar `useBoardLayout()` ou `BoardLayoutContext` com refs de deck/pile/discard
8. **Discard flick** usando essas refs
9. **Draw force** (deck → hand)
10. **Play force** (hand → pile)
11. **Deal burst pop** no início da rodada (usando o hook)

### Fase 3 — Redesenho de pile pra F04
12. Refatorar `PlayArea` pile pra suportar zonas atacante/defensor temporárias
13. **beatPairForce + beatLoserPush + doubleShockwave** ao bater par

### Fase 4 — Shuffle stage
14. Criar palco de shuffle (overlay durante fase shuffling)
15. Tocar **riffle** (ou rotacionar 2-3 variantes) antes do deal

### Fase 5 — Polimento opcional
16. **StageTremor** em momentos grandes
17. **Token rise / rank up** na tela final (confirmar onde mora)
18. **Force draw em opponent** (definir abordagem)

## Verificação

- Cada item das Fases 1-2 deve ser PR pequena (<200 LOC), testada visualmente em `/dev/board` antes de produção quando aplicável.
- Cada PR liga uma animação; `prefers-reduced-motion` testado on/off.
- F04 (Fase 3) merece spec dedicado antes de codar — escopo maior.
- Shuffle stage (Fase 4) também merece spec dedicado.

## Resumo numérico

- **~25 animações** entram na Fase 1 (sem mudança).
- **~14 animações** entram na Fase 2 (ajuste pequeno: hook de coordenadas).
- **~12 animações** (F04 + 10 shuffles) ficam na Fase 3+ (redesenho).

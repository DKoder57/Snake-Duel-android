# Arquitetura Técnica — Snake Duel: Solo

Este documento detalha os sistemas técnicos desta versão, seguindo o mesmo princípio de separação de responsabilidades do projeto original: lógica de jogo isolada de qualquer camada de plataforma (aqui, touch/Android em vez de rede).

> **Nota de transparência:** não houve acesso direto ao arquivo técnico original do projeto (referenciado no README principal como `snake-duel-arquitetura-tecnica.md`) durante a geração deste documento — apenas ao README público. A estrutura abaixo foi inferida a partir das descrições públicas do projeto original e adaptada para esta versão. Ajuste nomes de classe e detalhes conforme o código real antes de tratar este documento como fonte de verdade.

## Visão geral de camadas

```
Core/         → lógica pura de jogo (grid, cobra, itens, bombas)
AI/           → pathfinding A* sobre grid toroidal
Input/        → NOVO — abstrai touch (swipe, D-pad) em comandos de direção
Persistence/  → NOVO — recorde local, configurações, conquistas
UI/           → HUD, menus, telas de configuração (redesenhados para mobile)
```

A camada `Network/` (Mirror) do projeto original foi **removida** nesta versão — o modo solo nunca dependeu dela, então ela não é apenas desativada, e sim excluída do projeto.

## Core

- **`GridPosition`** — sistema de coordenadas com aritmética modular, responsável pelo wraparound toroidal. Reutilizado sem alterações do projeto original.
- **`SnakeState`** — representa o estado da cobra (corpo, direção, velocidade). Reutilizado do projeto original.
- **`BombSystem`** — calcula área de explosão 3x3 considerando wraparound. Precisa de testes unitários cobrindo bombas em `x=0`, `x=23` e cantos (ver riscos no README).
- **`GameLoop` / `TickController`** — controla o tick de movimento (acelerando ao longo da partida, ex: 180ms → 135ms). Sem sincronização de rede nesta versão, o dispositivo local é sempre a única fonte de verdade — elimina por completo o risco de dessincronia do projeto original.

## AI

- **`SnakeAI` (A*)** — pathfinding real sobre o grid toroidal, tratando bombas armadas como obstáculos temporários no grafo de busca. Herdado do projeto original.
- **Fallback de caminho não encontrado** *(a implementar — prioridade máxima)* — quando o grid está muito cheio perto do fim da partida, o A* precisa de um comportamento definido (continuar na direção atual, ou buscar a célula livre mais próxima) em vez de travar ou lançar exceção. Aqui a IA é o modo principal do jogo, não um extra, então este item vira bloqueador de lançamento.
- **Níveis de dificuldade** *(a implementar)* — parametrizar agressividade da IA (prioridade de itens vs. bombas, distância de reação, tempo de decisão) em tiers distintos, cada um com nome/persona simples para dar identidade ao oponente.

## Input (novo nesta versão)

- **`InputController`** — abstrai swipe e D-pad virtual, convertendo o gesto em comandos de direção compatíveis com o que o `Core` já esperava do teclado no projeto original. O `Core` não precisa saber a diferença entre as fontes de input.
- **Zona morta de swipe** configurável, para reduzir falsos positivos de direção em telas pequenas ou com toques imprecisos.

## Persistence (novo nesta versão)

- **`SaveSystem`** — recorde local (highscore), configurações do jogador (sensibilidade, som, vibração) e progresso de conquistas. Armazenamento local simples (ex: `PlayerPrefs` ou arquivo JSON local), sem backend — consistente com a filosofia "sem custo de infraestrutura" do projeto original.

## Build Android

- Unity Android module (IL2CPP), com `minSdkVersion`/`targetSdkVersion` configurados para API 36 (Android 16), atendendo ao requisito atual da Google Play para novos apps.
- Build gerado em formato AAB (Android App Bundle), não APK solto.
- Play App Signing habilitado; upload keystore com backup obrigatório (pelo menos duas cópias seguras) antes do primeiro upload — a perda dessa chave impede atualizações futuras do mesmo app.
- Relatório de crash/ANR (ex: Firebase Crashlytics) integrado antes do início do teste fechado, para monitorar estabilidade durante os 14 dias exigidos pela Google Play.

## Testes

Mesma filosofia do projeto original: testar toda a lógica de `Core`, `AI` e, agora, `Input`/`Persistence`, sem depender de build completo. Cobertura mínima recomendada antes da produção:

- `BombSystem`: bomba em bordas e cantos do grid (`x=0`, `x=23`)
- `SnakeAI`: fallback acionado corretamente quando não há caminho válido
- `InputController`: conversão correta de swipe em direção, incluindo casos de swipe diagonal ambíguo
- `SaveSystem`: persistência de recorde e configurações entre sessões (incluindo após fechar o app à força)

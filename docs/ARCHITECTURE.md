# Arquitetura Técnica — Snake Duel: Solo

Este documento detalha os sistemas técnicos desta versão, com profundidade suficiente para começar a implementação direto — assinaturas de classe, constantes assumidas, fluxo de execução e as decisões que ainda precisam ser fechadas antes de codar.

> **Nota de transparência:** não houve acesso direto ao código-fonte nem aos arquivos técnicos do projeto original durante a geração deste documento — apenas ao README público. Toda constante numérica abaixo (tamanho de grid, timings) é uma **suposição de trabalho**, marcada como tal, para destravar o desenvolvimento. Substitua pelos valores reais do projeto original assim que tiver acesso ao código, e apague as marcações de suposição conforme forem confirmadas.

---

## Princípio central: Core não conhece o motor

Regra que vale para todo este documento: nenhuma classe de `Core` ou `AI` referencia `GameObject`, `Transform`, `MonoBehaviour` ou qualquer tipo do namespace `UnityEngine` além de tipos de dados puros (`Vector2Int`, se necessário). Isso é o que torna possível rodar `GridPositionTests`/`SnakeAITests` em EditMode sem abrir uma cena, e é o que separa **estado do jogo** (o que aconteceu) de **apresentação** (como isso aparece na tela) — a UI reage a eventos do Core, o Core nunca sabe que existe uma UI.

---

## Estrutura de pastas e Assembly Definitions

```
Assets/
└── _Project/
    ├── Core/            (asmdef: SnakeDuelSolo.Core)
    │   ├── Grid/         GridPosition, GridBoard
    │   ├── Snake/        SnakeState, ActiveEffect
    │   ├── Items/        IItem, Food, PuffedApple, AbilityOrb, Bomb, BombSystem
    │   └── Loop/          GameLoop, TickController, MatchState
    ├── AI/               (asmdef: SnakeDuelSolo.AI — depende só de Core)
    │   ├── SnakeAI.cs
    │   ├── PathfindingService.cs (A*)
    │   └── AIDifficultyProfile.cs (ScriptableObject)
    ├── Input/            (asmdef: SnakeDuelSolo.Input — depende só de Core)
    │   └── InputController.cs, ITouchInputSource.cs
    ├── Persistence/       (asmdef: SnakeDuelSolo.Persistence — depende só de Core)
    │   └── SaveSystem.cs, SaveData.cs
    ├── Presentation/      (asmdef: SnakeDuelSolo.Presentation — depende de Core; NUNCA o contrário)
    │   ├── SnakeSegmentView.cs, GridTilemapRenderer.cs, BombTelegraphView.cs
    ├── UI/                (asmdef: SnakeDuelSolo.UI — depende de tudo acima)
    │   ├── HUD/, Menus/, Settings/
    ├── Bootstrap/         (asmdef: SnakeDuelSolo.Bootstrap — depende de tudo; é o único lugar que "conhece todo mundo")
    │   └── MatchController.cs
    └── Tests/
        ├── EditMode/      (asmdef: SnakeDuelSolo.Tests.Edit)
        └── PlayMode/      (asmdef: SnakeDuelSolo.Tests.Play)
```

**Por que Assembly Definitions (`.asmdef`) e não só pastas por convenção:** pastas sozinhas não impedem que um script em `UI/` referencie algo interno de `AI/` por engano — isso só é pego em code review, se for pego. Com `.asmdef` por camada e referências explícitas entre eles, um `Core` tentando referenciar `Presentation` **não compila**. Efeito colateral bom: cada assembly recompila independente, então mudanças em `UI/` não forçam recompilar `Core/AI` — iteração mais rápida no Editor.

**Regra de dependência:** `Core` não referencia nenhum outro assembly do projeto. `AI`, `Input` e `Persistence` referenciam só `Core`. `Presentation` referencia só `Core` (é onde `GameObject`/`MonoBehaviour` entram em cena pela primeira vez). `UI` referencia `Core` + `Presentation`. `Bootstrap` é o único assembly com permissão de conhecer todos os outros — é ele quem monta o grafo de dependências em runtime.

---

## Convenções e constantes assumidas

| Constante | Valor assumido | Status |
|---|---|---|
| `GRID_WIDTH` / `GRID_HEIGHT` | 24 x 24 | Suposição — inferida de exemplos de teste (`x=0`, `x=23`) discutidos, não confirmada no código real |
| Intervalo de tick inicial | 180 ms | Suposição — valor citado no README original, curva exata não confirmada |
| Intervalo de tick mínimo | 135 ms | Suposição — idem |
| Duração da partida (modo cronometrado) | 120 s | Suposição, com base no "2 minutos" mencionado no projeto original |
| Janela de escolha de buff/debuff | 2 s | Confirmada como mecânica existente no projeto original |
| Raio de explosão da bomba | 3x3 (Chebyshev, distância 1) | Confirmada como mecânica existente no projeto original |
| Telegraph da bomba antes de detonar | 500 ms | Suposição nova desta versão (tabuleiro compartilhado exige aviso visual) |

---

## Core

### `GridPosition`

```csharp
public readonly struct GridPosition : IEquatable<GridPosition>
{
    public int X { get; }
    public int Y { get; }

    // Construtor já normaliza para o wraparound toroidal:
    // x = ((x % width) + width) % width  (evita valor negativo do % em C#)
    public GridPosition(int x, int y, int width, int height) { /* ... */ }

    public static GridPosition operator +(GridPosition pos, Direction dir) { /* ... */ }

    // Distância "real" no toro: menor entre o caminho direto e o caminho
    // dando a volta em cada eixo. Usada pela heurística do A* (ver AI).
    public static int ToroidalManhattanDistance(GridPosition a, GridPosition b, int width, int height) { /* ... */ }
}

public enum Direction { Up, Down, Left, Right }
```

O `%` de C# preserva o sinal do operando à esquerda — `-1 % 24` dá `-1`, não `23`. Qualquer implementação de wraparound que ignore isso quebra silenciosamente na borda esquerda/superior do grid. Vale um teste unitário dedicado só pra essa normalização.

### Tabuleiro compartilhado

Com a decisão de usar um único tabuleiro (ver `DECISIONS.md`), `GridBoard` passa a ser dono de **duas** `SnakeState` (jogador e IA) sobre o mesmo espaço de coordenadas — não duas instâncias independentes de grid. Toda consulta de colisão (corpo próprio, corpo do oponente, bomba) precisa considerar as duas cobras.

```csharp
public class GridBoard
{
    public SnakeState Player { get; }
    public SnakeState Opponent { get; }
    public IReadOnlyList<IItem> ActiveItems { get; }

    public CollisionResult CheckCollision(GridPosition nextHead, SnakeState mover);
}

public enum CollisionResult { None, SelfBody, OpponentBody, OpponentHeadOnHead, Bomb }
```

`OpponentHeadOnHead` existe como caso próprio porque a regra pra ele **ainda não está definida** — ver "Decisões técnicas abertas" no final deste documento.

### `SnakeState`

```csharp
public class SnakeState
{
    public IReadOnlyList<GridPosition> Body { get; }   // Body[0] = cabeça
    public Direction CurrentDirection { get; private set; }
    public bool IsAlive { get; private set; }
    public int Score { get; private set; }
    public ActiveEffect? CurrentEffect { get; private set; } // null = nenhum ativo

    public void QueueDirection(Direction dir);   // input fica em fila até o próximo tick
    public void Advance(GridPosition newHead, bool grew);
    public void Kill();
    public void ApplyEffect(ActiveEffect effect); // substitui o efeito atual, não empilha
    public void TickEffectDuration(float deltaSeconds); // chamado 1x por tick pelo GameLoop
}
```

`QueueDirection` (em vez de aplicar a direção na hora) evita o bug clássico de Snake onde dois inputs rápidos no mesmo tick permitem a cobra "virar 180°" e colidir com o próprio pescoço.

### `ActiveEffect` (buff/debuff)

```csharp
public struct ActiveEffect
{
    public EffectType Type;
    public float RemainingDurationSeconds;
    public float Magnitude; // ex: 1.5 para velocidade x1.5, 0.5 para lentidão
}

public enum EffectType { SpeedBoost, SpeedSlow /* completar com os efeitos reais do projeto original */ }
```

Modelado como `struct` (não `class`) porque é um valor descartável recriado a cada aplicação — não precisa de identidade própria nem de referência compartilhada.

### Sistema de itens

```csharp
public interface IItem
{
    GridPosition Position { get; }
    void OnConsumed(SnakeState consumer, MatchState match);
}
```

Cada item conhecido do projeto original implementa essa interface: `Food` (cresce 1), `PuffedApple`/maçã bufada (efeito exato a confirmar — ver decisões abertas), `AbilityOrb` (concede habilidade ativa), `Bomb` (não é consumido por contato — ver `BombSystem`). Modelar como interface em vez de `enum` + `switch` central evita que `GameLoop` precise conhecer as regras de cada item; adicionar um item novo não exige tocar em código existente (Open/Closed).

### `BombSystem`

Área de explosão 3x3 usando distância de Chebyshev, aplicando o wraparound do `GridPosition` em cada célula afetada — sem isso, uma bomba em `x=23` "esquece" de afetar `x=0`.

```csharp
public enum BombPhase { Armed, Telegraphing, Detonated }

public class BombSystem
{
    public BombPhase Phase { get; private set; }
    public float TelegraphRemainingSeconds { get; private set; }

    public void Tick(float deltaSeconds); // Armed -> Telegraphing -> Detonated
    public IEnumerable<GridPosition> GetAffectedCells(int width, int height);
}
```

**Telegraph antes da detonação:** com o tabuleiro compartilhado, o jogador precisa de aviso visual antes da explosão pra ter chance real de desviar (isso não era necessário quando bombas só afetavam o próprio tabuleiro do jogador que a plantou). `Presentation` observa `Phase == Telegraphing` e faz a célula piscar; `Core` não sabe nada sobre "piscar", só expõe o estado.

Casos de teste obrigatórios: bomba em `x=0`, `x=GRID_WIDTH-1`, e nos 4 cantos do grid.

### `MatchState` e condição de vitória

```csharp
public enum MatchOutcome { InProgress, PlayerWins, OpponentWins, Draw }

public class MatchState
{
    public float ElapsedSeconds { get; private set; }
    public MatchOutcome Outcome { get; private set; }

    public void Advance(float deltaSeconds);
    public MatchOutcome EvaluateEndCondition(GridBoard board); // ver decisão aberta #6
}
```

A condição de fim de partida tem duas leituras possíveis — timeout (compara `Score` das duas cobras quando `ElapsedSeconds` atinge a duração da partida) ou eliminação (se uma cobra morre antes do tempo, a partida acaba ali). Isso está listado como decisão aberta porque muda o `EvaluateEndCondition` de forma incompatível — não dá pra implementar os dois "só por garantia".

### `GameLoop` / `TickController`

```csharp
public class TickController
{
    public float CurrentIntervalMs { get; private set; }
    public event Action OnTick;

    public void Tick(float deltaTime);           // acumula deltaTime, dispara OnTick no intervalo certo
    private float ComputeIntervalForElapsed(float matchElapsedSeconds);
}
```

Sem sincronização de rede nesta versão, o dispositivo local é sempre a única fonte de verdade — elimina por completo o risco de dessincronia do projeto original. A curva exata de aceleração (linear? por degraus a cada N segundos?) é uma das suposições em aberto listadas acima.

### Fluxo de execução por tick

Ordem fixa de operações a cada disparo de `OnTick` — importante deixar explícita porque a ordem errada cria bugs sutis (ex: checar colisão antes de mover as duas cobras faria uma cobra "ver o futuro" da outra):

```
1. Resolver a direção enfileirada de cada cobra
   (Player: última QueueDirection recebida · IA: SnakeAI.DecideNextMove)
2. Calcular a próxima GridPosition da cabeça de cada cobra (posição atual + direção)
3. Checar colisão das DUAS cabeças-alvo contra o estado ATUAL do tabuleiro
   (corpo próprio, corpo do oponente, cabeça-a-cabeça, bomba)
4. Aplicar Advance() nas duas cobras simultaneamente (nenhuma "vê" o resultado da outra antes do passo 3)
5. Resolver consumo de item para qualquer cabeça que tenha aterrissado num IItem
6. Avançar o timer de telegraph de bombas armadas; detonar as que chegarem a zero
7. Decrementar a duração do efeito ativo de cada cobra (TickEffectDuration); limpar os expirados
8. MatchState.EvaluateEndCondition — se retornar algo diferente de InProgress, encerrar a partida
9. Disparar eventos pra UI (placar mudou, efeito mudou, bomba telegraphing, partida terminou)
```

O passo 3 acontecer **antes** do passo 4 pras duas cobras é o que garante que a ordem de avaliação (jogador primeiro ou IA primeiro) não influencie quem "ganha" uma disputa pela mesma célula — ambas colidem, se for o caso, em vez de uma "roubar" a célula da outra por sorte de ordem de execução.

---

## AI

### `SnakeAI` (A*)

Pathfinding real sobre o grid toroidal. Nós do grafo de busca são `GridPosition`; heurística é `GridPosition.ToroidalManhattanDistance` (ver Core) — Manhattan comum, sem a adaptação ao wraparound, faz a IA subestimar sistematicamente distâncias perto da borda. Corpo de ambas as cobras e bombas armadas (fase `Armed` ou `Telegraphing`) entram como obstáculos temporários no grafo.

### Fallback de caminho não encontrado *(bloqueador de lançamento)*

```csharp
public Direction DecideNextMove(GridBoard board, SnakeState self)
{
    if (TryFindPath(board, self, out var path))
        return path.FirstStep;

    // Fallback, nessa ordem de prioridade:
    // 1. Continuar na direção atual, se a célula à frente estiver livre agora
    // 2. Escolher, entre os vizinhos livres, o que maximiza espaço livre ao redor
    //    (flood fill raso, não A* completo — mais barato de rodar todo tick)
    // 3. Se não houver nenhum vizinho livre, aceitar o movimento (morte
    //    inevitável) em vez de lançar exceção ou travar o frame
}
```

O passo 3 existe porque, no fim de uma partida cheia, pode genuinamente não haver saída — o objetivo do fallback não é evitar a morte da IA, é garantir que o jogo **nunca trava** por causa disso.

### Perfis de dificuldade

Modelar cada tier como `ScriptableObject` em vez de constantes no código:

```csharp
[CreateAssetMenu(menuName = "SnakeDuelSolo/AI Difficulty Profile")]
public class AIDifficultyProfile : ScriptableObject
{
    public string DisplayName;           // persona/nome do oponente
    public float DecisionDelayMs;        // tempo de "reação" simulado
    public float ItemPriorityWeight;     // o quanto a IA desvia de rota pra pegar item
    public float MistakeChance;          // chance de decisão subótima de propósito
}
```

Vantagem sobre valores hardcoded: designers (ou você mesmo, sem recompilar) ajustam dificuldade direto no Inspector, e cada tier vira um asset versionado no Git, revisável em PR como dado, não como código.

---

## Input

```csharp
public interface ITouchInputSource
{
    event Action<Direction> OnDirectionInput;
}

public class SwipeInputSource : MonoBehaviour, ITouchInputSource { /* ... */ }
public class VirtualDPadInputSource : MonoBehaviour, ITouchInputSource { /* ... */ }
```

`InputController` depende só da interface `ITouchInputSource`, nunca de `SwipeInputSource`/`VirtualDPadInputSource` diretamente — trocar o método de controle nas configurações é trocar qual implementação está ativa, sem tocar em `Core`. Zona morta de swipe configurável (valor sugerido pra começar: 40dp de deslocamento mínimo antes de registrar uma direção — ajustável, não confirmado por teste com usuário real).

---

## Persistence

```csharp
[Serializable]
public class SaveData
{
    public int SaveDataVersion;  // pra migração futura sem quebrar saves antigos
    public int HighScore;
    public float SwipeSensitivity;
    public float SoundVolume;
    public bool VibrationEnabled;
    public List<string> UnlockedAchievementIds;
}
```

Serializado via `JsonUtility` para um arquivo em `Application.persistentDataPath` — sem backend, consistente com a filosofia "sem custo de infraestrutura" do projeto original. `SaveSystem` expõe `Load()`/`Save(SaveData)` só; nada além disso precisa saber que o formato é JSON em disco. `SaveDataVersion` existe desde a v1 para evitar o problema clássico de "adicionei um campo novo e os saves antigos quebram" — sem ele, migração vira reescrita manual do zero mais tarde.

---

## Presentation (camada de visual — nova nesta versão de detalhamento)

Onde `GameObject`/`MonoBehaviour`/`SpriteRenderer` entram pela primeira vez — `Core` nunca referencia nada daqui (ver "Princípio central" no topo).

- **`GridTilemapRenderer`** — usa o componente `Tilemap` nativo do Unity pro fundo do grid (mais barato que instanciar um `GameObject` por célula, e permite trocar o tileset visual sem tocar em código).
- **`SnakeSegmentView`** — um pool de `SpriteRenderer` (ver "Boas práticas" abaixo) reflete `SnakeState.Body` a cada tick; nunca decide nada, só desenha o que `Core` já decidiu.
- **`BombTelegraphView`** — observa `BombSystem.Phase`; anima o piscar durante `Telegraphing`.
- **Direção artística "neon-arcade" (ver `DECISIONS.md`):** URP (Universal Render Pipeline) com pós-processamento **Bloom** é o caminho padrão pra esse efeito — materiais com emissão nas cores de papel (azul do jogador, laranja da IA, etc. — ver tabela de cores) "brilham" via Bloom sem precisar desenhar glow manualmente em cada sprite.

---

## Bootstrap: como tudo se conecta

```csharp
public class MatchController : MonoBehaviour
{
    // Único ponto do projeto que instancia e conecta Core, AI, Input,
    // Persistence, Presentation e UI entre si. Se alguma dessas camadas
    // precisar conhecer outra diretamente (fora daqui), é sinal de que a
    // regra de dependência dos Assembly Definitions foi violada.
    private GridBoard _board;
    private TickController _tickController;
    private SnakeAI _ai;
    private InputController _input;

    private void Awake()
    {
        // monta o grafo de dependências, assina eventos do Core na
        // Presentation/UI, e só então dá Start() na partida
    }
}
```

---

## Build Android

- Unity Android module (IL2CPP), `minSdkVersion`/`targetSdkVersion` = API 36 (Android 16).
- Build gerado em formato AAB (Android App Bundle), não APK solto.
- Play App Signing habilitado; upload keystore com backup obrigatório (ver `DECISIONS.md`) antes do primeiro upload.
- Firebase Crashlytics (ou equivalente) integrado antes do início do teste fechado, para monitorar estabilidade durante os 14 dias exigidos pela Google Play.

---

## Testes

Unity Test Framework (UTF), separado por assembly (ver estrutura de pastas):

- **EditMode** (`SnakeDuelSolo.Tests.Edit`): toda a lógica de `Core` e `AI` roda sem abrir uma cena — mais rápido, e é o que o job `code-check` do CI pode rodar a cada PR.
  - `GridPositionTests`: wraparound em `x=0`, `x=GRID_WIDTH-1`, e valores negativos
  - `BombSystemTests`: bomba em bordas e nos 4 cantos; transição `Armed → Telegraphing → Detonated`
  - `SnakeAITests`: fallback acionado corretamente quando não há caminho válido
  - `MatchStateTests`: `EvaluateEndCondition` nos dois cenários possíveis (timeout e eliminação — cobrir os dois até a decisão aberta #6 ser fechada)
  - `InputControllerTests`: conversão de swipe em direção, incluindo swipe diagonal ambíguo
- **PlayMode** (`SnakeDuelSolo.Tests.Play`): cenários que precisam de `MonoBehaviour`/frame real (ex: `TickController` acelerando corretamente ao longo do tempo, persistência sobrevivendo a um "fechar app à força" simulado).

Exemplo de assertion (EditMode), pra fixar o estilo esperado:

```csharp
[Test]
public void GridPosition_WrapsNegativeX_ToRightEdge()
{
    var pos = new GridPosition(-1, 5, width: 24, height: 24);
    Assert.AreEqual(23, pos.X);
}
```

---

## Boas práticas adicionais (comparação com projetos reais de Unity)

Pontos que não estavam no design original, mas que valem a pena adotar desde o início por serem prática comum em jogos Unity de grid/arcade:

- **Object pooling pros segmentos da cobra.** Cada `Advance()` que faz a cobra crescer não deve gerar/destruir um `GameObject` novo por segmento a cada tick — isso gera picos de GC perceptíveis em mobile. Um pool simples de `SnakeSegmentView` reaproveitados resolve isso sem complexidade extra.
- **ScriptableObject Event Channels** para comunicação entre `Core` e `UI`/`Presentation` (ex: "partida terminou", "buff aplicado") em vez de `UI` chamar `FindObjectOfType` ou `Core` ter uma referência direta a um Canvas. Mantém a regra de dependência da seção de Assembly Definitions realmente unidirecional.
- **`[SerializeField] private` em vez de campos públicos** nos `MonoBehaviour` de `UI`/`Input`, com os perfis de IA e configurações expostos só via `ScriptableObject` — reduz o que aparece exposto sem querer no Inspector.
- **Evitar `GetComponent`/`FindObjectOfType` dentro de `Update`** — é exatamente o tipo de erro que o Microsoft.Unity.Analyzers (já adicionado ao CI, ver `DECISIONS.md`) pega automaticamente, mas vale ter isso em mente ao já escrever o código pela primeira vez.

---

## Decisões técnicas abertas (resolver antes de codar F1)

Itens que este documento **não resolve sozinho** — precisam de uma decisão explícita (e um novo registro em `DECISIONS.md`) antes de virar código:

1. **Regra de colisão cabeça-a-cabeça simultânea** (`OpponentHeadOnHead`): empate (as duas cobras morrem), ou desempate por algum critério (tamanho, quem entrou na célula primeiro no sub-tick)?
2. **Confirmar `GRID_WIDTH`/`GRID_HEIGHT` reais** do projeto original — todo o resto deste documento assume 24x24.
3. **Confirmar a curva real de aceleração do tick** (180ms → 135ms) — linear ao longo da partida, ou em degraus a intervalos fixos?
4. **Efeito exato da "maçã bufada"** — o nome sugere um crescimento maior que o da comida normal, ou um buff temporário; isso muda a assinatura de `IItem.OnConsumed` e a lista de `EffectType`.
5. **A dificuldade da IA afeta a velocidade de tick dela**, ou só a qualidade das decisões (via `AIDifficultyProfile`)? Afeta diretamente se `TickController` precisa ser por-cobra ou único pra partida toda.
6. **Condição de vitória**: se uma cobra morre antes do cronômetro zerar, a partida termina ali (vitória imediata do sobrevivente) ou continua até o tempo acabar? Muda `MatchState.EvaluateEndCondition` de forma incompatível entre as duas leituras.

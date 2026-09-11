# Arquitetura Técnica — Snake Duel: Solo

Este documento detalha os sistemas técnicos desta versão, com profundidade suficiente para começar a implementação direto — assinaturas de classe, constantes assumidas e as decisões que ainda precisam ser fechadas antes de codar.

> **Nota de transparência:** não houve acesso direto ao código-fonte nem aos arquivos técnicos do projeto original durante a geração deste documento — apenas ao README público. Toda constante numérica abaixo (tamanho de grid, timings) é uma **suposição de trabalho**, marcada como tal, para destravar o desenvolvimento. Substitua pelos valores reais do projeto original assim que tiver acesso ao código, e apague as marcações de suposição conforme forem confirmadas.

---

## Estrutura de pastas e Assembly Definitions

```
Assets/
└── _Project/
    ├── Core/            (asmdef: SnakeDuelSolo.Core)
    │   ├── Grid/         GridPosition, GridBoard
    │   ├── Snake/        SnakeState, SnakeController
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
    ├── UI/                (asmdef: SnakeDuelSolo.UI — depende de tudo acima)
    │   ├── HUD/, Menus/, Settings/
    └── Tests/
        ├── EditMode/      (asmdef: SnakeDuelSolo.Tests.Edit)
        └── PlayMode/      (asmdef: SnakeDuelSolo.Tests.Play)
```

**Por que Assembly Definitions (`.asmdef`) e não só pastas por convenção:** pastas sozinhas não impedem que um script em `UI/` referencie algo interno de `AI/` por engano — isso só é pego em code review, se for pego. Com `.asmdef` por camada e referências explícitas entre eles, um `Core` tentando referenciar `UI` **não compila**. Isso é o que de fato mantém a separação Core/AI/Input/Persistence/UI válida ao longo do tempo, em vez de depender de disciplina manual. Efeito colateral bom: cada assembly recompila independente, então mudanças em `UI/` não forçam recompilar `Core/AI` — iteração mais rápida no Editor.

**Regra de dependência:** `Core` não referencia nenhum outro assembly do projeto. `AI`, `Input` e `Persistence` referenciam só `Core`. `UI` pode referenciar todos. Isso é o que torna `Core`/`AI` testáveis em EditMode sem precisar instanciar nada de UI ou Android.

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
}

public enum Direction { Up, Down, Left, Right }
```

O `% ` de C# preserva o sinal do operando à esquerda — `-1 % 24` dá `-1`, não `23`. Qualquer implementação de wraparound que ignore isso quebra silenciosamente na borda esquerda/superior do grid. Vale um teste unitário dedicado só pra essa normalização.

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
    public ActiveEffect? CurrentBuffOrDebuff { get; private set; } // null = nenhum ativo

    public void QueueDirection(Direction dir);   // input fica em fila até o próximo tick
    public void Advance(GridPosition newHead, bool grew);
    public void Kill();
    public void ApplyEffect(ActiveEffect effect);
}
```

`QueueDirection` (em vez de aplicar a direção na hora) evita o bug clássico de Snake onde dois inputs rápidos no mesmo tick permitem a cobra "virar 180°" e colidir com o próprio pescoço.

### Sistema de itens

```csharp
public interface IItem
{
    GridPosition Position { get; }
    void OnConsumed(SnakeState consumer, MatchState match);
}
```

Cada item conhecido do projeto original implementa essa interface: `Food` (cresce 1), `PuffedApple`/maçã bufada (efeito a confirmar — nome sugere crescimento maior ou buff temporário), `AbilityOrb` (concede habilidade ativa), `Bomb` (não é consumido por contato — ver `BombSystem`). Modelar como interface em vez de `enum` + `switch` central evita que `GameLoop` precise conhecer as regras de cada item; adicionar um item novo não exige tocar em código existente (Open/Closed).

### `BombSystem`

Área de explosão 3x3 usando distância de Chebyshev, aplicando o wraparound do `GridPosition` em cada célula afetada — sem isso, uma bomba em `x=23` "esquece" de afetar `x=0`.

**Telegraph antes da detonação:** com o tabuleiro compartilhado, o jogador precisa de aviso visual antes da explosão pra ter chance real de desviar (isso não era necessário quando bombas só afetavam o próprio tabuleiro do jogador que a plantou). Sugestão: `BombSystem` expõe um estado `Armed → Telegraphing (ex: 500 ms) → Detonated`, e a camada de `UI` reage ao estado `Telegraphing` piscando a célula.

Casos de teste obrigatórios: bomba em `x=0`, `x=GRID_WIDTH-1`, e nos 4 cantos do grid.

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

---

## AI

### `SnakeAI` (A*)

Pathfinding real sobre o grid toroidal. Nós do grafo de busca são `GridPosition`; heurística é distância de Manhattan **adaptada ao wraparound** (a distância real entre dois pontos é o mínimo entre o caminho direto e o caminho "dando a volta" em cada eixo — usar Manhattan comum sem essa adaptação faz a IA subestimar sistematicamente distâncias perto da borda). Corpo de ambas as cobras e bombas armadas entram como obstáculos temporários no grafo.

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
    public int HighScore;
    public float SwipeSensitivity;
    public float SoundVolume;
    public bool VibrationEnabled;
    public List<string> UnlockedAchievementIds;
}
```

Serializado via `JsonUtility` para um arquivo em `Application.persistentDataPath` — sem backend, consistente com a filosofia "sem custo de infraestrutura" do projeto original. `SaveSystem` expõe `Load()`/`Save(SaveData)` só; nada além disso precisa saber que o formato é JSON em disco (poderia virar PlayerPrefs ou um save na nuvem depois, sem mudar quem consome).

---

## UI / HUD

Hierarquia do HUD em partida, decidida na revisão de design:

- **Topo:** placar (esquerda) · temporizador (centro) · botão de configurações (direita)
- **Logo abaixo:** indicador de buff/debuff ativo, com contagem regressiva visível — sem isso o jogador não sabe quanto tempo falta pro efeito acabar
- **Área principal:** o tabuleiro único compartilhado
- **Legenda de itens:** fixa ou acessível via tutorial, nunca escondida em submenu

Paleta por papel semântico (constantes, não valores soltos por tela):

| Papel | Cor | Uso |
|---|---|---|
| Jogador | Azul (frio) | Cobra do jogador |
| IA | Laranja (quente) | Cobra da IA |
| Positivo | Verde | Comida |
| Atenção | Âmbar | Maçã bufada |
| Especial | Roxo | Orbe de habilidade |
| Perigo | Vermelho | Bomba |

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
  - `BombSystemTests`: bomba em bordas e nos 4 cantos
  - `SnakeAITests`: fallback acionado corretamente quando não há caminho válido
  - `InputControllerTests`: conversão de swipe em direção, incluindo swipe diagonal ambíguo
- **PlayMode** (`SnakeDuelSolo.Tests.Play`): cenários que precisam de `MonoBehaviour`/frame real (ex: `TickController` acelerando corretamente ao longo do tempo, persistência sobrevivendo a um "fechar app à força" simulado).

---

## Boas práticas adicionais (comparação com projetos reais de Unity)

Pontos que não estavam no design original, mas que valem a pena adotar desde o início por serem prática comum em jogos Unity de grid/arcade:

- **Object pooling pros segmentos da cobra.** Cada `Advance()` que faz a cobra crescer, hoje modelado como lista de `GridPosition`, não deve gerar/destruir um `GameObject` novo por segmento a cada tick — isso gera picos de GC perceptíveis em mobile. Um pool simples de segmentos reaproveitados resolve isso sem complexidade extra.
- **ScriptableObject Event Channels** para comunicação entre `Core` e `UI` (ex: "partida terminou", "buff aplicado") em vez de `UI` chamar `FindObjectOfType` ou `Core` ter uma referência direta a um Canvas. Mantém a regra de dependência da seção de Assembly Definitions realmente unidirecional.
- **`[SerializeField] private` em vez de campos públicos** nos `MonoBehaviour` de `UI`/`Input`, com os perfis de IA e configurações expostos só via `ScriptableObject` — reduz o que aparece exposto sem querer no Inspector.
- **Evitar `GetComponent`/`FindObjectOfType` dentro de `Update`** — é exatamente o tipo de erro que o Microsoft.Unity.Analyzers (já adicionado ao CI, ver `DECISIONS.md`) pega automaticamente, mas vale ter isso em mente ao já escrever o código pela primeira vez.

---

## Decisões técnicas abertas (resolver antes de codar F1)

Itens que este documento **não resolve sozinho** — precisam de uma decisão explícita (e um novo registro em `DECISIONS.md`) antes de virar código:

1. **Regra de colisão cabeça-a-cabeça simultânea** (`OpponentHeadOnHead`): empate (as duas cobras morrem), ou desempate por algum critério (tamanho, quem entrou na célula primeiro no sub-tick)?
2. **Confirmar `GRID_WIDTH`/`GRID_HEIGHT` reais** do projeto original — todo o resto deste documento assume 24x24.
3. **Confirmar a curva real de aceleração do tick** (180ms → 135ms) — linear ao longo da partida, ou em degraus a intervalos fixos?
4. **Efeito exato da "maçã bufada"** — o nome sugere um crescimento maior que o da comida normal, ou um buff temporário; isso muda a assinatura de `IItem.OnConsumed`.
5. **A dificuldade da IA afeta a velocidade de tick dela**, ou só a qualidade das decisões (via `AIDifficultyProfile`)? Afeta diretamente se `TickController` precisa ser por-cobra ou único pra partida toda.

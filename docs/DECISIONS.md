# Decisões Técnicas

Registro de decisões importantes do projeto, para evitar perda de contexto e facilitar manutenção futura.

> Todas as entradas abaixo foram registradas em 2026-09-11 (data de criação deste documento), mesmo quando a decisão em si foi tomada em conversas anteriores. Ajuste as datas individualmente se quiser granularidade histórica real.

---

### Data: 2026-09-11

**Decisão:**
Criar um repositório separado (fork) para a versão solo Android, mantendo o Snake Duel original como versão canônica multiplayer.

**Motivo:**
Ganhar experiência real de publicação no Google Play sem comprometer o projeto principal. Ideias validadas no fork (IA com persona, modo Sobrevivência Infinita, conquistas) podem retornar ao projeto original depois de testadas.

**Alternativas consideradas:**
- Criar uma branch dentro do próprio repositório original
- Adaptar o projeto original diretamente, sem separar os dois

---

### Data: 2026-09-11

**Decisão:**
Remover completamente o Mirror Networking (camada `Network`) do fork solo, em vez de apenas desativá-lo.

**Motivo:**
O modo solo nunca dependeu de rede. Manter o pacote infla o build sem necessidade e mantém superfície de risco (dessincronia, CGNAT, port forwarding) que não se aplica a essa versão.

**Alternativas consideradas:**
- Manter o Mirror desativado via feature flag, pensando numa eventual v2 multiplayer
- Deixar como está e simplesmente não chamar o código de rede

---

### Data: 2026-09-11

**Decisão:**
Migrar de dois tabuleiros lado a lado para um único tabuleiro compartilhado (estilo Tron / light cycles), com jogador e IA no mesmo grid toroidal.

**Motivo:**
Dois tabuleiros lado a lado não cabem bem em celular na orientação vertical. Um tabuleiro único também cria interação direta (colisão) entre jogador e IA, em vez de só comparação de pontuação.

**Alternativas consideradas:**
- Manter os dois tabuleiros lado a lado, aceitando o tamanho reduzido
- Empilhar os dois tabuleiros verticalmente em vez de lado a lado

---

### Data: 2026-09-11

**Decisão:**
Tratar o fallback do A* para grid sem caminho válido como item bloqueador de lançamento, não como melhoria futura.

**Motivo:**
Esse risco já era conhecido e não mitigado no projeto original. No modo solo, a IA é o modo principal do jogo — travar ou lançar exceção nesse caso é inaceitável.

**Alternativas consideradas:**
- Aceitar o risco e tratar como bug conhecido pós-lançamento
- Usar fallback simplificado (movimento aleatório entre vizinhos livres) em vez de buscar a célula livre mais próxima

---

### Data: 2026-09-11

**Decisão:**
Definir `minSdkVersion`/`targetSdkVersion` para API level 36 (Android 16).

**Motivo:**
Requisito vigente da Google Play para novos apps a partir de 31/08/2026 — não é uma escolha de engenharia, é compliance obrigatório de loja.

**Alternativas consideradas:**
- Nenhuma alternativa viável identificada; é requisito da plataforma

---

### Data: 2026-09-11

**Decisão:**
Recrutar os 12 testadores exigidos pelo teste fechado combinando comunidades de troca de créditos (ex: Testers Community) com, se necessário, serviços pagos de teste.

**Motivo:**
Atende ao requisito da Google Play para contas pessoais criadas após 13/11/2023 sem depender exclusivamente da rede pessoal do desenvolvedor. Engajamento real de cada testador (não só instalar) reduz o risco de rejeição por "engajamento insuficiente".

**Alternativas consideradas:**
- Recrutar sozinho via redes sociais e comunidades de desenvolvedores
- Contratar apenas um serviço pago dedicado, sem usar comunidades de troca

---

### Data: 2026-09-11

**Decisão:**
Usar Play App Signing e manter backup criptografado da upload keystore em pelo menos dois locais diferentes.

**Motivo:**
Perder a upload keystore impede publicar qualquer atualização futura do mesmo app na Google Play — risco silencioso e catastrófico se não for tratado desde o primeiro build.

**Alternativas consideradas:**
- Gerenciar a assinatura manualmente, sem Play App Signing
- Manter só uma cópia de backup da keystore

---

### Data: 2026-09-11

**Decisão:**
Adotar a direção de arte "grade neon-arcade minimalista" (fundo escuro, cores saturadas, sem elementos de personagem complexos) como estilo visual principal.

**Motivo:**
Melhor equilíbrio entre diferenciação frente aos concorrentes (poucos jogos de snake em grid usam essa estética — a maioria é `.io` de movimento livre ou remake clássico simples) e escopo de arte viável para um desenvolvedor solo no primeiro jogo.

**Alternativas consideradas:**
- Cartoon estilizado (mais comum nos concorrentes `.io`, porém exige produção de arte bem maior)
- Retrô pixelado (barato de produzir, porém nicho mais saturado de remakes simples)

---

### Data: 2026-09-11

**Decisão:**
Usar contraste complementar quente/frio para diferenciar as cobras (azul para o jogador, laranja para a IA) e cor fixa por categoria semântica de item (verde = comida, âmbar = maçã bufada, roxo = orbe de habilidade, vermelho = bomba).

**Motivo:**
Facilita reconhecimento instantâneo num jogo de grid rápido. Cor por categoria fixa (em vez de decorativa) reduz carga cognitiva do jogador em qualquer tela do jogo.

**Alternativas consideradas:**
- Diferenciar cobras só por padrão/textura, sem depender de cor
- Deixar a paleta de itens livre, sem convenção fixa por categoria

---

### Data: 2026-09-11

**Decisão:**
Automatizar o build Android via GitHub Actions usando `game-ci/unity-builder`, e remover os workflows herdados de template genérico baseados em npm (ESLint, TypeScript Check, Security Audit).

**Motivo:**
O projeto é Unity/C#, não Node/TypeScript — os workflows de npm não checavam nada de real nesse stack e dariam falsa sensação de cobertura.

**Alternativas consideradas:**
- Adaptar os workflows de npm pra "quase funcionar" no lugar de C#
- Não ter nenhum CI de build automatizado por enquanto

---

### Data: 2026-09-11

**Decisão:**
Adicionar Microsoft.Unity.Analyzers (Roslyn analyzers) como checagem estática de código C#, aplicado via `dotnet build -warnaserror` num job dedicado (`code-check`) no CI.

**Motivo:**
Equivalente funcional ao ESLint/tsc removidos, mas específico para armadilhas comuns de Unity/C# (uso incorreto de coroutines, `GetComponent` em `Update`, etc.), em vez de um linter genérico de C#.

**Alternativas consideradas:**
- Rodar os analyzers só localmente na IDE, sem enforcement automático no CI
- Usar apenas StyleCop, sem os analyzers específicos de Unity

---

### Data: 2026-09-11

**Decisão:**
Remover o bloco `npm` do `dependabot.yml`, manter o bloco `github-actions`, e deixar um bloco `nuget` comentado como opcional.

**Motivo:**
O projeto não usa npm. Atualização de GitHub Actions é independente de stack e vale para qualquer projeto. NuGet só passaria a ser relevante se o projeto adotar pacotes .NET via NuGetForUnity no futuro.

**Alternativas consideradas:**
- Apagar o arquivo `dependabot.yml` inteiro (perderia a atualização automática das próprias Actions)

---

<!-- Copie o bloco acima para cada nova decisão -->

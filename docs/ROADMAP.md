# Roadmap

Planejamento macro do projeto, por fases. O detalhamento de cada item vive nas Issues (ver `issues.csv` e convenção `[Fase][Categoria] Título`).

- [ ] **F0 — Setup**: configuração inicial do repositório, ferramentas e ambiente *(docs e CI já modelados; projeto Unity em si ainda não versionado neste repositório)*
- [ ] **F1 — Core**: regras e lógica principal do domínio *(fork do Core original, fallback do A*, `BombSystem` em grid compartilhado)*
- [x] **F2 — Network**: comunicação, API ou sincronização (se aplicável) *(não aplicável a este fork — Mirror removido por decisão, ver `DECISIONS.md`)*
- [ ] **F3 — UI**: interface e experiência do usuário *(tabuleiro único compartilhado, controles touch, HUD, paleta de cores)*
- [ ] **F4 — Mobile**: adaptação/publicação mobile *(build Android, keystore, target API 36, teste fechado)*
- [ ] **F5 — Release**: testes finais, documentação e publicação *(produção na Google Play)*

> Sem detalhamento excessivo aqui — cada fase vira Issues específicas conforme o desenvolvimento avança.

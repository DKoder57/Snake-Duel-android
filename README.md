# Snake Duel: Solo

**Snake competitivo em grid toroidal contra uma IA com pathfinding A* real, feito para Android — sem nenhuma dependência de rede.**

[![Build](https://img.shields.io/github/actions/workflow/status/DKoder57/snake-duel-solo/build.yml?label=build)](../../actions)
[![License](https://img.shields.io/github/license/DKoder57/snake-duel-solo)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/DKoder57/snake-duel-solo)](../../commits/main)

## 📑 Sumário

- [Sobre](#-sobre)
- [Features](#-features)
- [Tecnologias](#-tecnologias)
- [Como rodar](#-como-rodar)
- [Estrutura do projeto](#-estrutura-do-projeto)
- [Roadmap](#-roadmap)
- [Contribuindo](#-contribuindo)
- [Licença](#-licença)

## 📖 Sobre

Fork solo do [Snake Duel](https://github.com/DKoder57/Snake-Duel) original, adaptado para uma partida single-player contra IA e publicação na Google Play. Existe pra ganhar experiência real de primeiro lançamento — o projeto original continua como versão canônica (multiplayer host-and-join por IP), e ideias validadas aqui podem voltar pra lá depois de testadas nesta versão mais enxuta.

Documentação técnica completa (arquitetura, avaliação de design, decisões) vive em [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) e [`docs/DESIGN_REVIEW.md`](docs/DESIGN_REVIEW.md) — aqui fica só o essencial pra começar.

## ✨ Features

- [x] Modo Solo contra IA (herdado do Modo 1 do projeto original: grid toroidal, itens, buff/debuff)
- [x] Pathfinding A* real para a IA
- [ ] Fallback do A* para grid sem caminho válido
- [ ] Controles touch (swipe + D-pad virtual)
- [ ] Níveis de dificuldade de IA com persona
- [ ] Modo Sobrevivência Infinita
- [ ] Build Android publicável (ver [Roadmap](#-roadmap))

## 🛠️ Tecnologias

| Camada           | Tecnologia |
|------------------|------------|
| Engine           | Unity (C#) |
| Plataforma       | Android (AAB, target API 36) |
| Persistência     | Armazenamento local (sem backend) |
| Infra / Deploy   | Google Play Console |
| CI/CD            | GitHub Actions ([workflows](.github/workflows)) |

## ▶️ Como rodar

```bash
# Clonar
git clone https://github.com/DKoder57/snake-duel-solo.git
cd snake-duel-solo
```

Abra a pasta do projeto no Unity Hub (versão do Editor definida em `ProjectSettings/ProjectVersion.txt`), aguarde a importação dos assets e rode a cena principal em `Assets/Scenes`.

Para gerar o build Android: `File > Build Settings > Android > Switch Platform`, depois `Build` (ou `Build App Bundle` para gerar o `.aab` de publicação).

## 📂 Estrutura do projeto

```text
.
├── .github/          # CI (GitHub Actions)
├── Assets/           # Código-fonte, cenas e assets do Unity
├── ProjectSettings/  # Configurações do projeto Unity
├── docs/             # Arquitetura, roadmap e avaliação de design
└── issues.csv        # Backlog inicial, importado como GitHub Issues
```

## 🗺️ Roadmap

Planejamento macro por fases (F1–F7) em [`issues.csv`](issues.csv), do fork do Core até a publicação em produção. O detalhamento de cada item vira Issue no GitHub.

## 🤝 Contribuindo

Projeto pessoal/solo — sugestões e issues são bem-vindas, mas não há processo formal de contribuição externa no momento.

## 📄 Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE) para mais detalhes.

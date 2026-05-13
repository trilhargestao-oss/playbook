# Playbook Técnico Trilhar

Princípios, arquitetura, padrões e processo pra construir software robusto na Trilhar.

**Versão:** 1.0 (2026-05-13)
**Origem:** destilação de lições aprendidas durante a construção do Kenji
(plataforma de análise financeira IA), de janeiro a maio de 2026.

## Estrutura

- **[PLAYBOOK.md](./PLAYBOOK.md)** — Documento principal. 13 seções cobrindo
  filosofia, arquitetura, code quality, testing, security, frontend, processo
  e anti-patterns.

## Como usar

**Ao iniciar projeto novo:** lê o PLAYBOOK inteiro (~30-40 min). Não pula
seção achando que "minha stack é diferente" — princípios são transferíveis.
Os exemplos do Kenji são ilustrativos; aplica adaptado ao contexto.

**Ao tomar decisão técnica relevante:** consulta a seção pertinente. Se a
decisão diverge do playbook, documenta o porquê em ADR do projeto.

**Ao identificar pattern novo reusável OU anti-pattern recorrente:** abre
PR neste repo atualizando seção pertinente + bumps CHANGELOG (quando
existir). Playbook é documento vivo.

## Audiência

- Desenvolvedores Trilhar (Cauê + outros que entrarem)
- Claude (planner em chat, executor em Claude Code, agentes futuros)
- Consultores externos contratados pra projetos pontuais

## Histórico

- **v1.0 (2026-05-13):** Versão inicial. Destilação de lições do Kenji
  (jan-mai 2026): 16+ sub-colas Two-Gate, 53 D-IAs Fase 4, 13 D-Refino,
  5 ADRs consolidadores, padrão Promise.allSettled em 3 contextos,
  security audit + 2FA setup, dotenv override, git commit -F.

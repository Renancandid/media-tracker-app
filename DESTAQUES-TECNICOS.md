# Destaques técnicos · Estante

A **Estante** é um app mobile multiplataforma (iPhone e web) com **banco local que funciona offline**, **sincronização em nuvem entre aparelhos** e **testes automatizados**.

Este documento mostra as decisões de arquitetura e os problemas que precisaram ser resolvidos.

| | |
|---|---|
| **App** | React Native + Expo (SDK 58), TypeScript, Expo Router |
| **Dados** | SQLite no aparelho (fonte principal) + Supabase (sincronização) |
| **Segurança** | Supabase Auth, Row Level Security, Face ID |
| **Testes** | Jest com SQLite real (sem mocks de banco) |

---

## Sumário

1. [Arquitetura](#1-arquitetura)
2. [Offline primeiro e sincronização](#2-offline-primeiro-e-sincronização)
3. [Segurança](#3-segurança)
4. [Banco de dados](#4-banco-de-dados)
5. [Acessibilidade e design system](#5-acessibilidade-e-design-system)
6. [Testes e qualidade](#6-testes-e-qualidade)
7. [Build e distribuição sem Mac](#7-build-e-distribuição-sem-mac)
8. [Decisões registradas](#8-decisões-registradas)

---

## 1. Arquitetura

```
┌──────────────── App (React Native / Expo) ────────────────┐
│  Telas (Expo Router)  →  Modelos (regras puras, TypeScript) │
│            ↓                                                │
│   SQLite local  ←→  Motor de sincronização ─────────────────┼──→  Supabase
│                                                             │     (Auth + Postgres + RLS)
└─────────────────────────────────────────────────────────────┘
```

- **Organização por funcionalidade** (`features/`), com **regras de negócio em funções puras** (`models/`): estatísticas, progresso, busca e status. Isso permite testá-las sem interface.
- **Injeção de dependências** no sincronizador: o servidor real pode ser trocado por um falso nos testes.
- **Modo de demonstração:** sem conta configurada, o app abre com dados de exemplo e funciona inteiro sem login e sem internet.

---

## 2. Offline primeiro e sincronização

**Problema:** o app precisa funcionar sem internet (no metrô, no avião) e, mesmo assim, manter os dados iguais em vários aparelhos.

**Solução:** o **SQLite local é a fonte principal**. O Supabase só transporta as mudanças entre aparelhos. A sincronização (`services/sync/engine.ts`) foi escrita do zero:

- **Envio só do que mudou:** cada linha alterada recebe uma marca `dirty`. O envio é feito em lotes de 200, sem chaves repetidas no mesmo lote.
- **Recebimento incremental por cursor:** o app pede ao servidor *"o que mudou desde X"*, em páginas de 500, usando um carimbo de tempo **do servidor** (`server_updated_at`), definido por um trigger no Postgres. Assim, relógios errados nos aparelhos não afetam a sincronização.
- **Sem perder gravações simultâneas:** o cursor volta 5 segundos a cada rodada. Linhas gravadas no mesmo instante não escapam, e reaplicar uma linha não causa efeito colateral (operação idempotente).
- **Sem perder edição feita durante o envio:** uma linha só é marcada como enviada se não mudou desde que foi lida:
  ```sql
  UPDATE media_items SET dirty = 0 WHERE id = ? AND updated_at = ?
  ```
- **Resolução de conflitos:** prevalece a versão mais recente (*last-write-wins* por `updated_at`). Exclusões são marcadas com `deleted_at` (*soft delete*) e também sincronizam.
- **Ordem de dependência:** itens primeiro, porque listas e histórico apontam para eles.
- **Troca de conta:** se outra pessoa entra no mesmo aparelho, os dados da conta anterior são apagados do aparelho, mas continuam salvos na conta dela.
- **Primeiro login com dados no aparelho:** o app pergunta se mantém os dados existentes ou começa do zero.

---

## 3. Segurança

**No Supabase:**
- **Row Level Security** em todas as tabelas: cada usuário só lê e altera os próprios dados.
- O `user_id` é preenchido **pelo servidor** (`auth.uid()`) num trigger, então o app não consegue gravar dados em nome de outra pessoa.
- **Cadastro só por convite**, com confirmação de e-mail, criação de senha pelo link do convite e "esqueci a senha".
- Na tela de conta: alterar senha, sair e **excluir a conta**.

**No aparelho:**
- **Bloqueio opcional com Face ID**.
- Chaves e endereços ficam num `.env` fora do Git.

---

## 4. Banco de dados

- **Migrações versionadas** no SQLite com `PRAGMA user_version`. Regra seguida: *nunca editar uma migração publicada, só acrescentar uma nova*. Assim, quem já tem o app instalado atualiza sem perder dados.
- **Índices** nos acessos mais comuns, como tipo + status + prioridade, e usuário + data do servidor no Supabase.
- **Histórico de atividades** (`activity_log`), que alimenta gráficos, metas e a sequência de dias de leitura.
- **Script SQL idempotente** no Supabase: pode ser executado de novo sem quebrar nada.
- **Backup** em JSON (com versão de formato e validação de cada item ao importar) e em CSV com `;` e BOM UTF-8, para abrir certo no **Excel em português**. Na importação, prevalece o registro mais recente.

---

## 5. Acessibilidade e design system

- **Design system próprio:** tokens de cor, tipografia e espaçamento, e cerca de 20 componentes (Button, Card, Sheet, Toast, Skeleton, StarRating, SegmentedControl...), com uma tela de **catálogo** para visualizá-los.
- **Teste automatizado de contraste WCAG AA** na paleta (`contrast.test.ts`): se uma cor nova deixar o texto ilegível, o teste falha.
- **Daltonismo:** as cores dos gráficos foram ajustadas para continuarem distinguíveis com deuteranopia. Um gráfico de rosca foi trocado por barras ordenadas porque reprovou nesse critério.
- **Movimento reduzido:** as animações respeitam a configuração "Reduzir Movimento" do sistema.
- **Gráficos próprios em SVG** (barras, linha e anéis), em vez de uma biblioteca pesada.

---

## 6. Testes e qualidade

| O que é testado | Arquivo |
|---|---|
| Motor de sincronização | `sync.test.ts` |
| Banco local e migrações | `localdb.test.ts` |
| Estatísticas e metas | `stats.test.ts` |
| Progresso de leitura e episódios | `progress.test.ts` |
| Busca, filtros e ordenação | `query.test.ts` |
| Regras de status | `status.test.ts` |
| Exportar e importar backup | `backup.test.ts` |
| Contraste de cores (WCAG AA) | `contrast.test.ts` |

O banco usado nos testes é um **SQLite real em Node** (`test-utils/nodeSqlite.ts`), não um mock. Por isso as consultas e as migrações são testadas de verdade. Além dos testes, `tsc --noEmit` (tipos) e ESLint.

---

## 7. Build e distribuição sem Mac

- O app foi desenvolvido **inteiramente no Windows**. O build para iPhone é feito **na nuvem** com o **EAS Build**, com perfis de desenvolvimento, prévia e produção, e está pronto para o **TestFlight** via EAS Submit.
- **Versão web** publicada no Cloudflare. No iPhone, ela pode ser instalada na tela inicial como app (PWA, com ícone e manifesto).
- **SQLite também na web**, via WebAssembly. Isso exigiu configurar os cabeçalhos COOP/COEP no servidor e o Metro para aceitar arquivos `.wasm`.

---

## 8. Decisões registradas

O projeto mantém um registro do que mudou em relação ao planejamento original e **por quê** (`docs/PROGRESSO.md`). Alguns exemplos:

| Planejado | Decidido | Motivo |
|---|---|---|
| Swift + SwiftUI | React Native + Expo | permite build na nuvem, sem Mac |
| SwiftData + CloudKit | SQLite + Supabase | funciona offline e sincroniza fora do ecossistema Apple |
| Biblioteca de gráficos | gráficos próprios em SVG | só eram necessários 3 tipos; evita uma dependência pesada |
| Rosca de gêneros | barras ordenadas | a rosca reprovou no validador de daltonismo |
| Sign in with Apple | e-mail e senha, cadastro por convite | necessário para sincronizar; login com Apple/Google fica para depois |

O desenvolvimento foi feito com **Claude Code**. O meu trabalho foi definir o produto, dividi-lo em fases, tomar e registrar essas decisões, testar no aparelho e revisar o resultado.
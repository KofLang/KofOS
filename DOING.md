# DOING.md — coordenação multi-agente (quem faz o quê)

> **Regra obrigatória para agentes (IA ou humano):**
> 1. **Antes** de começar qualquer trabalho: leia este arquivo.
> 2. Se o item já tem **dono + estado `EM CURSO`**, não toque.
> 3. Ao **reivindicar**: edite aqui **no mesmo commit** que começa o trabalho.
> 4. A **cada commit**, atualize sua linha.
> 5. Ao **concluir**: `FEITO` com data + commit + prova (teste verde).
> 6. Itens `EM CURSO` por mais de uma sessão → abandone com nota.
> 7. **Modo autônomo:** se o turno vai acabar, a ÚLTIMA coisa escrita aqui é
>    a linha **"PRÓXIMO PASSO"** abaixo (tarefa exata + arquivo + prova).

Estados: `ABERTO` · `EM CURSO` · `FEITO` · `BLOQUEADO`.

---

## PRÓXIMO PASSO (re-dispacho lê isto)

**PRÓXIMO PASSO (07/09)**: userland desktop + apps
→ HAL → memory → storage → scheduler → VFS → syscalls → userland (init + shell).
Prova: `kof run Main.kf` imprime a sequência de boot completa até "done" + `kof
test boot_test.kf` (2 testes PASS). Gap HW001 documentado em AGENTS.md.
PRÓXIMO: **PORT-SCHED** — preempção real por timer (`kof.time.interval`),
timeslice, bloqueio/espera (waitable), round-robin com prioridade em
`kernel/scheduler.kf`. Prova: `kof test` verifica que 2+ processos alternam
ticks e que um processo bloqueado não roda.

---

## Em curso agora

| Gap/Item | Estado | Dono | Branch | Arquivos principais | Notas |
|---|---|---|---|---|---|
| **PORT-TODO** — fundação (boot + HAL + Main) | `FEITO` | agente-kofos | `main` | `Main.kf`, `kernel/boot.kf`, `kernel/hal/hal.kf`, `kernel/kernel_init.kf`, `kernel/globals.kf`, `kernel/process.kf`, `kernel/scheduler.kf`, `userland/userland.kf`, `boot_test.kf` | 07/09: boot→kernel_entry→init→shell→done roda. 2 testes PASS. Mecanismo de import descoberto: `import <dir>.<arquivo>` carrega `<dir>/<arquivo>.kf`; testes E2E na raiz do módulo. |
| **PORT-SERVICES** — microkernel (init/video/input/fs/storage/console) | `ABERTO` | — | — | `kernel/services/` | com gap: video.c 2000+, audio.c 10000+ linhas; PRÓXIMO: userland desktop + apps
| **PORT-SCHED** — scheduler com preempção real | `ABERTO` | — | — | `kernel/scheduler.kf` (a remover — gap bytecode putfield Int) | PRÓXIMO: reimplementar scheduler em Kof com `kof.time.interval`, timeslice, algoritmo de score (prioridade/afinidade/wake-boost), bloqueio/espera (waitable), round-robin com prioridade. Pendência: gap de field-assign de `Int` no compilador Kof (regra R6 — nunca silencioso). |
| **PORT-MEM** — heap, páginas, arenas | `ABERTO` | — | — | `kernel/memory.kf` | após scheduler |
| **PORT-SYSCALL** — syscalls + IPC + waitable | `ABERTO` | — | — | `kernel/syscall.kf`, `kernel/ipc.kf` | após mem |
| **PORT-VFS** — VFS + ramfs + AppFS | `ABERTO` | — | — | `kernel/vfs.kf` | após services |
| **PORT-USERLAND** — shell + desktop + apps | `ABERTO` | — | — | `userland/` | após VFS |
| **PORT-E2E** — boot→shell→desktop + docs + branding | `ABERTO` | — | — | `README.md`, `docs/` | final |

## Concluídos recentemente

| Gap/Item | Estado | Dono | Data | Prova |
|---|---|---|---|---|
| Toolchain Kof + análise do VibeOS | `FEITO` | agente-kofos | 07/09 | `hello.kf` roda no JVM; relatório do kernel no AGENTS.md |

## Abertos (livres pra pegar)

| Gap/Item | Prioridade | Escopo | Notas |
|---|---|---|---|
| **HW001** | alta | Gap de plataforma: kernel bare-metal abstraído | decisão de design, não silencioso |

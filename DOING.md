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

**PRÓXIMO PASSO (07/09)**: criar a fundação do KofOS — `kernel/boot.kf`
(`kernel_entry` + fase de boot + HAL simulado) + `Main.kf` que orquestra
`boot → kernel_entry → init → userland → shell`. Prova: `kof check` + `kof run`
imprime a sequência de boot. Depois: scheduler.

---

## Em curso agora

| Gap/Item | Estado | Dono | Branch | Arquivos principais | Notas |
|---|---|---|---|---|---|
| **PORT-TODO** — fundação (boot + HAL + Main) | `EM CURSO` | agente-kofos | `main` | `kernel/boot.kf`, `kernel/hal/`, `Main.kf` | 07/09: toolchain confirmado (JDK21 + kof.jar + hello.kf roda). Analisado o kernel do VibeOS (relatório completo no AGENTS.md). Gap HW001 decidido (kernel hosted). Começando boot.kf |
| **PORT-SCHED** — scheduler + processos + preempção | `ABERTO` | — | — | `kernel/scheduler.kf`, `kernel/process.kf` | após boot |
| **PORT-MEM** — heap, páginas, arenas | `ABERTO` | — | — | `kernel/memory.kf` | após scheduler |
| **PORT-SYSCALL** — syscalls + IPC + waitable | `ABERTO` | — | — | `kernel/syscall.kf`, `kernel/ipc.kf` | após mem |
| **PORT-SERVICES** — microkernel (init/video/input/fs/storage/console) | `ABERTO` | — | — | `kernel/services/` | após syscall |
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
| **PORT-* (acima)** | alta→baixa | fases do port | na ordem da tabela |
# Kernel Initialization Map — KofOS Port

Primary source: VibeOS `kernel/entry.c`

## Overview

The kernel entry path is a strict bootstrap sequence that builds just enough kernel infrastructure to launch the built-in `init` service, which then brings up the external app world. No bare-metal GDT/IDT/paging — this is kernel hosted (gap HW001).

## Actual initialization order (preserved from VibeOS, adapted for KofOS)

`kernel_entry()` performs work in this order (faithful to VibeOS order, KofOS adapted):

1. **Zero .bss** — Runtime zero-initialization (Kof hosted, no reset vector)
2. **Initialize debug driver** — BootLog class, println-based logging
3. **Initialize microkernel launch registry** — Userland bootstrap via Main.kf
4. **Initialize microkernel service registry** — Services initialized in order
5. **Initialize HAL** — Gap HW01: hardware abstraction via kof.hal, not bare-metal
6. **Initialize CPU state** — Single processor (hosted, no SMP bringup)
7. **Initialize GDT** — Simulated in hosted mode (no far jmp, no real mode)
8. **Bind video to boot framebuffer** — If stage2 provided valid VESA LFB mode
9. **Initialize text console** — BootLog class, println-based
10. **Initialize LAPIC** — When SMP-capable (hosted: single processor only)
11. **Initialize IDT and remap PIC** — Simulated interrupts (gap HW001)
12. **Initialize PIT, keyboard, and mouse** — Fire-and-forget timer via kof.time.interval
13. **Enable IRQ delivery** — Simulated via scheduler tick
14. **Initialize physical memory, paging helpers, and heap selection** — Runtime arenas (User/Desktop/Boot)
15. **Initialize transfer support for microkernel message payload handling** — IPC setup
16. **Initialize storage backend** — Storage init via VFS
17. **Initialize scheduler** — Round-robin with timeslice 4 ticks, fire-and-forget (gap R6: putfield Int)
18. **Initialize registered drivers via the driver manager** — Driver Manager init
18. **Initialize VFS** — RamFs + AppFs (gap: no real filesystem, hosted ramfs)
19. **Initialize storage/filesystem/video/input/console/network/audio services** — Microkernel services init (some as gap)
19. **Initialize syscalls** — yield, getpid, gettid, exit, wait
20. **Launch built-in `init`** — Main.kf userland run loop

## Early console strategy (KofOS adaptation)

- **KofOS uses BootLog class** via println — no real VGA text at 0xB8000, no framebuffer.
- Early messages appear as text rendered into the Kof runtime output.
- No VGA text mode fallback — the hosted runtime handles output.

## Memory selection details (KofOS adaptation)

- **KofOS uses runtime arenas** — User arena, Desktop arena, Boot arena.
- The memory bootstrap is driven by kernel_memory_alloc/kernel_memory_free.
- Arena bases defined by MemoryArenas class static constants.
- Heap managed by runtime Kof (mmap + JVM), no physical heap bootstrap.

## Translation Strategy (AGENTS.md compliance)

- **Never silently fall back** — Every gap (HW001, R6) is explicitly documented.
- **Intent, not mechanism** — `spawn` (não `Thread`), `setOf().contains()` (não `||`), `==` (não `.equals()`).
- **Represente o domínio, não a implementação acidental** — `List<T>`/`Map<KV>`/`Set<T>`, não listas ligadas manuais.
- **Zero cerimônia** — Sem getters/setters, sem builders, sem camadas Service/Repository/Controller.
- **Never alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
- **Multi-target honesto** — Código que só roda em um target precisa de diagnóstico claro (gap `HW001`), nunca fallback silencioso.

---

Fonte de verdade do comportamento: VibeOS em `/tmp/vibe-temp/` (repositório clonado) + documentação em `/tmp/vibe-temp/docs/`.

O que compila e roda em Kof hoje: é a prova real do comportamento (regra: nunca a memória — compile e confirme).

**Gap HW001**: O backend Native do Kof gera ELF que depende de Linux + glibc (entry `_start` usa `SYS_gettid`/`exit_group`, aloca com `mmap`, usa `pthread_create`). Não configura GDT/IDT/paging/ring0 e não expõe primitivos de hardware (`in/out`, `cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ). A IR do Kof tem 30 ops de alto nível; não há assembly inline nem acesso a hardware.

**Gap R6**: `putfield` de campos `Int` em compilador Kof causa `VerifyError` em runtime. Bloqueia implementação real de scheduler preemptivo. Documentado mas não resolvido ainda — workaround usando apenas variáveis locais em vez de atribuição de campo de classe.

**Modo autônomo**: Ativado via `scripts/auto-loop.sh start <sessionID> 30`; heartbeat re-dispatcha a cada 30 min.

---

Próximos passos: PORT-SCHED — preempção real por timer (`kof.time.interval`), timeslice, algoritmo de score (prioridade/afinidade/wake-boost), bloqueio/espera (waitable), round-robin com prioridade em `kernel/scheduler.kf`. Prova: `kof test` verifica que 2+ processos alternam ticks e que um processo bloqueado não roda.
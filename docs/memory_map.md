# Memory and Disk Layout — KofOS Port

Primary source: VibeOS `memory_map.md` (adapted for KofOS hosted, gap HW001)

## Overview

The VibeOS uses a BIOS-level memory layout and disk layout to bootstrap the kernel and launch apps. In KofOS hosted (gap HW001), the BIOS/bootloader stages are replaced by the Kof runtime boot (Main.kf → kernel_entry). The memory layout concepts are preserved, but the implementation differs: no physical memory map from BOOTINFO, no MBR partitions, no FAT32 boot world — instead, the runtime Kof manages memory via mmap and the app world is launched from the runtime filesystem.

## 1. Low-memory boot layout (KofOS adaptation)

Endereços físicos tradicionais importam no VibeOS pois cada stage do boot compartilha o primeiro megabyte. No KofOS hosted, esses endereços são irrelevantes pois não há boot stages em hardware real. O que substitui:

- **Endereços virtuais fixos** — User arena, Desktop arena, Boot arena definidos por MemoryArenas class constants.
- **Runtime heap** — Gerenciado por `kernel_memory_alloc()` / `kernel_memory_free()`, sobre o heap do runtime Kof (mmap + JVM).
- **BootLog buffer** — String buffer em vez de buffer de trace em 0x1000.

A tabela abaixo compara o layout original com o KofOS:

| Endereço/VibeOS | KofOS equivalent |
|---|---|
| `0x0000-0x03FF` | BIOS IVT — irrelevante em hosted |
| `0x0400-0x04FF` | BIOS Data Area — irrelevante |
| `0x7C00-0x7DFF` | VBR / stage 1 load address — substituído por Main.kf entry |
| `0x8D00-0x8D83+` | `BOOTINFO` — substituído por BootGlobals class |
| `0x9000-...` | stage 2 image — substituído por kernel_entry em Main.kf |
| `0x10000` | kernel physical load base — substituído por MemoryArenas base addresses |
| `0x10000000` | main app arena — substituído por MemoryArenas.USER_ARENA_START |
| `0x14000000` | desktop app arena — substituído por MemoryArenas.DESKTOP_ARENA_START |
| `0x18000000` | boot app arena — substituído por MemoryArenas.BOOT_ARENA_START |

## 2. Kernel memory policy (KofOS adaptation)

No VibeOS, o kernel consome a partir do maior intervalo utilizável do BOOTINFO e faz reservas para:
- kernel/core low memory até o fim da imagem do kernel
- reserva de stack do userland
- bytes de guarda do heap
- range do framebuffer do VESA, quando modo VESA válido fornecido pelo stage 2
- três arenas de 16 MiB para apps, quando um intervalo livre grande o suficiente existe

No KofOS hosted, o heap é gerenciado pelo runtime Kof. As arenas virtuais são definidas por MemoryArenas class e não dependem de BOOTINFO. O restante maior região livre torna-se o heap do runtime.

## 3. App virtual arenas (KofOS adaptation)

O tempo de execução de apps modulares espera três arenas virtuais fixas:

| Endereço virtual | Propósito | MemoryArenas constant |
|---|---|---|
| `0x10000000` | arena principal de apps | USER_ARENA_START |
| `0x14000000` | arena de apps desktop | DESKTOP_ARENA_START |
| `0x18000000` | arena de apps boot | BOOT_ARENA_START |

Cada arena é de 16 MiB (16 * 1024 * 1024 bytes).

Esses não são constantes arbitrárias em docs apenas. Elas são parte da ABI de tempo de execução do AppFS, definidas em `lang/include/vibe_app.h` e usadas via `MemoryArenas` class em KofOS.

## 4. Estratégia de tradução (AGENTS.md compliance)

- **Never alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
- **Multi-target honesto** — Código que só roda em um target precisa de diagnóstico claro (gap `HW001`), nunca fallback silencioso.
- **Intent, not mechanism** — `spawn` (não `Thread`), `setOf().contains()` (não `||`), `==` (não `.equals()`).
- **Represente o domínio, não a implementação acidental** — `List<T>`/`Map<K,V>`/`Set<T>`, não listas ligadas manuais.
- **Zero cerimônia** — Sem getters/setters, sem builders, sem camadas Service/Repository/Controller.
- **Gap R6 documentado** — `putfield` de campos `Int` em compilador Kof causa `VerifyError`. Workaround: usar variáveis locais em vez de atribuição de campo de classe.

---

Fonte de verdade do comportamento: VibeOS em `/tmp/vibe-temp/` (repositório clonado) + documentação em `/tmp/vibe-temp/docs/`.

O que compila e roda em Kof hoje: é a prova real do comportamento (regra: compile e confirme).

**Gap HW001**: O backend Native do Kof gera ELF que depende de Linux + glibc. Não configura GDT/IDT/paging/ring0 e não expõe primitivos de hardware (`in/out`, `cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ). A IR do Kof tem 30 ops de alto nível; não há assembly inline nem acesso a hardware.

**Gap R6**: `putfield` de campos `Int` em compilador Kof causa `VerifyError` em runtime. Bloqueia implementação real de scheduler preemptivo. Documentado mas não resolvido ainda — workaround usando apenas variáveis locais em vez de atribuição de campo de classe.

**Modo autônomo**: Ativado via `scripts/auto-loop.sh start <sessionID> 30`; heartbeat re-dispatcha a cada 30 min.

---

Próximos passos: traduzir `lang/include/vibe_app.h` para o equivalente Kof usando `MemoryArenas` class e `kernel_memory_alloc`/`kernel_memory_free`.

Estratégia documentada conforme AGENTS.md — regras de modo autônomo, intenção não mecanismo, complexidade pertence à plataforma, represente o domínio, zero cerimônia, null alucinação evitada, multi-target honesto.
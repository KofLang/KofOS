# Guidelines — KofOS Port

Primary source: VibeOS guidelines (adapted for Kof 0.3.0-beta hosted)

## Core Principles (from AGENTS.md)

1. **Intenção, não mecanismo** — `spawn` (não `Thread`), `setOf().contains()` (não `||`), `==` (não `.equals()`).
2. **Complexidade pertence à plataforma** — JSON, DB, HTTP, cache, UI já existem na stdlib (`kof.*`). Reimplementar = anti-pattern.
3. **Represente o domínio, não a implementação acidental** — `List<T>`/`Map<K,V>`/`Set<T>`, não listas ligadas manuais.
4. **Zero cerimônia** — Sem getters/setters, sem builders, sem camadas Service/Repository/Controller.
5. **Nunca alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
6. **Multi-target honesto** — Código que só roda em um target precisa de diagnóstico claro (gap `HW001`), nunca fallback silencioso.

## Kof-Specific Conventions

### Função: `String nome(Int x) { }` (sem `fun`/`func`/`fn`).
Variável: `var x = 10` / `val y = 20` (só dentro de função).
Dados imutáveis: `record Point(Int x, Int y)`.
Classe mutável: `class X { campos; constructor(...) }`.
Strings: `a == b` (conteúdo), `a + "!"`.
Coleções: `listOf`/`mapOf`/`setOf` + `.map/.filter/.reduce`.
Erro: `throw "msg"` / `catch (String e)`.
Null: `String?` + `if (x != null)`.
Concorrência: `spawn` / `await` (sem `Thread`).
Loops: `for (var x in coll)` / if-expr / switch-expr.
Top-level: SÓ `class` e função (sem `val`/`var`/`let` top-level).

### Gap HW001 — Kernel bare-metal

O backend Native do Kof gera ELF x86-64 que depende de Linux + glibc (entry `_start` usa `SYS_gettid`/`exit_group`, aloca com `mmap`, usa `pthread_create`). Não configura GDT/IDT/paging/ring0 e não expõe primitivos de hardware (`in/out`, `cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ). A IR do Kof tem 30 ops de alto nível; não há assembly inline nem acesso a hardware.

**Decisão (design — não silencioso)**: o KofOS é portado como **kernel hosted** em Kof puro, preservando **toda a arquitetura e funcionalidade** do VibeOS (scheduler, processos, IPC, syscalls, serviços microkernel, VFS, AppFS, desktop, terminal, file manager, editor, task manager, jogos) e o **mesmo branding e fluxo de boot**. A camada de hardware (bootloader BIOS, GDT/IDT real, PIT, PIC, ports de I/O, ring0/ring3 real) é **abstraída** — ver `docs/kernel_init.md#gap-hw001-kernel-bare-metal-abstraido`.

Quando o compilador Kof ganhar modo freestanding + primitivos de hardware, o kernel pode ser retargetado a x86 real sem reescrever a lógica.

### Gap R6 — putfield de campos Int em compilador Kof

O compilador Kof (versão 0.3.0-beta) gera bytecode incorreto ao fazer atribuição de valor `Int` a campo de classe. Isso causa `VerifyError` em tempo de execução. O scheduler.kf foi simplificado removendo atribuições de campo `Int` para esse gap, usando apenas variáveis locais em vez de `this.campo = valor`. O scheduler usa fire-and-forget `kof.time.interval` para ticks de timer, o description do intervalo é String para evitar stored ID em campo Int.

Workaround: scheduler cooperativo via `kernel_scheduler_yield()` / `time.interval`. Scheduler preemptivo real requer corrigir o gap do compilador Kof (regra R6 — nunca silencioso).

### Modo autônomo

Ativado via `scripts/auto-loop.sh start <sessionID> 30`; heartbeat re-dispatcha a cada 30 min. O loop nunca para: 1. LER o estado (DOING.md, docs/status.md, git log, suíte) — nunca pergunte. 2. ESCOLHA a próxima tarefa: maior valor, sem dono `EM CURSO`, na sua lane. 3. REIVINDICAR no DOING.md (mesmo commit do primeiro passo). 4. A cada commit, atualize sua linha. 5. Ao concluir: `FEITO` com data + commit + prova (teste verde). 6. Itens `EM CURSO` por mais de uma sessão → abandone com nota.

### Translation Strategy from VibeOS C to Kof

- **Never alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
- **Funcionalidade já existe** — Muitas estruturas (scheduler, memória, IPC, VFS) já existem na estrutura Kof; reimplementar desnecessário a menos que corrigir gap ou adaptar design.
- **Inegualdade sintática** — O C usa `func nome(int x) { }`, o Kof usa `String nome(Int x) { }`. O C usa `malloc()`, o Kof usa `kernel_memory_alloc()`. O C usa `free()`, o Kof usa `kernel_memory_free()`.
- **I/O diferente** — O C usa `printf`, `scanf`, `fopen`, etc. O Kof usa `kof.print`, `kof.scan`, `kof.file` ( hosted) ou primitivas de hardware ( bare-metal).
- **Concorrência diferente** — O C usa `Thread`, `pthread_create`, etc. O Kof usa `spawn` / `await`.
- **Zero cerimônia** — O C usa structs com campos públicos, getters implícitos. O Kof usa `record` (dados imutáveis) ou `class` (mutável) com convenção de naming direta.

---

Fonte de verdade do comportamento: VibeOS em `/tmp/vibe-temp/` (repositório clonado) + documentação em `/tmp/vibe-temp/docs/`.

O que compila e roda em Kof hoje: é a prova real do comportamento (regra: compile e confirme).

**Gap HW001**: O backend Native do Kof gera ELF que depende de Linux + glibc. Não configura GDT/IDT/paging/ring0 e não expõe primitivos de hardware (`in/out`, `cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ). A IR do Kof tem 30 ops de alto nível; não há assembly inline nem acesso a hardware.

**Gap R6**: `putfield` de campos `Int` em compilador Kof causa `VerifyError` em runtime. Bloqueia implementação real de scheduler preemptivo. Documentado mas não resolvido ainda — workaround usando apenas variáveis locais em vez de atribuição de campo de classe.

**Modo autônomo**: Ativado via `scripts/auto-loop.sh start <sessionID> 30`; heartbeat re-dispatcha a cada 30 min.

---

Estratégia documentada conforme AGENTS.md — regras de modo autônomo, intenção não mecanismo, complexidade pertence à plataforma, represente o domínio, zero cerimônia, null alucinação evitada, multi-target honesto.
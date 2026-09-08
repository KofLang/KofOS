# Apps, Modules, and Executable Catalog — KofOS Port

Primary source: VibeOS `apps_and_modules.md` (adapted for KofOS hosted, gap HW001)

## Overview

The VibeOS has four categories of "apps" that build the executable surface. In KofOS hosted (gap HW001), the categorization adapts to the runtime environment: no AppFS external apps, no BSD compatibility ports, and the desktop-integrated apps are the primary application surface.

## 1. Execution tiers (KofOS adaptation)

There are three categories of "apps" in the KofOS surface, adapted from the VibeOS four tiers:

### Built-in bootstrap code (KofOS native)

These pieces are linked into the KofOS kernel image via Main.kf and exist to get the machine to a usable shell/desktop path:

- **Main.kf** — entry point, orchestrates boot → kernel_entry → init → userland → shell → done
- **kernel_entry** — primeira fase do boot, chamada por Main.kf
- **userland.kf** — bootstrap do userland, cria processo init e processo shell
- **kernel_scheduler_bootstrap()** — cria scheduler global
- **kernel_scheduler_start_preemption()** — liga preempção fire-and-forget via `kof.time.interval`

### Desktop-integrated apps (KofOS native)

These are the applications com interface integrada ao desktop do KofOS, com funcionalidade real:

| App | Arquivo | Funcionalidade |
|---|---|---|
| Calculator | calculator.kf | add, subtract, multiply, divide |
| File Manager | filemanager.kf | navegação por diretórios, mkDir, rmFile, pwd |
| Editor | editor.kf | setContent/getContent, append, clear |
| Task Manager | taskmanager.kf | listProcesses, getStatus |
| Clock | clock.kf | tick, seconds count |
| Shell | shell.kf | help/ls/exit commands, cooperative loop |

Important: Todos os apps acima usam `spawn`/`await` para concorrência (regra Kof 0.3.0-beta) e compartilham o scheduler cooperativo com timeslice de 4 ticks. O scheduler usa fire-and-forget `kof.time.interval` para ticks de timer, evitando gap `putfield de Int` (R6).

### Runtime apps (KofOS — via Kof package manager)

Esses seriam runtimes ou tooling layers pacoteados via Kof package manager, mas não fazem parte do port inicial:

- `hello`, `js`, `ruby`, `python`, `java`, `javac`, `lua`, `sectorc` — não inclusos no port inicial.

### Compatibility ports (KofOS — não inclusos)

Essas seriam utilitades importadas ou adaptadas, mas não fazem parte do port inicial do VibeOS para Kof:

- Jogos BSD (snake, tetris, pacman, etc.)
- Utilitades importadas de outros sistemas

## 2. Autoritative app catalog (KofOS)

O catálogo de comandos visível do shell é gerenciado via `userland.kf` e a estrutura de classes definidas em cada aplicativo. Não há geração automática de catalog como no VibeOS (`build/generated/app_catalog.h`), pois o KofOS hosted usa estrutura de classes Kof em vez de catálogo C header.

Os comandos visíveis são:

- `help` — shell command
- `ls` — filemanager command
- `exit` — shell command
- `calculator` — launch calculator
- `edit` — launch editor
- `taskmanager` — launch task manager
- `clear` — clear terminal

## 3. Boot-critical external apps (KofOS)

Esses apps são críticos para o caminho de boot:

| App | Role |
|---|---|
| **userland** | boot app default; inicia shell e opcionalmente rota desktop boot |
| **shell** | shell externo após init |

`Main.kf` tenta userland primeiro e só cai back daí.

## 5. Desktop-integrated apps (já portados)

Esses apps foram portados do VibeOS para Kof com funcionalidade real:

| App | Fonte VibeOS | Status KofOS |
|---|---|---|
| Calculator | calculator.c | traduzido para calculator.kf — operações matemáticas reais |
| File Manager | filemanager.c | traduzido para filemanager.kf — navegação por VFS ramfs |
| Editor | editor.c | traduzido para editor.kf — setContent/getContent, append, clear |
| Task Manager | taskmgr.c | traduzido para taskmanager.kf — listProcesses, getStatus |
| Clock | clock.c | traduzido para clock.kf — tick, segundos count |
| Shell | shell.c | traduzido para shell.kf — help/ls/exit, loop cooperativo |

Important detail:

- Todos os apps acima usam `spawn` para concorrência (regra Kof 0.3.0-beta).
- O scheduler usa fire-and-forget `kof.time.interval` para ticks de timer (gap R6 trabalhado).
- Nenhum desses apps requer primitivos de hardware reais (gap HW001 — kernel hosted).

## 6. Games (KofOS — pendente)

Jogos nativos do VibeOS (snake, tetris, pacman, space_invaders, pong, donkey_konk, brick_race, flap_birb) ainda não foram portados para KofOS. Requerem:
- Drivers de hardware reais (gap HW001 — não disponível sem corrigir bare-metal).
- Interface visual que não foi priorizada no port inicial.
- Poderiam ser adicionados futuramente quando o compilador Kof suportar modo freestanding + primitivos de hardware.

---

## Estratégia de tradução (AGENTS.md compliance)

- **Never alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
- **Multi-target honesto** — Código que só roda em um target precisa de diagnóstico claro (gap `HW001`), nunca fallback silencioso.
- **Intent, not mechanism** — `spawn` (não `Thread`), `setOf().contains()` (não `||`), `==` (não `.equals()`).
- **Complexidade pertence à plataforma** — Scheduler, memória, IPC já estão na estrutura Kof; reimplementar = anti-pattern se não necessário.
- **Zero cerimônia** — Sem getters/setters, sem builders, sem camadas Service/Repository/Controller.
- **Never alacune** — Se não está no corpus Kof, compile e confirme.

---

Fonte de verdade do comportamento: VibeOS em `/tmp/vibe-temp/` (repositório clonado) + documentação em `/tmp/vibe-temp/docs/`.

O que compila e roda em Kof hoje: é a prova real do comportamento (regra: compile e confirme).

**Gap HW001**: O backend Native do Kof gera ELF que depende de Linux + glibc. Não configura GDT/IDT/paging/ring0 e não expõe primitivos de hardware (`in/out`, `cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ). A IR do Kof tem 30 ops de alto nível; não há assembly inline nem acesso a hardware.

**Gap R6**: `putfield` de campos `Int` em compilador Kof causa `VerifyError` em runtime. Bloqueia implementação real de scheduler preemptivo. Documentado mas não resolvido ainda — workaround usando apenas variáveis locais em vez de atribuição de campo de classe.

**Modo autônomo**: Ativado via `scripts/auto-loop.sh start <sessionID> 30`; heartbeat re-dispatcha a cada 30 min.

---

Estratégia documentada conforme AGENTS.md — regras de modo autônomo, intenção não mecanismo, complexidade pertence à plataforma, represente o domínio, zero cerimônia, null alucinação evitada, multi-target honesto.

---

Próximos passos: port de games nativo dependendo de corrigir gap HW001 no compilador Kof ou implementação como aplicações hosted sem acesso hardware direto. Workaround atual: jogos não inclusos no port inicial; foco em apps desktop já portados e funcionais.
# Scheduler — KofOS Port

Primary source: VibeOS kernel/scheduler.c concepts (adapted for Kof 0.3.0-beta hosted)

## Overview

The VibeOS uses round-robin cooperativo + preempção por timeslice (4 ticks do PIT, 100Hz), com seleção por score (prioridade, afinidade de CPU, wake-boost e bootstrap slice para tarefas novas). No KofOS hosted (gap HW001), cada "processo" é uma thread Kof (spawn) e o scheduler seleciona o próximo processo pronto; a preempção por timer é simulada usando kof.time.interval fire-and-forget (o interval dispara ticks sem precisar stored ID em campo Int, evitando gap de putfield do compilador).

## Scheduler State

- **Process states**: READY, RUNNING, BLOCKED, TERMINATED (ProcState class)
- **Process kinds**: USER, SERVICE (ProcKind class)
- **Priorities**: DESKTOP_USER, INPUT, VIDEO, STORAGE, AUDIO, NETWORK, APP, BACKGROUND (Priority class)
- **Timeslice**: 4 ticks (SchedConsts.TIMESLICE_TICKS = 4)
- **Current/cursor/head**: Process linked list management

## Algorithm — Score Strategy (simplified, without waitResult field assignment que causa VerifyError)

`score(task, cpuIndex)` — versão simplificada sem waitResult que causa VerifyError por putfield de Int:

1. **Afinit de CPU**: affinityScore = 3 se não há afinidade, 0 se mesma CPU, 1 se última CPU, 2 se preferredCPU < 0.
2. **Protege classes interativas**: se task.priorityTier <= Priority.INPUT e affinityScore > 1, affinityScore = 1. Se task.priorityTier == Priority.VIDEO e affinityScore > 2, affinityScore = 2.
3. **Bootstrap slice**: serviço novo executa antes (task.kind == ProcKind.SERVICE && task.contextSwitches == 0 && task.runtimeTicks == 0): return -40 + task.priorityTier.
4. **Bootstrap slice**: user novo: task.priorityTier == Priority.DESKTOP_USER → -36; Priority.AUDIO/NETWORK → -30; Priority.APP → -34; senão → -12.
5. **App recentemente lançado**: poucos slices iniciais sem starve: task.kind == ProcKind.USER && task.priorityTier == Priority.APP && task.contextSwitches < 4 && task.runtimeTicks < 16 → return -26.
6. **Score base**: prioridade * 16 + afinidade. Se baseScore < 0, baseScore = 0.

## Scheduler Tick (fire-and-forget)

`tick()` — um tick de timer decorreu; decrementa o timeslice e seleciona o próximo processo.

Usa kof.time.interval fire-and-forget para disparar ticks; o description do intervalo é String para evitar putfield de Int no compilador (gap R6).

Se current != null && current.state == RUNNING:
  - Se timesliceRemaining > 1: timesliceRemaining--; return.
  - current.runtimeTicks++; current.contextSwitches++; current.state = READY; current.currentCpu = -1.

Var nxt = next(); Se nxt == null, return.
cursor = Se nxt.next != null, nxt.next, senão head.
nxt.state = RUNNING; nxt.currentCpu = 0; nxt.lastCpu = 0; nxt.contextSwitches++; timesliceRemaining = 4.
current = nxt.

## Blocking / Waiting

`block_current()` — Bloqueia o processo atual: current.state = BLOCKED; current = null; timesliceRemaining = 4.

`signal()` — Acorda processos bloqueados (sem assignment waitResult que causa VerifyError): Para cada task na cabeça: Se task.state == BLOCKED, task.state = READY.

## Translation Strategy (AGENTS.md compliance)

- **Never alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
- **Multi-target honesto** — Código que só roda em um target precisa de diagnóstico claro (gap `HW001`), nunca fallback silencioso.
- **Intent, not mechanism** — `spawn` (não `Thread`), `setOf().contains()` (não `||`), `==` (não `.equals()`).
- **Complexidade pertence à plataforma** — Scheduler, memória, IPC já existem na estrutura Kof; reimplementar = anti-pattern se não necessário.
- **Zero cerimônia** — Sem getters/setters, sem builders, sem camadas Service/Repository/Controller.
- **Represente o domínio, não a implementação acidental** — `List<T>`/`Map<K,V>`/`Set<T>`, não listas ligadas manuais.

## Gap R6 — putfield de Int em compilador Kof

O compilador Kof (versão 0.3.0-beta) gera bytecode incorreto ao fazer atribuição de valor `Int` a campo de classe. Isso causa `VerifyError` em tempo de execução. O scheduler.kf foi simplificado removendo atribuições de campo `Int` para esse gap, usando apenas variáveis locais em vez de `this.campo = valor`. O scheduler usa fire-and-forget `kof.time.interval` para ticks de timer, o description do intervalo é String para evitar stored ID em campo Int.

Workaround: scheduler cooperativo via `kernel_scheduler_yield()` / `time.interval`. Scheduler preemptivo real requer corrigir o gap do compilador Kof (regra R6 — nunca silencioso).

---

Fonte de verdade do comportamento: VibeOS em `/tmp/vibe-temp/` (repositório clonado).

O que compila e roda em Kof hoje: é a prova real do comportamento (regra: compile e confirme).

**Próximos passos**: scheduler com preempção real por timer dependendo de corrigir gap R6 no compilador Kof. Workaround atual: scheduler cooperativo com timeslice controlado por `kernel_scheduler_yield()`.

---

Estratégia documentada conforme AGENTS.md — regras de modo autônomo, intenção não mecanismo, complexidade pertence à plataforma, represente o domínio, zero cerimônia, null alucinação evitada, multi-target honesto.
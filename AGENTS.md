# AGENTS.md — KofOS (guia obrigatório para agentes de IA)

Este é o guia **obrigatório** para qualquer agente de IA (ou humano) que escreva
código Kof neste repositório. É o port completo do **VibeOS**
(https://github.com/melmonfre/vibe-os) para a linguagem **Kof**. Leia antes de
gerar qualquer `.kf`.

**Versão:** 0.1.0 · Última atualização: 07/09/2026 (port iniciado)

---

## Missão

> Portar **todas as funcionalidades** do VibeOS para Kof, mantendo a mesma
> arquitetura, o mesmo branding e o mesmo fluxo de boot
> (`BIOS -> MBR -> stage1 -> stage2 -> KERNEL -> kernel -> init -> AppFS ->
> userland -> shell -> desktop`).

O VibeOS é um sistema operacional x86 (32-bit) BIOS com bootloader próprio,
kernel (scheduler, memória paginada, ELF loader, VFS, IPC, serviços bootstrap),
userland (shell, desktop, terminal, file manager, editor, task manager, jogos)
e apps modulares carregados do AppFS em ring3.

**Fonte de verdade do comportamento:** o código do VibeOS em
`/tmp/vibe-temp/` (repositório clonado) + sua documentação em `/tmp/vibe-temp/docs/`.
**O que compila e roda em Kof hoje:** é a prova real do comportamento
(regra: nunca a memória — compile e confirme).

---

## Modo autônomo (definição — o padrão de operação desta sessão)

> **Entrar em modo autônomo = trabalhar sem interromper o humano, por dias a
> fio, até a próxima interferência humana.** O humano não está disponível para
> perguntas; o repo é a única fonte de verdade. Tudo de que você precisa já
> está nos documentos e no código do VibeOS — se não está, é porque precisa
> ser escrito (e você escreve).

**O loop (nunca pare no meio):**

```
1. LEIA o estado (DOING.md, docs/status.md, git log, suíte) — nunca pergunte.
2. ESCOLHA a próxima tarefa: maior valor, sem dono `EM CURSO`, na sua lane.
3. REIVINDIQUE no DOING.md (mesmo commit do primeiro passo).
4. QUEBRE em escopos realizáveis numa sessão (ver "partes pequenas" abaixo).
5. EXECUTE um escopo → teste → commit → atualize DOING.md + todowrite.
6. VOLTE ao passo 1. Não anuncie "fim"; só pare por condição de parada.
```

**O loop dentro de UM turno:** encadeie tool calls até (a) condição de parada,
(b) contexto quase esgotar, ou (c) turno sem trabalho novo. **Proibido terminar
o turno com resumo/status** ("pushed", "resumo da sessão"). Se o turno vai
acabar, a ÚLTIMA coisa escrita é: commit final + `DOING.md` com a linha
`PRÓXIMO PASSO: <tarefa exata + arquivo + prova esperada>` + `todowrite`
espelhando isso.

**Heartbeat de cron (obrigatório ao entrar no modo autônomo):**

```bash
scripts/auto-loop.sh start <sessionID> 30     # re-disparo a cada 30 min
scripts/auto-loop.sh status                   # confirmar que está ativo
```

**Sinal para o humano:** `todowrite` é a janela desta conversa (atualize a
cada etapa); `DOING.md` é a memória entre sessões.

---

## Regras de ferro (aplicam-se ao KofOS também — herdadas do Kof4j)

1. **Intenção, não mecanismo.** `spawn` (não `Thread`), `setOf().contains()`
   (não `||`), `==` (não `.equals()`), `json.encode` (não parser manual).
2. **Complexidade pertence à plataforma.** JSON, DB, HTTP, cache, UI já
   existem na stdlib (`kof.*`). Reimplementar = anti-pattern.
3. **Represente o domínio, não a implementação acidental.** `List<T>`/`Map<K,V>`/
   `Set<T>`, não listas ligadas manuais.
4. **Zero cerimônia.** Sem getters/setters, sem builders, sem utility classes,
   sem camadas Service/Repository/Controller.
5. **Nunca alucine sintaxe.** Se não está no corpus Kof
   (`/home/mel/Kof4j/training/`), **compile e confirme** antes de usar.
6. **Multi-target honesto.** Código que só roda em um target precisa de
   diagnóstico claro (gap `HW001`), nunca fallback silencioso.

### Regras de qualidade (vindas do Kof4j AGENTS.md, aplicam-se aqui)

- **≤500 linhas por arquivo `.kf`.** Se passar, divida. (O port do VibeOS tem
  arquivos gigantes em C — ao portar, divida em colaboradores coesos.)
- **Compile antes de commit.** `kof check <file>` ou o teste E2E — nunca
  entregue código Kof que não compila.
- **Sintaxe Kof real** (verificado no compilador 0.3.0-beta):
  - Função: `String nome(Int x) { }` (sem `fun`/`func`/`fn`).
  - Variável: `var x = 10` / `val y = 20` (só dentro de função).
  - Dados imutáveis: `record Point(Int x, Int y)`.
  - Classe mutável: `class X { campos; constructor(...) }`.
  - Strings: `a == b` (conteúdo), `a + "!"`.
  - Coleções: `listOf`/`mapOf`/`setOf` + `.map/.filter/.reduce`.
  - Erro: `throw "msg"` / `catch (String e)`.
  - Null: `String?` + `if (x != null)`.
  - Concorrência: `spawn` / `await` (sem `Thread`).
  - Loops: `for (var x in coll)` / if-expr / switch-expr.
  - Top-level: SÓ `class` e função (sem `val`/`var`/`let` top-level).
- **Testes:** cada unidade de kernel/userland tem um teste de comportamento
  (`test "nome" { }` + `assert(cond, "msg")`). A suíte é gate de merge.

---

## Gap de plataforma honesto — HW001 (kernel bare-metal)

**Contexto (verificado 07/09/2026):** o backend Native do Kof (`NativeBackend`)
gera ELF x86-64 que **depende do Linux + glibc** (entry `_start` usa
`SYS_gettid`/`exit_group`, aloca com `mmap`, usa `pthread_create`). Não
configura GDT/IDT/paging/ring0 e **não expõe primitivos de hardware** (`in/out`,
`cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ). A IR do Kof tem 30 ops de alto nível;
não há assembly inline nem acesso a hardware.

**Decisão (design — não silenciosa):** o KofOS é portado como **kernel hosted**
em Kof puro, preservando **toda a arquitetura e funcionalidade** do VibeOS
(scheduler, processos, IPC, syscalls, serviços microkernel, VFS, AppFS, desktop,
terminal, file manager, editor, task manager, jogos) e o **mesmo branding e
fluxo de boot**. A camada de hardware (bootloader BIOS, GDT/IDT real, PIT, PIC,
I/O ports, ring0/ring3 real) é **abstraída** — ver `docs/HW001-hal.md`. Quando o
compilador Kof ganhar modo freestanding + primitivos de hardware, o kernel pode
ser retargetado a x86 real sem reescrever a lógica.

- **HAL (abstração de hardware):** interface em `kernel/hal/` que mapeia
  hardware do VibeOS (timer, input, vídeo, storage) para a plataforma Kof
  (`kof.time`, `kof.io`, `kof.ui`), para que a lógica do kernel não mude.
- **Fluxo de boot preservado:** `boot` → `kernel_entry` → `init` → `AppFS` →
  `userland` → `shell` → `desktop`, como no VibeOS, mas como fases de uma
  aplicação Kof.

---

## Corpus de referência

| Arquivo | Conteúdo |
|---|---|
| `/tmp/vibe-temp/README.md` | Visão geral do VibeOS (estado, arquitetura, build, validação) |
| `/tmp/vibe-temp/docs/README.md` | Índice da documentação técnica |
| `/tmp/vibe-temp/docs/overview.md` | Visão arquitetural |
| `/tmp/vibe-temp/docs/workflow.md` | Fluxo de trabalho/desenvolvimento |
| `/tmp/vibe-temp/docs/kernel_init.md` | Inicialização do kernel |
| `/tmp/vibe-temp/docs/scheduler.md` | Scheduler |
| `/tmp/vibe-temp/docs/drivers.md` | Drivers |
| `/tmp/vibe-temp/docs/apps_and_modules.md` | Apps modulares + AppFS |
| `/tmp/vibe-temp/kernel/` | Kernel C (source da verdade do comportamento) |
| `/tmp/vibe-temp/userland/` | Userland (shell, desktop, apps) |
| `/tmp/vibe-temp/headers/` | Headers de interface |
| `/home/mel/Kof4j/training/` | Corpus de idiomática Kof |
| `/home/mel/Kof4j/docs/` | Arquitetura da linguagem/compilador |

---

## Estrutura do KofOS (port)

```
KofOS/
├── AGENTS.md            ← este arquivo
├── DOING.md             ← coordenação multi-agente (quem faz o quê)
├── README.md            ← branding + visão
├── docs/                ← documentação do port
│   └── HW001-hal.md     ← gap de plataforma (kernel bare-metal)
│   └── architecture.md  ← arquitetura do KofOS (port)
├── kernel/              ← kernel hosted em Kof
│   ├── boot.kf          ← kernel_entry / fase de boot
│   ├── scheduler.kf     ← scheduler + processos + preempção
│   ├── memory.kf        ← heap, páginas, arenas
│   ├── syscall.kf       ← syscalls (int 0x80 simulado)
│   ├── ipc.kf           ← IPC (mailboxes, waitables)
│   ├── services/        ← serviços microkernel (init, video, input, fs, ...)
│   ├── vfs.kf           ← VFS + ramfs + AppFS
│   └── hal/             ← abstração de hardware (timer, input, vídeo, storage)
├── userland/            ← userland (shell, desktop, terminal, apps)
│   ├── shell.kf
│   ├── desktop.kf
│   ├── terminal.kf
│   ├── filemanager.kf
│   ├── editor.kf
│   ├── taskmgr.kf
│   └── apps/            ← jogos, calculator, clock, imageviewer
└── scripts/             ← build, auto-loop
```

---

## Loop de verificação (obrigatório)

```bash
export PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH
export KOF=/home/mel/Kof4j/bin/kof

# 1. Compilar/checar um arquivo
$KOF check kernel/boot.kf

# 2. Rodar o programa principal (boot → shell → desktop)
$KOF run Main.kf

# 3. Rodar a suíte de testes
$KOF test <dir>
```

**Nunca** entregue código Kof que você não compilou.
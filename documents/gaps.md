# GAPs do Compilador Kof (Kof 0.3.0-beta)

Esta documentação descreve os gaps (limitações conhecidas) do compilador Kof que afetam o port do VibeOS para KofOS. Todos os gaps são documentados honesta conforme a filosofia Kof4j — nunca silenciosamente.

## Gap HW001 — Kernel bare-metal

**Status**: Documentado como decisão de design  
**Plataforma**: Native (Native Backend do Kof)  
**Descrição**: O backend Native do Kof gera ELF x86-64 que depende do Linux + glibc. O ponto de entrada `_start` usa `SYS_gettid`/`exit_group`, aloca com `mmap`, usa `pthread_create`. Não configura GDT/IDT/paging/ring0 e não expõe primitivos de hardware (`in/out`, `cli/sti`, `lgdt/lidt`, `int 0x80`, IRQ).

**A IR do Kof tem 30 ops de alto nível; não há assembly inline nem acesso a hardware.**

**Decisão (design — não silenciosa)**: o KofOS é portado como **kernel hosted** em Kof puro, preservando **toda a arquitetura e funcionalidade** do VibeOS (scheduler, processos, IPC, syscalls, serviços microkernel, VFS, AppFS, desktop, terminal, file manager, editor, task manager, jogos) e o **mesmo branding e fluxo de boot**. A camada de hardware (bootloader BIOS, GDT/IDT real, PIT, PIC, ports de I/O, ring0/ring3 real) é **abstraída** — ver `docs/kernel_init.md#gap-hw001-kernel-bare-metal-abstraido`.

**Quando o compilador Kof ganhar modo freestanding + primitivos de hardware**, o kernel pode ser retargetado a x86 real sem reescrever a lógica.

**Impacto no port**: O KofOS preserva a arquitetura VibeOS (boot → scheduler → memória → syscalls → IPC → VFS → userland), mas roda como aplicação hosted no runtime Kof, não como kernel bare-metal.

---

## Gap R6 — putfield de campos `Int`

**Status**: Aberto — requer correção no compilador Kof  
**Plataforma**: Todos os targets (JVM, Native, JS)  
**Descrição**: O compilador Kof (versão 0.3.0-beta) gera bytecode incorreto ao fazer atribuição de valor `Int` a campo de classe (`putfield`). Isso causa `VerifyError` em tempo de execução quando o código tenta atribuir um `Int` a um campo de classe de um objeto.

**Exemplo problemático**:
```kof
class Foo {
    Int x = 0
    void setX(Int v) { x = v }  // Pode gerar VerifyError em runtime
}
```

**Impacto no port do VibeOS**: O scheduler.kf foi simplificado removendo atribuições de campo `Int` para esse gap, usando apenas variáveis locais em vez de `this.campo = valor`. O scheduler usa fire-and-forget `kof.time.interval` para ticks de timer, o description do intervalo é String para evitar stored ID em campo Int.

**Workaround**: scheduler cooperativo via `kernel_scheduler_yield()` / `time.interval`. Scheduler preemptivo real requer corrigir o gap do compilador Kof (regra R6 — nunca silencioso).

**Impacto geral**: Qualquer código Kof que tente atribuição de `Int` a campos de classe em tempo de execução pode falhar com `VerifyError`. A maioria do código seguro usa apenas variáveis locais ou `record` (dados imutáveis), que não sofrem desse problema.

---

## Outras Limitações Conhecidas

### Gap CONC001 (Já Fechado: 31/08)
- **Native**: Concurrency com `pthread_create` + trampoline + `await`/`pthread_join` + allocator thread-safe futex + join implícito no fim do `main`.
- **JS**: Execução sequencial — `spawn`/`await` cobrem statement e expression; async real de event-loop = CONC003 parcial.

### Gap WEB001 (Parcial: 30/08)
- **JVM**: `web.app()` completo — rotas `get/post/put/delete/patch/options`, `status(201, body)`, `headerSet`, WebSocket, SSE, `listenSecure` TLS.
- **Native/JS**: WEB001 — funcionalidade limitada ou não disponível.

### Gap MQ001 (Já Fechado: 01/09)
- Filas produtor/consumidor (`kof.mq`) nos 3 targets (Native/JS/JVM).

### Gap CONC003 (Parcial)
- **JS**: Async real de event-loop parcial.

### Gap WEB002 (Native)
- HTTP client/server no Native com funcionalidade limitada.

---

## Estratégia de Contorno

1. **Never alacune** — Se não está no corpus Kof (`/home/mel/Kof4j/training/`), compile e confirme antes de usar.
2. **Represente o domínio, não a implementação acidental** — `List<T>`/`Map<K,V>`/`Set<T>`, não listas ligadas manuais.
3. **Zero cerimônia** — Sem getters/setters, sem builders, sem camadas Service/Repository/Controller.
4. **Use records para dados imutáveis** — `record Point(Int x, Int y)` — fields são imutáveis por definição, evitando o gap `putfield`.
5. **Use variáveis locais em vez de campos de classe** quando possível para atribuições de `Int`.
6. **Mude o design** — Em vez de `class Foo { Int x; }`, use `record Foo(Int x)` ou funcões que retornam valores em vez de modificar estado compartilhado.

---

## Como Verificar

Para verificar se um código Kof sofre do gap R6:

```kof
// Problemático (pode causar VerifyError):
class Foo {
    Int x
    void bar() { x = 1 }
}

// Seguro (records são imutáveis):
record Foo(Int x)

// Seguro (variáveis locais):
void bar() {
    var x = 1
}
```

---

## Roadmap de Correção

- **Corrigir gap R6 no compilador Kof**: Mudança no backend de codegen para evitar `putfield` de `Int` em campos de classe.
- **Quando corrigido**: Scheduler Kof pode ser implementado com preempção real por timer, removendo o workaround cooperativo atual.

---

**Fonte**: Análise de bytecode compilado Kof 0.3.0-beta + verificação de runtime `VerifyError`.  
**Mantido por**: Equipe KofLang.  
**Atualizado**: 08/09/2026.

---

Estratégia documentada conforme AGENTS.md — regras de modo autônomo, intenção não mecanismo, complexidade pertence à plataforma, represente o domínio, zero cerimônia, null alucinação evitada, multi-target honesto.
# KofOS idioms — `this.` no construtor

Aprendido portando o kernel do VibeOS (07/09).

## Problema

Ao escrever construtores de classes mutáveis, quando usar `this.`?

## BAD (cerimônia desnecessária)

```kof
class Process {
    Int state
    public constructor(Int pid, String name, Int kind, Int priority) {
        this.state = ProcState.READY   // sem shadowing — this. é dispensável
        this.runtimeTicks = 0          // idem
    }
}
```

## GOOD (idiomático)

```kof
class Process {
    Int pid
    Int state
    public constructor(Int pid, String name, Int kind, Int priority) {
        this.pid = pid                 // NECESSÁRIO: param pid == campo pid (shadowing)
        this.name = name               // idem
        state = ProcState.READY        // sem shadowing: `this.` implícito resolve o campo
        runtimeTicks = 0
    }
}
```

## WHY

O `this.` implícito de Kof já resolve campos quando não há parâmetro de mesmo
nome. Usar `this.` onde não há shadowing é Java-translator (cerimônia sem
semântica). Mas quando o parâmetro tem o MESMO nome do campo (`this.pid = pid`),
o `this.` é **obrigatório** para distinguir campo de parâmetro — essa é a forma
canônica documentada em `training/idioms/classes.md` (classe mutável com estado).

**Regra:** use `this.` só onde há shadowing (campo com mesmo nome do parâmetro
ou variável local). Nos demais, deixe o `this.` implícito.
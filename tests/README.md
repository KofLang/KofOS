# Testes do KofOS

O `kof test <arquivo>` usa o diretório do arquivo como raiz do módulo. Como o
KofOS é um único módulo com `Main.kf` na raiz, os testes E2E que importam o
kernel (`import kernel.boot`) precisam estar na **raiz do repositório**.

## Convenção

- Testes unitários que não importam o kernel: podem ficar em `tests/`.
- Testes E2E do boot (importam `kernel.*`): devem ficar na **raiz** (ex.
  `boot_test.kf`), porque a raiz do módulo Kof é onde está o `Main.kf`.

## Rodar a suíte E2E do boot

```bash
export PATH=/usr/lib/jvm/java-21-openjdk-amd64/bin:$PATH
/home/mel/Kof4j/bin/kof test boot_test.kf
```

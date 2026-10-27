# Análise inicial — TinyGo bare-metal no Raspberry Pi 3B

## Resultado executivo

É tecnicamente viável executar um programa TinyGo sem Linux no Raspberry Pi 3B (BCM2837, quatro Cortex-A53), mas **o TinyGo não fornece hoje um target pronto para essa placa**. A instalação verificada nesta máquina é `tinygo 0.41.1 windows/amd64`, com Go `1.26.5` e LLVM `20.1.1`; a enumeração de targets e o diretório `targets/` não contêm um alvo `raspberry`, `rpi` ou `pi3`.

A primeira implementação deve criar um target AArch64 específico, um linker script, código de entrada/vector de exceção e uma pequena extensão do runtime do TinyGo. Um simples JSON de target não basta: o runtime bare-metal genérico requer símbolos do linker e implementações dependentes da plataforma para tempo, saída, parada e inicialização.

A escolha recomendada é **AArch64**, com uma imagem plana `kernel8.img` carregada em `0x80000`, execução inicial em um único núcleo e sem MMU, interrupções ou drivers além de uma futura UART de polling. AArch32 permanece uma alternativa de contingência, não o ponto de partida.

Nada do Circle foi modificado, nenhum driver foi convertido e esta etapa não contém implementação.

## Escopo, evidências e limites

O Circle foi apenas lido em `D:\Gustavo\Projetos\.Circle_Master\circle-master`. A cópia local é um arquivo ZIP extraído, sem diretório `.git`; a constante `CIRCLEVER` em `Rules.mk` é `510101`. As conclusões vêm do código e da documentação presentes nessa cópia e das fontes oficiais vinculadas em [Referências](#referências).

Também foram verificados diretamente:

- `tinygo version`: `0.41.1 windows/amd64`, Go `1.26.5`, LLVM `20.1.1`;
- `tinygo targets`: nenhum target Raspberry Pi 3/AArch64 bare-metal;
- `tinygo build -h`: `-target` aceita o nome de um alvo **ou o caminho de um arquivo JSON de target**;
- a distribuição instalada contém `targets/nintendoswitch.json`, que é ARM64, porém não é Raspberry Pi nem bare-metal genérico.

Não se deve deduzir compatibilidade de placa a partir de o LLVM aceitar `aarch64`: boot, layout de memória, exceções, temporizador e MMIO continuam sendo responsabilidade do target.

## 1. Inicialização do Raspberry Pi 3B no Circle

### Caminho AArch64 usado como referência

| Arquivo do Circle | Responsabilidade observada | Relevância para o port |
| --- | --- | --- |
| `boot/config64.txt` | Seleciona `arm_64bit=1`, `kernel_address=0x80000` e, em `[pi3]`, `kernel=kernel8.img`. | Define o contrato de boot a respeitar. |
| `Rules.mk` | Para `AARCH=64` e `RASPPI=3`, seleciona `-mcpu=cortex-a53`, `TARGET=kernel8` e `LOADADDR=0x80000`. | Confirma CPU, nome e endereço da imagem. |
| `circle.ld` | Declara `ENTRY(_start)`, ordena as seções e exporta limites de BSS. | Modelo do linker; o script TinyGo terá símbolos próprios. |
| `lib/startup64.S` | Implementa `_start`; trata EL, prepara `sp_el1`, `VBAR_EL2`, baixa de EL2 para EL1, configura `sp_el0`, `VBAR_EL1` e salta para `sysinit`. | Principal referência de transição de nível de exceção e stacks. |
| `lib/exceptionstub64.S` | Cria vector table alinhada a 2 KiB, com slots de exceção e salvamento de contexto para IRQ/FIQ. | Referência para `VBAR_EL1`; não é código a copiar. |
| `lib/sysinit.cpp` | Limpa `.bss` e `.tbss`, confere o limite da imagem, inicializa memória/interrupções, chama construtores e a aplicação. | Define a ordem: memória estática válida antes do runtime/aplicação. |
| `include/circle/memorymap64.h` e `doc/memorymap.txt` | Para Pi 3 AArch64: imagem em `0x80000`, stacks após ela, região coerente e heap posterior; quatro núcleos. | Base de análise do mapa de memória, não um mapa a copiar literalmente. |
| `lib/multicore.cpp` e `include/circle/bcm2836.h` | Inicializam secundários por spin table local, usam `SEV` e cache/barreiras; periféricos locais em `0x40000000`. | Justifica estacionar os núcleos 1–3 e adiar multicore. |
| `include/circle/bcm2835.h` | Para `RASPPI <= 3`, fixa MMIO principal em `0x3f000000`; define GPIO, PL011, mailbox e controlador de interrupção. | Referência para endereços BCM2837 a validar contra documentação do hardware. |

O comentário de `lib/startup64.S` registra a entrada usual via firmware/ARM stub em EL2. Ele também trata a entrada já em EL1. A nossa entrada deve ter a mesma postura defensiva: ler `CurrentEL`, mascarar exceções, estabelecer stacks alinhadas e só então transferir o controle ao runtime.

No Circle, `sysinit` limpa BSS antes de construir objetos ou chamar a aplicação. Para a nossa imagem plana, os bytes inicializados de `.data` já estarão na imagem carregada pelo firmware; `.bss`/NOLOAD devem ser limpos explicitamente. Não haverá loader de ELF no Raspberry Pi: `kernel8.img` precisa ser binário cru.

### Equivalente AArch32, analisado para comparação

| Arquivo | Diferença relevante |
| --- | --- |
| `boot/config32.txt` | Seleciona `arm_64bit=0`; para Pi 3 o firmware pode carregar `kernel7.img` ou uma imagem explicitamente escolhida. |
| `Rules.mk` | Com `AARCH=32`, `RASPPI=3` usa `-mcpu=cortex-a53 -marm -mfpu=neon-fp-armv8`, `TARGET=kernel8-32` e `LOADADDR=0x8000`. |
| `lib/startup.S` | Alterna modos ARM (FIQ, IRQ, abort, system), cria stack por modo e chama `sysinit`; não há transição EL2 → EL1. |
| `lib/exceptionstub.S` e `lib/cache-v7.S` | Implementam vetores/CP15 e cache de 32 bits. |
| `include/circle/memorymap.h` | Usa páginas de 4 KiB, tabelas 32-bit/LPAE e `MEM_KERNEL_START=0x8000`. |

## 2. Requisitos de CPU, memória, stack e linker

### Contrato de boot AArch64

O firmware do Raspberry Pi continua necessário — “bare-metal” significa sem Linux, não sem firmware da GPU/boot ROM. A FAT de boot deverá conter firmware oficial compatível, `config.txt` e `kernel8.img`; não haverá partição raiz nem kernel Linux. A documentação oficial confirma `kernel8.img` como imagem 64-bit da Pi 3 e `arm_64bit=1` como seletor de modo.

O bootstrap AArch64 mínimo terá de:

1. Entrar por `_start` em `0x80000`, ler `CurrentEL` e mascarar `DAIF`.
2. Deixar cores 1–3 em `WFE`/loop seguro. Não usar spin table, mailbox local ou scheduler multicore na primeira prova de vida.
3. Se entrar em EL2, liberar contador genérico a EL1, configurar `HCR_EL2` para EL1 AArch64 e retornar com `ERET`; se já estiver em EL1, não repetir a transição.
4. Criar stack de bootstrap com alinhamento de 16 bytes; separar depois stack de exceção (EL1h) da stack da aplicação (EL1t/SP_EL0), ou manter explicitamente EL1h no primeiro marco.
5. Instalar vector table AArch64 de 2 KiB alinhada em `VBAR_EL1`. Antes de IRQ, os 16 slots devem ir a handler seguro que pare e, quando a UART existir, reporte ESR/ELR/FAR.
6. Limpar `.bss`, preservar `.data`, configurar limites de heap e chamar o ponto de entrada do runtime TinyGo, que executa inicializadores Go antes de `main.main`.

No primeiro marco, manter MMU, I-cache, D-cache e IRQ/FIQ desabilitados após inicialização arquitetural explícita e barreiras adequadas. Assim, MMIO usa endereços físicos/identidade e há menos variáveis de falha. Para ativar MMU depois serão necessários `MAIR_EL1`, `TCR_EL1`, `TTBR0_EL1`, tabelas, atributos distintos para RAM normal e Device-nGnRE, invalidação de TLB/cache e `DSB`/ `ISB`.

### Memória e símbolos obrigatórios

O linker AArch64 próprio deve materializar pelo menos:

```text
0x00080000  .text.boot / _start e vector table
            .text, .rodata, .data
            .bss (NOLOAD; zerada pelo bootstrap)
            stacks reservadas (NOLOAD, alinhadas)
            heap estático [ _heap_start, _heap_end )
```

Além dos limites de BSS, deverá exportar ao menos `_heap_start`, `_heap_end`, `_globals_start`, `_globals_end` e `_stack_top`. O `src/runtime/baremetal.go` do TinyGo instalado declara esses símbolos externamente: eles determinam heap fixo e globais examinados pelo runtime. Os nomes e a semântica deverão ser verificados contra o commit fixado do fork, não inventados.

`_heap_end` não pode ser fixado em 1 GiB. A Pi 3B compartilha RAM com GPU e a parcela ARM varia com `gpu_mem`/firmware. A primeira implementação deve obter a região ARM via property mailbox (`GET_ARM_MEMORY`, exposto no Circle como `PROPTAG_GET_ARM_MEMORY`) antes de liberar heap, ou usar limite conservador temporário e documentado. Imagem, stacks, buffer de mailbox alinhado e futura DMA devem ficar fora do heap.

## 3. TinyGo e necessidade do target personalizado

### O que existe de fato

O TinyGo instalado tem backend ARM64: há `src/runtime/arch_arm64.go`, `src/runtime/asm_arm64.S` e `targets/nintendoswitch.json`. Este último, porém, define `cpu: cortex-a57`, modelo PIC e chamadas Horizon do Nintendo Switch. Não contém boot de Pi, MMIO BCM2837, UART, vetor de exceção ou contrato de memória do firmware Raspberry Pi; reutilizá-lo seria incorreto.

Os targets bare-metal Cortex-M servem apenas de padrão estrutural. Seu JSON declara build tags, linker, `compiler-rt`, `picolibc`, script e assembly adicional; depois, um runtime específico — por exemplo `src/runtime/runtime_atsamd51.go` — exporta `Reset_Handler`, chama `preinit()`/ `run()` e fornece relógio, serial e parada. O vetor de reset Cortex-M não serve para Cortex-A53.

### Decisão

**Precisamos de target personalizado.** Ele terá duas partes, pinadas à mesma revisão do TinyGo:

1. JSON `rpi3b-aarch64`, linker script e assembly de boot/vetores;
2. extensão do pacote interno `runtime` do TinyGo, selecionada pela tag `rpi3baremetal`, para entrada, heap, tempo, `putchar`, espera e operações críticas/atômicas em AArch64.

Isso requer **fork do TinyGo** ou contribuição upstream. Código Go do projeto da aplicação não consegue acrescentar arquivos ao pacote padrão `runtime` que TinyGo compila de `TINYGOROOT`. Um JSON no projeto pode apontar artefatos de build, mas não substitui runtime Go interno.

O JSON seguirá o padrão bare-metal existente, com valores validados na implementação: LLVM target AArch64, CPU `cortex-a53`, `goarch: arm64`, `ld.lld`, `compiler-rt`, `picolibc`, tags `baremetal`/ `arm64`/ `rpi3baremetal`, script e assembly próprios. `goos: linux` pode servir apenas para compatibilidade de seleção da stdlib, como nos targets Cortex-M; a tag `baremetal` exclui caminhos que exigem Linux. Isso não inclui Linux na imagem.

O perfil inicial proposto é `-scheduler=none -gc=leaking`:

- `scheduler=none` remove goroutines/channels e reduz variáveis antes de timer/IRQ;
- `gc=leaking` mantém alocação simples, sem liberação; adequado só a testes curtos com orçamento de heap;
- `gc=none` é útil para localizar alocações, mas falha no link onde houver alocação;
- GC conservador fica para depois de validar stack, exceções e símbolos de globais em AArch64.

## 4. AArch64 comparado com AArch32

| Critério | AArch64 (recomendado) | AArch32 (contingência) |
| --- | --- | --- |
| Imagem/entrada Circle | `kernel8.img` em `0x80000`; entrada usual em EL2 e transição para EL1. | `kernel7.img` normalmente, ou `kernel8-32.img`; entrada em `0x8000`, modos ARM clássicos. |
| CPU Pi 3B | Uso nativo do Cortex-A53 de 64 bits, registradores `x0`–`x30`. | Mesmo chip em 32 bits, registradores de 32 bits, VFP/NEON e CPSR. |
| Referência Circle | `startup64.S`, `exceptionstub64.S`, `memorymap64.h`. | `startup.S`, `exceptionstub.S`, `cache-v7.S`, `memorymap.h`. |
| Estado TinyGo verificado | Backend ARM64, mas nenhum target Pi bare-metal. | Muitos targets Cortex-M, mas nenhum Pi/A-profile; startup e vetores não servem para A53. |
| Complexidade | Exige EL2/EL1 e vetor de 2 KiB, mas usa arquitetura-alvo nativa. | Evita transição AArch64, porém acrescenta modos de CPU, ABI 32-bit e ainda exige target próprio. |
| Decisão | Caminho principal. | Plano B somente se uma limitação concreta do TinyGo ARM64 for demonstrada. |

Não há evidência de suporte direto ao BCM2837 em nenhuma das arquiteturas. AArch32 não reduz a necessidade de boot, linker, runtime e suporte de máquina.

## 5. Ferramentas para Windows 11

### Estado constatado nesta máquina

| Ferramenta | Estado | Uso |
| --- | --- | --- |
| TinyGo 0.41.1 | Instalado via Scoop. | Base de estudo; será substituída ou executada a partir do fork pinado. |
| Go 1.26.5 | Instalado. | Construir o fork TinyGo. |
| Git 2.49.0 | Instalado. | Clonar/pinar TinyGo e registrar proveniência. |
| OpenOCD 0.12.0 | Instalado. | JTAG posterior, com probe e configuração da Pi. |
| CMake 4.0.1, Python e `mingw32-make` | Instalados. | Base para build de TinyGo/LLVM a partir do fonte. |
| `gdb-multiarch` | Instalado. | Depuração de ELF AArch64, a validar com JTAG. |
| `aarch64-none-elf-*` e `qemu-system-aarch64` | Não encontrados no `PATH`. | Recomendados para objdump/GDB e teste de boot. |

`make.exe` encontrado no `PATH` é o do Borland Delphi, não GNU Make. Não deve ser usado para construir TinyGo/Circle; para compilar o fork, usar Git Bash/MSYS2 com GNU Make, CMake, Ninja e Python. A documentação do TinyGo para build manual no Windows exige ambiente tipo Unix (Git Bash).

Para **compilar a imagem**, GCC Arm não é obrigatório se o fork permanecer em `ld.lld`/ `compiler-rt` distribuídos com TinyGo. Mesmo assim, Arm GNU Toolchain com prefixo `aarch64-none-elf-` é recomendado para `objdump`, GDB e comparação com Circle. Não usar `aarch64-linux-gnu`, que visa binários hospedados.

Usar saída `.bin`, por exemplo `tinygo build -o build/kernel8.bin ...`, e publicar esse arquivo como `kernel8.img` depois de validá-lo. A extensão controla o formato no TinyGo: `.bin` é cru, mas uma extensão genérica como `.img` produz executável regular/ELF. Isso evita gravar ELF no SD esperando que o firmware o carregue como imagem plana.

Para hardware: cartão microSD gravável, leitor, fonte estável e adaptador USB–UART TTL de 3,3 V para o console PL011. JTAG exigirá probe e ligação próprios, fora do primeiro marco.

## 6. Arquitetura inicial proposta

```text
.Circle_Tinygo/
  ANALISE_INICIAL.md
  docs/                       # decisões, mapa de memória, procedimentos
  toolchain/tinygo-rpi3/      # fork pinado do TinyGo (submódulo ou clone)
    targets/rpi3b-aarch64.json
    targets/rpi3b-aarch64.ld
    targets/rpi3b-boot.S
    targets/rpi3b-vectors.S
    src/runtime/runtime_rpi3b_aarch64.go
  platform/rpi3b/             # API Go pública e mínima
    mmio.go                   # acesso volátil e barreiras
    mailbox.go                # posterior: RAM ARM
    uart.go                   # posterior: PL011 polling
    timer.go                  # posterior: generic timer
  cmd/hello/                  # programa mínimo de aceitação
  boot/                       # templates config/lista de firmware
  scripts/                    # build, validação ELF/bin, cópia manual ao SD
```

O fork deve ser explícito e pinado, não uma cópia silenciosa da instalação em `C:\Users\...\tinygo`. A aplicação usará APIs de `platform/rpi3b`, sem importar `runtime`, assembly ou símbolos de linker. Nenhum driver do Circle faz parte desta arquitetura.

## 7. Plano de implementação por etapas

### Etapa 0 — proveniência e ambiente

1. Criar fork TinyGo pinado e registrar versão de Go/LLVM.
2. Registrar versão do firmware Raspberry Pi do cartão de teste.
3. Preparar script que gera ELF de depuração e BIN cru, confere tipo, entrada e tamanho, sem gravar cartão automaticamente.

**Aceitação:** artefatos reproduzíveis e relatório de mapa; nenhum boot ainda.

### Etapa 1 — target AArch64, boot de um núcleo e imagem plana

1. Adicionar target JSON, linker, `_start`, parking de secundários, limpeza de BSS, stack e vetor mínimo.
2. Integrar a entrada mínima do runtime para chegar a `main.main`; manter MMU/cache/IRQ desabilitados.
3. Produzir `kernel8.img` cru e testar primeiro em QEMU `raspi3b` quando disponível, depois em SD físico.

**Aceitação:** a Pi entra em `main`; handler seguro existe; layout não invade a região reservada.

### Etapa 2 — console e tempo mínimo

1. Implementar PL011 polling, GPIO 14/15 e `runtime.putchar`.
2. Expor `println`/panic no console e leitura do contador genérico com escala validada.
3. Manter `scheduler=none` e `gc=leaking`; medir heap.

**Aceitação:** `cmd/hello` imprime assinatura, versão e EL atual por UART; panic controlado não corrompe estado.

### Etapa 3 — RAM ARM, exceções e interrupções

1. Consultar mailbox para RAM ARM e configurar heap real.
2. Implementar handlers com ESR/ELR/FAR e controlador BCM2837.
3. Configurar generic timer e só então avaliar scheduler `tasks`.

**Aceitação:** exceção proposital gera diagnóstico, timer não perde ticks e heap respeita GPU.

### Etapa 4 — MMU/cache e base de drivers

1. Projetar tabelas/atributos e habilitar MMU/cache com testes de barreiras.
2. Converter **um** periférico por vez para driver Go próprio, usando documentação BCM2837 e Circle só como referência comportamental.
3. Só depois avaliar GPIO, I2C, SPI, framebuffer, SD e DMA.

**Aceitação:** RAM e MMIO têm atributos distintos; cada driver tem teste isolado.

### Etapa 5 — multicore, GC e estabilidade

1. Acordar secundários com spin table, cache maintenance e barreiras corretas.
2. Validar atômicos, stacks por núcleo e GC conservador.
3. Tornar QEMU, UART e JTAG parte da regressão documentada.

**Aceitação:** quatro núcleos executam tarefas controladas sem corrupção e o target é reproduzível no hardware.

## 8. Riscos e mitigação

| Risco | Impacto | Mitigação |
| --- | --- | --- |
| Não existe suporte TinyGo pronto para Pi 3 AArch64. | Alto: não há fallback de board suportada. | Fork pinado, target/runtime mínimos e validação a cada marco. |
| ABI interna do TinyGo/runtime muda. | Alto: pode falhar antes de `main`. | Fixar revisão; tratar atualização como migração testada. |
| Confundir ELF e imagem plana. | Alto: firmware não inicia a aplicação. | Gerar `.bin`, renomear após validar magic/layout e preservar ELF para GDB. |
| Estado inicial de EL/cache/cores varia entre firmware/emulador. | Alto: travamento silencioso. | Ler `CurrentEL`, mascarar exceções, estacionar secundários e usar barreiras; testar Pi real. |
| Heap invade GPU, mailbox, stacks ou DMA. | Alto: corrupção não determinística. | Consultar RAM ARM, reservar regiões no linker e adiar DMA. |
| UART depende de clocks/configuração de firmware. | Médio: diagnóstico indisponível. | PL011 polling, clock documentado e fallback JTAG. |
| QEMU não emula todos os periféricos Pi 3. | Médio: falso positivo. | QEMU só para boot/EL/linker; aceite final no hardware. |
| Copiar assembly/drivers do Circle. | Legal e técnico: Circle local está sob GPLv3 e o escopo proíbe conversão agora. | Implementação independente; se houver derivação, preservar licença e revisar distribuição. |
| Assumir que pacote Go equivale ao Go hospedado. | Médio: syscalls, arquivos e partes da stdlib exigem plataforma. | Começar com runtime mínimo e APIs próprias; ampliar apenas após testes. |

## Referências

- Circle local: `README.md`, `Rules.mk`, `circle.ld`, `boot/README`, `boot/config64.txt`, `lib/startup64.S`, `lib/sysinit.cpp`, `lib/exceptionstub64.S`, `lib/multicore.cpp`, `include/circle/memorymap64.h` e `doc/memorymap.txt`. Repositório: <https://github.com/rsta2/circle>.
- [Configuração e arquivos de kernel — Raspberry Pi](https://www.raspberrypi.com/documentation/computers/configuration.html).
- [Raspberry Pi 3 em AArch64/EL2 — Arm Trusted Firmware](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/plat/rpi3.rst).
- [TinyGo: instalação no Windows](https://tinygo.org/getting-started/install/windows/), [build e formatos de saída](https://tinygo.org/docs/reference/usage/subcommands/) e [target, GC e scheduler](https://tinygo.org/docs/reference/usage/important-options/).
- [TinyGo: build manual no Windows](https://tinygo.org/docs/guides/build/manual-llvm/) e [requisitos bare-metal](https://tinygo.org/docs/guides/build/additional-requirements/).


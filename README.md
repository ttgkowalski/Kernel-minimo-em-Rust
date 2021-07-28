# Kernel mínimo em Rust

Antes de qualquer coisa, esse tutorial é uma versão traduzida do tutorial do **[Philipp Oppermann](https://github.com/phil-opp)** que eu estou fazendo para praticar a linguagem de programação Rust, aprender sobre funcionamento em baixo nível de sistemas operacionais e de quebra praticar inglês. Então não deixe de visitar o **[tutorial oficial](https://os.phil-opp.com/minimal-rust-kernel/)** feito pelo Philipp Oppermann.

Para cada passo do roadmap terá uma versão traduzida do tutorial seguido.

##### [VERSIONS]
- *[rustc]*: rustc 1.56.0-nightly (d9aa28767 2021-07-24)
- *[cargo]*: cargo 1.55.0-nightly (cebef2951 2021-07-22)

##### [ADDITIONAL_RUST_COMPONENTS]
- rustup component add rust-src
- rustup component add llvm-tools-preview

##### [REFERÊNCIAS]
- *<a id="versionamento">[VERSIONAMENTO DO RUST]* - (https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#choo-choo-release-channels-and-riding-the-trains)
- <a id="start">*[_START function]* - (https://embeddedartistry.com/blog/2019/04/08/a-general-overview-of-what-happens-before-main/)

---

# Roadmap
- [x] Fazendo isso aqui funcionar
- [ ] Explorar os buffers de texto VGA
- [ ] Testes unitários

---

## **Fazendo isso aqui funcionar**

#### *Escolhendo a versão*:
Antes de escolher a versão, lembre-se de criar o projeto com o `cargo new <nome do projeto>` e navegar até ele.
Vamos iniciar escolhendo a versão do Rust que será utilizada. O Rust nos dispõe basicamente três versões da release:

- *[nightly]*: Contém os novos recursos experimentais da linguagem. Essa versão recebe uma nova release todos os dias e é a branch *master*.
- *[beta]*: A cada seis semanas, a equipe de desenvolvimento prepara uma nova release, a *beta*. Para lançar uma nova release beta os desenvolvedores criam uma nova branch a partir da *master*.
- *[stable]*: Seis semanas após a criação da release *beta*, os desenvolvedores fazem a mesma coisa, porém criando a branch *stable* a partir da *beta*.

Para mais detalhes sobre o *[versionamento do Rust](#versionamento)* , visite a documentação oficial.

<pre>
nightly: * - - * - - * - - * - - * - - * - * - *
                    |                          |
beta:                * - - - - - - - - *       *
                                       |
stable:                                *
</pre>


Para esse projeto, será necessário instalar a versão **nightly**. Eu recomendo fortemente a utilização do rustup para gerenciar as instalações do Rust. O rustup te permite instalar as versões nightly, beta e stable lado a lado.
Você pode facilmente utilizar o compilador nightly do Rust no diretório atual com o comando: 
```console
ttgkowalski@fedora:~$ rustup override set nightly
info: using existing install for 'nightly-x86_64-unknown-linux-gnu'
info: override toolchain for '/home/ttgkowalski/Development/Rust/<project_directory>' set to 'nightly-x86_64-unknown-linux-gnu'

  nightly-x86_64-unknown-linux-gnu unchanged - rustc 1.56.0-nightly (d9aa28767 2021-07-24)
```

#### *Especificando a arquitetura final para a compilação*:

O cargo pode compilar seu código Rust para diversas arquiteturas de CPUs através do parâmetro `--target`.

A lista de arquiteturas suportadas pelo cargo encontra-se no [capítulo 7](https://doc.rust-lang.org/nightly/rustc/platform-support.html) da documentação oficial do Rust **nightly**.

As arquiteturas suportadas incluem Windows, MacOS, Linux, Android ARM e até BSD.

Para este projeto, porém, nenhuma dessas arquiteturas parece se encaixar muito bem, mas, felizmente, o Rust nos permite definir o nosso próprio target através de um arquivo JSON de configuração. Abaixo, um exemplo de como se parece um JSON que define um target `x86_64-unknown-linux-gnu`:

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

A maioria desses campos são exigidos pelo [LLVM](https://llvm.org/) para gerar o código para a plataforma.
Nosso JSON vai ficar bem parecido com o JSON acima, então, vamos começar criando um arquivo com o nome do target na raiz do projeto. Eu coloquei `x86_64-mrk.json`, mas você pode colocar outro nome.
Nesse JSON, coloque o seguinte conteúdo:
```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
}
```

Repare que os campos "llvm-target" e "os" não possuem sequer a palavra "linux".
Isso porque nós estamos passando para o LLVM que esse código vai rodar em bare-metal, ou seja, direto em um servidor físico dedicado.

Agora vamos substituir o linker padrão da plataforma pelo [LLD](https://lld.llvm.org/) adicionando o seguinte conteúdo ao JSON:
```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

Após adicionar o LLD, será necessário também adicionar um campo dizendo que o target não suporta stack unwinding no caso de panic. Para isso, adicione ao JSON:
```json
"panic-strategy": "abort",
```
Em resumo, [stack unwinding(Rust)](https://doc.rust-lang.org/nomicon/unwinding.html) é o processo que ocorre no lançamento de um "panic"(Exception), onde o "panic" fará que a execução normal da thread pare e chame "destructors"(destruidores) como se cada função instantaneamente retornassem seus resultados. Esse processo é importante para otimizar o gerenciamento de memória prevenindo vazamentos.

* [IMPORTANTE]:
Você também poderia adicionar o "panic-strategy" no Cargo.toml, mas para esse projeto é necessário que ele esteja nesse JSON de custom-target. Isso porque lá na frente vamos recompilar a lib `core`.

Para lidar com as [interrupções](https://linux-kernel-labs.github.io/refs/heads/master/lectures/interrupts.html)(Eventos que alteram o fluxo padrão de execução de um software) ao nível do kernel, precisamos desabilitar a redzone. Para fazer isso, adicione o seguinte conteúdo ao JSON:

```json
"disable-redzone": true,
```

A redzone, em resumo, é uma importante otimização da
**ABI**(Application Binary Interface) do **System V** que permite que as funções usem temporariamente os 128 bytes abaixo de sua estrutura de pilha sem ajustar o ponteiro da pilha.

Caso queira uma explicação mais detalhada, você poderá encontrá-la [neste post](https://os.phil-opp.com/red-zone/) do próprio Philipp.

Bom, já estamos na metade do caminho.

Antes de irmos para o código Rust, precisamos ativar e desativar algumas features específicas do nosso target adicionando o seguinte campo ao JSON:
```json
"features": "-mmx,-sse,+soft-float"
```

Para ativar ou desativar uma feature precisamos adicionar um sinal de mais(+) ou de menos(-), respectivamente. Repare que cada item é separado por uma vírgula e não pode haver espaços, senão, o LLVM vai falhar ao interpretar as flags.

As features que desativamos `mmx` e `sse` determinam o suporte ao [Single Instruction Multiple Data (SIMD)](https://pt.wikipedia.org/wiki/SIMD), o qual pode acelerar significativamente os programas. Porém, utilizar diversos registros com o SIMD pode nos conduzir a sérios problemas de performance.

O problema de desabilitar o SIMD é que operações com ponto flutuante(decimais) na arquitetura x86_64 exige o SIMD por padrão. Para resolver esse problema, basta ativar a feature `soft-float`, que emula todas as operações com decimais através de funções de software baseadas em inteiros.

Caso queira uma explicação mais detalhada, você poderá encontrá-la [neste post](https://os.phil-opp.com/disable-simd/) do próprio Philipp.

Agora, colocando tudo isso junto, temos o seguinte resultado:

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float"
}
```

#### *Construindo nosso kernel*:

Para compilar para o nosso próprio target, vamos utilizar uma convenção geral no nosso código. Isso significa que o nosso ponto de entrada, ou seja, nossa função principal(entrypoint) não será a main como de costume, mas sim a função _start.

No tutorial do Philipp ele disse que usaríamos essa convenção do Linux porque o LLVM exigia essa convenção e ele não sabia exatamente o porquê. Mas eu andei pesquisando e eis aqui o motivo:

O uso do _start é meramente convencional. A função principal varia entre os sistemas, compiladores e bibliotecas padrões. Por exemplo, o OS X contém apenas aplicações vinculadas dinamicamente e o próprio [loader](https://embeddedartistry.com/fieldmanual-terms/program-loader/) se encarrega das configurações, então a função principal é realmente a main.

Você poderá encontrar estas informações nessas referências: [[_START function]](#start)

Caso queira uma alternativa, pode encontrá-la [nesse post](https://stackoverflow.com/questions/67918256/rust-custom-bare-metal-compile-target-linker-expects-start-symbol-and-discar) do Stackoverflow

Vamos ao código!
Adicione o seguinte conteúdo ao arquivo `src/main.rs`

```rust
#![no_std] // Desvincula a biblioteca padrão do Rust
#![no_main] // Desabilita todas as funções de entrada no Rust-level

use core::panic::PanicInfo;

/// Essa função é chamada quando ocorre algum panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle] // Isso diz para o Rust não alterar o nome desta função
pub extern "C" fn _start() -> ! {
    // Esta é a função de entrada, uma vez que o linker proucura por uma função.
    // Repare que o nome desta função é '_start', ou seja, essa é a função de entrada, a principal.
    loop {}
}
```

Tecnicamente, tudo certo pra dar errado, vamos compilar nosso kernel utilizando o cargo com o parâmetro `--target` passando nosso JSON.
```condole
ttgkowalski@fedora:~$ cargo build --target x86_64-mrk.json

error[E0463]: can't find crate for `core`
```

E deu errado mesmo!

Basicamente, a mensagem de erro está nos informando que o Rust não conseguiu encontrar a biblioteca `core`, que é uma biblioteca entregue junto com o compilador Rust como uma biblioteca precompilada. Ou seja, ela é entregue na lib `std`, a qual desativamos aqui:

```rust
// src/main.rs

#![no_std]
```

É aí que entra a feature [build-std](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#build-std) do cargo. Ela permite **recompilar** a lib `core` e várias outras bibliotecas padrões. Lembrando que essa é uma função nova, pouco testada e não finalizada, por isso está marcada como 'unstable' e está disponível apenas na versão **nightly** do Rust.

Para utilizar essa função, precisamos criar um arquivo de configuração do cargo em `.cargo/config.toml` e adicionar o seguinte conteúdo:

```toml
[unstable]
build-std = ["core", "compiler_builtins"]
```

Isso indica ao cargo que ele tem que recompilar as bibliotecas `core` e `compiler-buildins`. Mas para o cargo recompilar essas bibliotecas, ele precisa de acesso ao código fonte; Vamos dá-lo acesso ao código fonte adicionando o componente *rust-src* através do comando: `rustup component add rust-src`

Quase lá!

##### Particularidades relacionadas à memória

O compilador do Rust assume que algumas funções padrões estão disponívels em todos os sistemas. A maioria dessas funções são providas pelo `compiler-buildin`, que nós acabamos de recompilar. No entanto, existem algumas funções relacionadas à memória que não estão habilitadas por padrão porque geralmente são providas pela biblioteca do C do próprio sistema. Estas funções incluem o [memset](https://www.tutorialspoint.com/c_standard_library/c_function_memset.htm), [memcpy](https://www.tutorialspoint.com/c_standard_library/c_function_memcpy.htm) e o [memcmp](https://www.tutorialspoint.com/c_standard_library/c_function_memcmp.htm).

Já que não podemos vincular nosso sistema com nenhuma biblioteca C do sistema, precisamos de uma alternativa para prover essas funções ao compilador.

Felizmente, o `compiler-buildins` já possui implementações para todas as funções que vamos usar, elas só ficam desabilitadas por padrão para não colidir com as implementações da biblioteca do C.

Para fazer isso, vamos adicionar essa biblioteca para ser habilitada por padrão pelo cargo na hora de compilar nosso kernel.


```toml
# .cargo/config.toml

[unstable]
build-std-features = ["compiler-builtins-mem"]
build-std = ["core", "compiler_builtins"]
```

Agora sim, configurações finalizadas!

Mas nosso sistema ainda não faz nada, vamos coloca-lo para escrever alguma coisa na tela.

##### Escrevendo na tela

A maneira mais fácil de escrever algo na tela é através do [VGA text buffer](https://en.wikipedia.org/wiki/VGA_text_mode). o VGA text buffer é uma área especial mapeada para o hardware VGA que contém o conteúdo a ser mostrado na tela. Geralmente, ele consiste em 25 linhas, onde cada uma contém 80 células. Cada célula mostra um caractere ASCII com uma cor de fundo e uma do caractere.

Para mostrar a frase  "Hello World!", nós apenas precisamos saber que o buffer está localizado no endereço `0xb8000` e que cada célula consiste em um caractere em ASCII, uma cor de fundo, e a cor do caractere.

No código fica algo mais ou menos assim:

```rust
static HELLO: &[u8] = b"Hello World!"; // Isso aqui está logo acima da função principal

#[no_mangle]
pub extern "C" fn _start() -> ! { // E essa é a função principal
    let vga_buffer = 0xb8000 as *mut u8;

    for (i, &byte) in HELLO.iter().enumerate() {
        unsafe {
            *vga_buffer.offset(i as isize * 2) = byte;
            *vga_buffer.offset(i as isize * 2 + 1) = 0xb;
        }
    }

    loop {}
}
```

Basicamente, nós transformamos o inteiro `0xb8000` em um ponteiro. Então, iteramos sobre os bytes da [byte string](https://doc.rust-lang.org/reference/tokens.html#byte-string-literals) *[static](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime)* `HELLO`.

Para o corpo da do loop, utilizamos o método [offset](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset) para escrever a string byte e sua respectiva cor representada em byte(`0xb` equivale à cor ciano claro).

Repare que, todas as inscrições na tela estão ao dentro de um bloco chamado [unsafe](https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html).

A razão para isso é que o compilador do Rust não pode provar que os ponteiros que nós criamos são válidos. Eles podem apontar para qualquer lugar e ocasionar uma corrupção de dados. Ao coloca-los dentro de um bloco `unsafe`, estamos dizendo ao compilador Rust que temos absoluta certeza de que as operações são válidas.
Vale ressaltar que, colocar um código dentro de um bloco `unsafe` não desativa as verificações de segurança do Rust. O `unsafe` apenas nos permite fazer estas [cinco coisas adicionais](https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers).

#### Rodando o Kernel

Para transformar nosso kernel em uma imagem bootável, precisamos ligá-lo a um bootloader. Para isso, nós vamos utilizar a *crate* [bootloader](https://crates.io/crates/bootloader). Essa crate implementa uma BIOS básica sem utilizar nenhuma dependência do C, apenas Rust e assembly.

Vamos adicionar ao `Cargo.toml` o seguinte conteúdo:

```toml
[dependencies]
bootloader = "0.9.8"
```

E rodar o seguinte comando: `cargo install bootimage`.

Para o `bootimage` rodar, você precisa instalar o componente *llvm-tools-preview* do rustup. Para fazer isso, basta rodar o comando: `rustup component add llvm-tools-preview`.
  
O bootimage utiliza o target padrão do cargo, mas nós estamos utilizando um target customizado, então, para evitar erros na hora da compilação, adicione o nome do JSON que você criou no topo do arquivo `.cargo/config.toml`, ficará algo como isso:

```toml
[build]
target = "x86_64-mrk.json.json"
```

Agora sim podemos compilar nosso kernel. Para isso, execute o seguinte comando:

```console
Não se assuste com os milhões de warnings que irão aparecer.

ttgkowalski@fedora:~$ cargo bootimage
WARNING: `CARGO_MANIFEST_DIR` env variable not set
Building kernel
    Finished dev [unoptimized + debuginfo] target(s) in 1.01s
    .
    .
    .
Finished release [optimized + debuginfo] target(s) in 0.26s
Created bootimage for `minimal-rust-kernel` at `< path do diretório do seu projeto>/target/< seu json target >/debug/bootimage-minimal-rust-kernel.bin`
```

Agora sim, sistema finalizado!

Para testar seu sistema, basta iniciar por alguma vm ou gravar em um pendrive bootável.

#### Rodando o sistema pelo QEMU

```console
qemu-system-x86_64 -drive format=raw,file=target/< seu target >/debug/< seu target >.bin
```

E é isso!

# E agora?

O próximo passo é explorar o VGA text buffer com mais detalhes e escrever uma interface segura para isso.
E também adicionar suporte ao macro `println`.

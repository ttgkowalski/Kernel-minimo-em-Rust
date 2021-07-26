# Kernel mínimo em Rust

Antes de qualquer coisa, esse tutorial é uma versão traduzida do tutorial do **[Philipp Oppermann](https://github.com/phil-opp)** que eu estou fazendo para praticar a linguagem de programação Rust, aprender sobre funcionamento de baixo nível do linux e de quebra praticar inglês. Então não deixe de visitar o **[tutorial oficial](https://os.phil-opp.com/minimal-rust-kernel/)** feito pelo Philipp Oppermann.

Para cada passo do roadmap terá uma versão traduzida do tutorial seguido.

##### [VERSIONS]
- *[rustc]*: rustc 1.56.0-nightly (d9aa28767 2021-07-24)
- *[cargo]*: cargo 1.55.0-nightly (cebef2951 2021-07-22)

##### [ADDITIONAL_RUST_COMPONENTS]
- rustup component add rust-src
- rustup component add llvm-tools-preview

##### [REFERENCES]
- *[<a id="versionamento">VERSIONAMENTO DO RUST]* - (https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#choo-choo-release-channels-and-riding-the-trains)

---

# Roadmap
- [x] Fazendo isso aqui funcionar
- [ ] Explorar os buffers de texto VGA
- [ ] Testes unitários

---

## **Fazendo isso aqui funcionar**

##### *Escolhendo a versão*:
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

##### *Especificando a arquitetura final para a compilação*:

O cargo pode compilar seu código rust para diversas arquiteturas de CPUs através do parâmetro `--target`.

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

A partir desse momento as coisas começam a fazer mais sentido e a tomar forma, repare que os campos "llvm-target" e "os" não possuem sequer a palavra "linux".
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

Caso queira uma explicação mais detalhada, você poderá encontrá-la [neste post](https://os.phil-opp.com/red-zone/) do próprio Philip.


# Kernel mínimo em Rust

Antes de qualquer coisa, esse tutorial é uma versão traduzida do tutorial do **[Philipp Oppermann](https://github.com/phil-opp)** que eu estou fazendo para praticar a linguagem de programação Rust, aprender sobre funcionamento de baixo nível do linux e de quebra praticar inglês. Então não deixe de visitar o **[tutorial oficial](https://os.phil-opp.com/minimal-rust-kernel/)** feito pelo Philipp Oppermann.

Para cada passo do roadmap terá uma versão traduzida do *[tutorial](#tutorial)* seguido.

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

Para mais detalhes sobre o *[versionamento do Rust](#tutorial)* , visite a documentação oficial.

<pre>
nightly: * - - * - - * - - * - - * - - * - * - *
                    |                          |
beta:                * - - - - - - - - *       *
                                       |
stable:                                *
</pre>


Para esse projeto, será necessário instalar a versão nightly.


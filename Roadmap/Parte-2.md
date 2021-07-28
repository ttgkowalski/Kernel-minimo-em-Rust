## **Explorarando os buffers de texto**


O [modo de texto ](https://en.wikipedia.org/wiki/VGA_text_mode) do [VGA](https://en.wikipedia.org/wiki/Video_Graphics_Array)(Video Graphics Array) é uma maneira simples de mostrar algum texto na tela. Nessa parte do nosso roadmap, nós vamos criar uma interface que implementa o uso do VGA text mode de maneira simples e segura encapsulando todas as funções que podem quebrar o sistema em um módulo separado. Também iremos implementar um suporte aos macros de formatação([formatting macros](https://doc.rust-lang.org/std/fmt/#related-macros)).

#### O buffer de texto do VGA
Para facilitar a compreensão, o seu hardware de VGA é a sua placa de vídeo.

Para mostrar um caractere na tela através do modo de texto do VGA, é necessário escrevê-lo no buffer de texto do hardware de VGA. Basicamente, o buffer de texto do VGA é um array de duas dimensões, com geralmente 25 linhas e 80 colunas, o qual é renderizado diretamente na tela.

No início eu demorei um pouco para entender, mas vou tentar simplificar as coisas.


Cada caractere é representado por 2 bytes alinhados como uma palavra de 16 bits acessível pela CPU em uma única operação.

O primeiro byte representa um ponto de código na codificação [ASCII](http://web.alfredstate.edu/faculty/weimandn/miscellaneous/ascii/ascii_index.html) e é chamado de *character byte*. Na verdade, não é bem ASCII, é [code page 437](https://en.wikipedia.org/wiki/Code_page_437). O code page 437 é o conjunto de caracteres original do [IBM PC](https://en.wikipedia.org/wiki/Code_page_437). Basicamente, o code page 437 inclui todos os caracteres da codificação ASCII, extendido de letras acentuadas, algumas letras do alfabeto grego, ícones e alguns simbolos desenhados com linhas.

Já o segundo byte representa os atributos do caractere, ou seja, como ele deve ser mostrado na tela. Ele é chamado de *attribute byte*.

Abaixo, uma foto desse conjunto de caracteres:
![https://en.wikipedia.org/wiki/Code_page_437#/media/File:Codepage-437.png](./images/Codepage-437.png)

E coloquei também uma tabela exemplificando como é descrito um caractere através do VGA text buffer logo abaixo:
<table>
    <tr>
        <th colspan="8">Byte de Atributo</th>
        <th colspan="8">Byte de Caractere</th>
    </tr>
    <tr>
        <th colspan="1">Piscar ou Cor</th>
        <th colspan="3">Cor de fundo</th>
        <th colspan="4">Cor do caractere</th>
        <th colspan="8">Código do caractere</th>
    </tr>
    <tr>
        <td>Bit 15</td>
        <td>Bit 15</td>
        <td>Bit 13</td>
        <td>Bit 12</td>
        <td>Bit 11</td>
        <td>Bit 10</td>
        <td>Bit 9</td>
        <td>Bit 8</td>
        <td>Bit 7</td>
        <td>Bit 6</td>
        <td>Bit 5</td>
        <td>Bit 4</td>
        <td>Bit 3</td>
        <td>Bit 2</td>
        <td>Bit 1</td>
        <td>Bit 0</td>
    </tr>
    <tr>
        <td>0</td>
        <td>1</td>
        <td>1</td>
        <td>0</td>
        <td>1</td>
        <td>0</td>
        <td>1</td>
        <td>0</td>
        <td>0</td>
        <td>1</td>
        <td>0</td>
        <td>0</td>
        <td>0</td>
        <td>0</td>
        <td>0</td>
        <td>1</td>
    </tr>
</table>

O **Byte de Atributo** como sendo um conjunto de 8 bits, nos descreve um byte que representa os atributos do caractere. Os primeiros 3 bits [8-11] do byte de atributo representam a cor do primeiro plano (caractere), os 3 próximos bits [12-14] representam a cor de fundo do caractere.

E sobrou o último bit [15]. Esse bit pode ser utilizado para definir se este caractere deve piscar ou não, mas também pode ser utilizado como um quarto bit de cor de fundo (o que nos permitiria utilizar todas as 16 cores disponíveis a serem usadas como cor de fundo). 

O **Byte de Caractere** também é um conjunto de 8 bits, ou seja, um byte. Mas este byte aponta para um código da tabela code page 437. Nesse exemplo, ele nos aponta o código 65 da tabela, que equivale ao caractere 'A' maiúsculo.


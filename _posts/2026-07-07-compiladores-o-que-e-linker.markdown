---
layout: post
title: "Compiladores: o que é um linker e como funciona"
date: 2026-07-07 16:00:00 -0300
categories: compiladores linker baremetal
---

Nunca escrevi um blog antes, e para o primeiro post do meu primeiro blog, estou meio perdida em como começar, mas vamos lá. Estou escrevendo este artigo na noite de uma terça feira, enquanto escuto a discografia do Pink Floyd pela **INTEGER_OVERFLOW** vez. Para quem não me conhece, sou a Aria, costumo postar no LinkedIn várias coisas sobre computação e matemática, com bastante foco em desenvolvimento de software low-level. Há muito tempo prometo começar com um blog, e finalmente estou cumprindo essa promessa. E como assunto do primeiro post, quero falar algo que sempre toquei no assunto mas nunca me aprofundei, graças ao limite de caracteres do LinkedIn que não me permite: **Linkers**.

Quem me acompanha já me viu falando sobre compiladores diversas vezes, um exemplo é [neste post aqui][post-anthropic] em que faço algumas considerações sobre o compilador em C da Anthropic escrito puramente com agentes. Futuramente quero cobrir todas as etapas do processo de compilação aqui no blog, uma em cada post dedicado, mas hoje quero dedicar minha escrita ao Linker. Para relembrar, o processo de compilação de código em linguagem alto nível para linguagem de máquina contece em quatro etapas[^dragonbook]:

1. Preprocessamento: Aqui são removidos comentários, feitas substituições de strings etc
2. Tradução: O código fonte é convertido em instruções de CPU (Várias otimizações ocorrem aqui)
3. Assembly: Tradução das instruções de CPU para binário
4. Linker: Organização do executável final

Sim, eu sei que a etapa de Link é a última, mas quero começar o blog com ela por duas razões. A primeira é que, de todas elas, é a que menos falei até hoje, mesmo sendo a mais interessante de todas (pra mim!). A segunda é que, quando você lê que o linker organiza o binário, dá a entender que ele apenas troca coisas de lugar ou algo do tipo, sendo que o linker é, na verdade, o responsável por definir, por exemplo, o endereço de memória onde seu software será executado, os limites da stack etc[^dragonbook].

Pois é, imagino que pra quem nunca tinha ouvido falar antes, agora deve ter uma noção melhor da importância do linker.

O linker é a etapa da compilação responsável por pegar todos os compilados intermediários (no caso de C os arquivos .o ou .a), e transformar em um único executável. Além disso ele também resolve toda a symbol table do seu software e as dependências. Se você tem declaradas as variáveis `foo` e `bar`, é ele quem decide o endereço final absoluto de memória onde essas variáveis ficarão localizadas. Se você importou bibliotecas externas, o linker é o responsável por alocar também a memória dos recursos utilizados por elas, endereço de memória das funções, tudo.

Normalmente, quando estamos escrevendo software que vai rodar em cima de um sistema operacional, é extremamente raro precisarmos escrever o script do linker, independente de estarmos trabalhando em C, Rust, Go etc. O compilador já faz esse trabalho por nós (e muito melhor do que uma pessoa conseguiria). Então, para este post aqui, resolvi ir um pouco abaixo do low-level tradicional que costumo postar e ir direto para o bare-metal. Escolhi um STM32, mais especificamente o STM32F103C8T6 também conhecido como blue-pill, pra criar esse exemplo. Escolhi ele porque é a placa que costumo utilizar nos meus projetos pessoais, e portanto a que tenho disponível aqui. Essa placa possui um processador ARM Cortex M3, e vai servir muito bem para mostrar como funciona o linker. Como estarei criando um driver simples do zero, utilizando apenas a datasheet da placa para mapear a memória na mão e manipular manualmente os registradores, o script do linker é bem mais curto e simples do que um script gerado por compilador, que está referenciando várias funções da stdlib, syscalls para o Kernel etc.

Não deixarei aqui neste post tudo que fiz para compilar a mão e gravar o código na memória Flash da placa, mas quem tiver interesse me sinalize que posso pensar em fazer um post dedicado a isso no futuro. Existem diversas IDEs que são super simples e prontas para programar essas placas, quase um plug-and-play, mas não é isso que vou fazer aqui. Pode não ser muito prático, mas criar seu próprio código manipulando o hardware, sem IDEs nem nada, utilizando apenas a datasheet fornecida pelo fabricante é uma das coisas mais recompensadoras na computação! Para quem tiver interesse, aqui está a [datasheet][datasheet] desta placa específica, que utilizei para criar o código C e o script do Linker.

Retomando o tema do linker, imaginem o seguinte. Acabamos de colocar as mãos nessa plaquinha bonitinha e queremos testar. Qual a coisa mais simples que podemos fazer? Essa placa possui um LED embutido, que a datasheet nos informa ser o pino número 13. Podemos então, para fins de teste, tentar fazer este pino piscar.

Em tese é muito simples, mas como estamos fazendo tudo do zero, sem IDEs, libs externas nem nada, temos um problema. O STM32 não roda em um OS, então precisaremos criar uma Interrupt Vector Table (IVT) na mão. A IVT é uma tabela de ponteiros que mapeia sinais do hardware a funções de software. Um exemplo é quando você aperta `Ctrl + C` no seu terminal. Na IVT do seu Kernel está registrado que quando esse sinal do hardware for detectado, um determinado endereço de memória deve ser chamado. Nesse endereço está a função que interrompe a execução do seu programa.

Deixarei aqui um exemplo da IVT que codei para este nosso driver. Como não é o foco deste post não entrarei em detalhes, mas como disse antes, se for do interesse de vocês, posso fazer um post apenas sobre bare-metal no futuro:

````c
int main();
void Reset_Handler(void);
void HardFault_Handler(void) __attribute__((weak));

__attribute__((section(".isr_vector")))
void (*const g_pfnVectors[])(void) = {
    (void (*)(void))0x20005000,
    Reset_Handler,
    HardFault_Handler,
};

void Reset_Handler(void) {
    extern unsigned int _sdata, _edata, _la_data;
    unsigned int *pSrc = &_la_data;
    unsigned int *pDest = &_sdata;
    while (pDest < &_edata) *pDest++ = *pSrc++;

    extern unsigned int _sbss, _ebss;
    pDest = &_sbss;
    while (pDest < &_ebss) *pDest++ = 0;

    main();
    while(1);
}

void HardFault_Handler(void) { while(1); }
````

E aqui deixarei o código para piscar o LED:

````c
#define PERIPH_BASE     0x40000000
#define APB2PERIPH_BASE (PERIPH_BASE + 0x10000)
#define AHBPERIPH_BASE  (PERIPH_BASE + 0x20000)

#define RCC_BASE        (AHBPERIPH_BASE + 0x1000)
#define GPIOC_BASE      (APB2PERIPH_BASE + 0x1000)

#define RCC_APB2ENR     (*((volatile unsigned int*)(RCC_BASE + 0x18)))
#define GPIOC_CRH       (*((volatile unsigned int*)(GPIOC_BASE + 0x04)))
#define GPIOC_ODR       (*((volatile unsigned int*)(GPIOC_BASE + 0x0C)))

void delay(volatile unsigned int count) {
    while(count--);
}

int main(void) {
    RCC_APB2ENR |= (1 << 4);

    GPIOC_CRH &= ~(0xF << 20);
    GPIOC_CRH |= (0x3 << 20);

    while(1) {
        GPIOC_ODR &= ~(1 << 13);
        delay(500000);
        GPIOC_ODR |= (1 << 13);
        delay(500000);
    }
}

void SystemInit(void) {  }
````

Para não deixar tudo sem explicação: Como informado na documetação da placa, a memória RAM começa a partir do endereço `0x20000000`, e por isso declarei o início da Stack bem próximo. A datasheet também nos informa que a IVT deve ficar no endereço de memória `0x08000000`, que também é o início da memória flash, e temos também as informações dos pinos e registradores, que utilizei para escrever os endereços de memória corretos.

Ok, temos nosso código, hora de escrever nosso linker!

````ld
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM (rwx)   : ORIGIN = 0x20000000, LENGTH = 20K
}
ENTRY(Reset_Handler)

SECTIONS {
    .isr_vector : { KEEP(*(.isr_vector)) } > FLASH
    .text : { *(.text) *(.text*) } > FLASH
    .rodata : { *(.rodata) *(.rodata*) } > FLASH
    _la_data = LOADADDR(.data);
    .data : {
        _sdata = .;
        *(.data) *(.data*);
        _edata = .;
    } > RAM AT> FLASH
    .bss : {
        _sbss = .;
        *(.bss) *(.bss*) *(COMMON);
        _ebss = .;
    } > RAM
}
````

Essa aqui é, na minha opinião, a parte mais linda da etapa de compilação. Vamos ver o que está acontecendo:

- Campo MEMORY
    - Alí definimos duas variáveis: FLASH e RAM. Memória flash é aquela usada como "disco", é onde ficará registrado o binário compilado do nosso software. Dizemos no linker que essa memória iniciará no endereço `0x08000000`, e que ela tem tamanho de 64Kb. Ambas as informações nos foram fornecidas pela datasheet. Reparem que alí definimos a Flash como read only e a RAM como read and write.
    - Na variável RAM seguimos o mesmo padrão, utilizamos as informações da datasheet para mapear seu tamanho e onde começa.
- Campo ENTRY
    - Aqui estamo dizendo para o linker o seguinte: Assim que o hardware bootar, o primeiro código executado é essa função. O hardware do Cortex-M3 automaticamente procura pelo endereço do `Reset_Handler` dentro da nossa IVT (lembra dela?) para saber por onde começar a rodar nosso programa.
- Campo SECTIONS
    - É exatamente aqui que mapeamos onde tudo vai ficar. Deixamos claro no linker, assim como manda a datasheet, que a IVT ficará no primeiro endereço da memória Flash
    - Depois vem a sessão `.text`. Na linguagem do linker, `.text` é a sessão onde fica todo o código compilado do nosso software, também armazenado na memória Flash.
    - `.rodata` é uma abreviação de Read Only data. Todas as constantes do nosso software ficarão armazenadas neste segmento de memória.
    - `.data` aqui já é a nossa primeira sessão de memória alocada na RAM. Todas as variáveis inicializadas vivem nesta sessão. Mas detalhe: essas variáveis começam a vida gravadas na Flash (para não serem perdidas quando a placa desligar). É o nosso `Reset_Handler` que as copia para a RAM (usando os símbolos _sdata e _edata criados aqui no linker) logo antes do main() rodar.
    - `.bss` esse bloco é bem interessante. Aqui ficam todas as variáveis globais e variáveis estáticas não inicializadas. Elas são armazenadas em um único bloco contínuo de RAM

Nesse script, estamos basicamente dizendo ao compilador onde tudo deve ser posicionado, para que o software consiga se comunicar perfeitamente com o hardware. O linker é quem faz o software entender o hardware.

Deixei aqui uma foto da placa funcionando. A placa preta é um conversor de USB para UART, que precisei utilizar para escrever direto da porta USB do meu Mac para a STM. Espero que tenham gostado, e que tenha sido possível aprender um pouco mais sobre como os softwares são compilados!

![imagem stm]({{"/assets/images/linker_baremetal_example.jpeg" | relative_url}})

## Bibliografia

[^dragonbook]: AHO, A. V.; LAM, M. S.; SETHI, R.; ULLMAN, J. D. Compilers: principles, techniques, and tools. 2. ed. Boston: Pearson Addison-Wesley, 2006. p. 3.

[post-anthropic]: https://www.linkedin.com/feed/update/urn:li:activity:7425672799689756673
[datasheet]: https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://www.st.com/resource/en/datasheet/stm32f103c8.pdf&ved=2ahUKEwjqwKDLzcGVAxVUGbkGHSyGChsQFnoECA8QAQ&usg=AOvVaw0rd6I_7fuhTLdZOoycvGV5

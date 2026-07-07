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

Sim, eu sei que a etapa de Link é a última, mas quero começar o blog com ela por duas razões. A primeira é que, de todas elas, é a que menos falei até hoje, mesmo sendo a mais interessante de todas (pra mim!). A segunda é que, quando você lê que o linker organiza o binário, dá a entender que ele apenas troca coisas de lugar ou algo do tipo, sendo que o linker é, na verdade, o responsável por definir, por exemplo, o endereço de memória onde seu software será executado, o que fica na stack ou no heap etc[^dragonbook].

Pois é, imagino que pra quem nunca tinha ouvido falar antes, agora deve ter uma noção melhor da importância do linker.

![imagem stm]({{"/assets/images/linker_baremetal_example.jpeg" | relative_url}})

## Bibliografia

[^dragonbook]: AHO, A. V.; LAM, M. S.; SETHI, R.; ULLMAN, J. D. Compilers: principles, techniques, and tools. 2. ed. Boston: Pearson Addison-Wesley, 2006. p. 3.

[post-anthropic]: https://www.linkedin.com/feed/update/urn:li:activity:7425672799689756673

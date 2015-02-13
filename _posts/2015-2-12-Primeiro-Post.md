---
layout: post
title: Primeiro Post
---

Olá! Esse é o primeiro post do meu blog, em que pretendo falar sobre programação, tecnologia e outras coisas nessa linha.

![_config.yml]({{ site.baseurl }}/images/posts/one/first-blog-post-winning.jpg)

Para a criação do blog utilizei o [Jekyll](http://jekyllrb.com/) por alguns motivos:

1. Prático e fácil, além de me permitir customizar o blog como desejar.

2. GitHub Pages, ou seja, não se preocupar com hospedagem e ter várias vantagens com isso.

3. "Commitar" seus próprios posts é muito legal :smiley:

4. Mais um desafio :computer:

A parte mais interessante do projeto como um todo foi criar uma estrutura bilíngue para o blog inteiro. Existem alguns plugins
mas não queria utilizar nada do tipo. Encontrei alguns posts pela web também que me serviram de base. No fim acabei criando uma
estrutura para manter isso:

![_config.yml]({{ site.baseurl }}/images/posts/one/blog-structure.png)

Os posts em português ficam dentro de `_posts` como devem ficar mesmo. Os diferentes arquivos de layout cuidam das diferenças entre
as páginas. Dessa forma consigo manter separado pela categoria `EN` o conteúdo em inglês.

Bem por hoje é isso. Logo mais trarei alguns posts interessantes de verdade (é sério).


### Alguns links interessantes para quem quiser saber mais sobre Jekyll:

- [Tutorial simples sobre o assunto.](http://code.tutsplus.com/tutorials/using-jekyll--cms-20956)
- [Melhor post que achei sobre manter um blog bilíngue](http://kleinfreund.de/en/2014/08/jekyll-multilingual/). Utilizei muitas
coisas desse post, mas por incrível que pareça uma delas não foi o layout :stuck_out_tongue_winking_eye:

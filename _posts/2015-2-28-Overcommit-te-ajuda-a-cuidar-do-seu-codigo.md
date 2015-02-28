---
layout: post
title: Overcommit te ajuda a cuidar do seu código
---

Já ouviu falar da _gem_ [Overcommit](https://github.com/causes/overcommit)? A função dela é ajudar a manter os padrões de
qualidade, definidos por você ou pelas diversas comunidades de linguagens de programação mundo afora.

Por definição, Overcommit é uma ferramenta para administrar e configurar _Git hooks_, que são ações pré configuradas
para executar um _script_ assim que ocorrerem. Encontramos algumas dessas ações na _gem_, sendo as mais importantes:
`commit-msg`, `pre-commit` e `post-checkout`. As duas primeiras ocorrem durante um _commit_, enquanto a última é
acionada sempre que ocorrer uma mudança de _branch_ via _checkout_ por exemplo. É possível criar o seu próprio _hook_,
mas só isso demandaria um post próprio.

## Mas o que ele faz?

Agora vamos ao que realmente importa: o que Overcommit pode fazer por mim? É o seguinte, sempre que você fizer um
_commit_ por exemplo, vários _scripts_ vão ser rodados. Esses _scripts_ vão checar desde a sua mensagem no _commit_
até a sintaxe do seu código. Vou detalhar alguns deles aqui:

* __CommitMsg - HardTabs, RussianNovel TrailingPeriod__: Esses três _scripts_ vão cuidar para ver como está sua mensagem
do _commit_, checando a formatação, tamanho e caracteres do mesmo.

* __Brakeman__: Checar por falhas de segurança

* __BundleCheck__: Checar as dependências do seu _Gemfile_

* __Coffee/Css/Html/Scss/Haml Lint__: Analisa a sintaxe de cada um desses, além de muitos outros, incluindo linguagens
de programação como Java, Ruby, Python etc.

* __MergeConflicts__: Sabe aquele conflito que você esqueceu de resolver em algum arquivo? Então :relieved:

* __PryBinding__: Esqueceu de um `pry` lá no `Controller`? Relaxa, ninguém vai te zoar agora. :joy:

* __Rubocop__: Talvez o mais importante para os rubystas. [Rubocop](https://github.com/bbatsov/rubocop) é um
analisador de código estático que segue vários padrões de código criados pela comunidade e mantidos no
[Ruby Style Guide](https://github.com/bbatsov/ruby-style-guide). Ele é customizável, então você não precisa seguir
tudo caso não queira.

## Instalando e Configurando

Para instalar é simples:

`gem install overcommit`

Depois, dentro do repositório que pretende utilizar:

`overcommit --install`

Isso irá criar o arquivo `.overcommit.yml` no seu repositório. Basta seguir as instruções ali e da página da _gem_
para configurar os _hooks_ que achar necessário para o seu projeto!

## Algumas dicas

Você pode bloquear um _commit_ ou apenas avisar que ele feriu alguma regra de um dos _scripts_. Dentro do
arquivo de configuração você vai ver algo assim:

{% highlight YAML %}
PreCommit:
  Rubocop:
    on_warn: fail
{% endhighlight %}

Dessa forma não será permitido que o _commit_ ocorra até que o código que foi alterado esteja de acordo com o Rubocop.
Caso julgue necessário, é possível permitir que o _commit_ ocorra, mas serão mostrados avisos de que algumas linhas
modificadas não estão de acordo. Para isso basta modificar a última linha para `on_warn: warn`.

Além disso, serão sempre mostradas como um aviso apenas as outras linhas do arquivo modificado que não estão de acordo
com alguma checagem. Sempre vai ficar a seu critério decidir se vale a pena modificar algo ou não. Além disso,
arquivos não modificados não serão examinados.

Talvez você ache necessário manter o `fail` na sua configuração, mas realmente precisa fazer um _commit_ e alguma
checagem está te tirando do sério. Não tem problema, você pode utilizar o `SKIP` e fugir por hora. Basta informar
qual _hook_ quer pular dessa vez e pronto.

`SKIP=rubocop git commit`

## O resultado final de tudo isso

![_config.yml]({{ site.baseurl }}/images/posts/three/overcommit-fail.png)

Essa primeira imagem mostra meu _commit_ sendo bloqueado pelos motivos que estão em vermelho. Nas minhas configurações
eu não consigo terminar minha ação enquanto não corrigir esses "problemas" ou utilizar o `SKIP`.

![_config.yml]({{ site.baseurl }}/images/posts/three/overcommit-success.png)

Aqui é um outro exemplo em que o que eu modifiquei seguiu todos os padrões esperados e meu _commit_ foi um sucesso!
Além disso é possível perceber que o Rubocop gerou uma _warning_, ou seja, ele está alertando que existem alguns
"problemas" no arquivo que eu modifiquei, mas não necessariamente onde eu modifiquei. O _commit_ já foi feito, mas
eu posso modificar isso depois para se adequara aos padrões.

## Conclusão

Alguns projetos são gigantescos, passam de mão em mão e são modificados diariamente por dezenas de pessoas. Algumas
vezes é difícil manter certos padrões, seja por barreiras físicas ou de tempo mesmo. Se você está sofrendo com isso
ou apenas quer se manter "na linha", a _gem_ Overcommit é uma ótima solução.

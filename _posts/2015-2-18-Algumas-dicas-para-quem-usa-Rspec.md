---
layout: post
title: Algumas dicas para quem usa Rspec
---

[Rspec](http://rspec.info/) é um framework BDD (Behaviour-Driven Development, traduzindo, Desenvolvimento Guiado por Comportamento).
Suportado por uma vasta comunidade, é atualizado com frequência e sempre novas funcionalidades são adicionadas, permitindo
mais flexibilidade e uma cobertura maior e melhor de possíveis cenários.

Após alguns anos utilizando o Rspec em projetos que trabalhei, é notável que existem várias funcionalidades que algumas
pessoas desconhecem, mais pelo fato do framework ser realmente amplo. Aqui vão algumas dicas que acabei incorporando ao
longo desse tempo:


## 1 - Apenas um expect por bloco de teste

Tudo bem, você realmente pode ter uma boa razão para colocar 2 `expects` dentro do mesmo bloco de teste, eu entendo. Mas, na
maior parte das vezes, ou seu teste está errado ou você está sobrecarregando seus testes, o que irá se tornar mais difícil
de entender e manter.

{% highlight ruby %}
context '#method' do
  let(:object) { create(:object) }

  before { subject.do_something_with(object) }

  # Not Good!!
  it 'does what I want with the object' do
    expect(object).name to eq  'changed'
    expect(object).email to eq 'changed@email.com'
  end

 # Better :)
  it 'changes the object name' do
    expect(object).name to eq  'changed'
  end

  it 'changes the object email' do
    expect(object).email to eq 'changed@email.com'
  end
end
{% endhighlight %}

Você também pode se aproveitar da sintaxe de uma linha quando aplicável. Recentemente houve uma grande mudança na
sintaxe geral do Rspec, mas para utilizar o `it` de uma linha é possível usar a nova e a antiga ainda:

{% highlight ruby %}
# Antiga
it { should be_empty }

# Nova
it { is_expected.to be_empty }
{% endhighlight %}

## 2 - FactoryGirl e sua sintaxe

No exemplo anterior eu utilizei uma `factory` de `object`. Se você utiliza ou pensa em utilizar o Rspec para seus
testes, muito provavelmente já se deparou com FactoryGirl em algum momento, pois ela nada mais é do que uma
substituição para as tradicionais _fixtures_.

Minha dica aqui é: você não precisa utilizar FactoryGirl antes dos métodos da mesma. No exemplo anterior eu
chamei o método `create` diretamente, enquanto antigamente era comum algo do tipo: `FactoryGirl.create`.

Uma configuração simples e genérica que deve resolver a maioria dos casos para que essa sintaxe funcione é essa:

{% highlight ruby %}
RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
end
{% endhighlight %}

Caso precise de algo diferente, acesse a página deles [aqui](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md).

## 3 - Let, let! e porque utilizar

Em vez de instanciar variáveis por todo seu arquivo de teste o Rspec fornece uma ferramente importante: `let`.
No primeiro eu criei um `let` de `object` para os meus testes. O mais importante aqui é que a criação definitiva
de `object` só irá ocorrer sua chamada, ou seja, no momento em que passei o mesmo para o método do `subject`.
A partir deste momento, em todos os lugares do meu teste poderei chamar `object` sem problemas.

Mas as vezes você necessita de mais do que isso. Imagine por exemplo que meu `subject` não possui contato direto
um objeto. Eu não poderia chamar `object` em nenhum lugar, mas ainda assim necessitaria que ele estivesse criado.

{% highlight ruby %}
let(:object) { create(:object) }

# What now?
before { subject.do_something_with_object }
{% endhighlight %}

Nesses casos (e em muitos outros) podemos fazer uso do `let!`. Dessa forma, `object` será criado sempre antes
de qualquer exemplo.

## 4 - Be true, be_truthy e a saga dos _matchers_

Existem diversos _matchers_ que você pode utilizar, como citado no título. Além desses existem também os seus
opostos, `be false`, `be_falsey` e etc. Mas, eles são muito parecidos em nome, como vou saber qual utilizar? :fearful:

`be true` checa literalmente se o valor de algo é `true`, enquanto `be_truthy` checa se o valor de algo é verdadeiro.

Acho melhor vermos os exemplos:

{% highlight ruby %}
# Works!
it { expect(true).to be true }
it { expect(false).to be false }

it { expect(true).to be_truthy }
it { expect("String").to be_truthy }
it { expect(false).to be_falsey }
it { expect(nil).to be_falsey }

# Fails!
it { expect('String').to be true }
it { expect(nil).to be false}

it { expect(nil).to be_truthy }
it { expect("String").to be_falsey }
{% endhighlight %}

## 5 - A sintaxe mudou, e agora?

Você pode estar utilizando uma versão do Rspect antiga e ouviu que ninguém usa mais `should` e `expect` é
a nova moda. Você possui infinitas mil linhas de código e será um trabalho realmente difícil mudar tudo
para poder atualizar seu Rspec.

Já ouviu falar da gema [Transpec](https://github.com/yujinakayama/transpec)? Ela vai salvar seu(s) dia(s),
eu garanto. Você pode até customizar exatamente o que quer alterar nos seus testes.

Definitivamente uma boa pedida mesmo para aqueles que já estão com a mais nova versão do Rspec
rodando perfeitamente (no momento que escrevo, 3.2). Algumas sugestões podem ser válidas e você
sempre acaba aprendendo algo novo não é?


Espero que essas dicas sejam realmente úteis para você. Sabe de uma bem legal também? Deixe
um comentário!

---
layout: post
title: Como viajar no tempo com segurança, mas apenas nos seus testes!
---
Ao longo de sua jornada como desenvolvedor você irá precisar manipular o tempo, seja congelando o mesmo, avançando ou
voltando para alguma data no passado. Utilizamos isso especialmente quando desejamos testar algum comportamento do código
em determinada data, por exemplo, alguma mudança de status de um objeto em uma data anterior.

Como sempre, existem gems que podem fazer o serviço pra você. Acredito que a opção mais difundida seja a gem
[Timecop](https://github.com/travisjeffery/timecop). Em algum momento a gem [Delorean](https://github.com/bebanjo/delorean)
também foi bastante utilizada, mas parece que a mesma está abandonada nesse momento.

Fora isso, é válido argumentar que incluir uma dependência no sistema apenas para fazer um tipo de teste não é lá
a melhor coisa a ser feita. Por essas e outras, na [atualização para a versão 4.1 do Rails](http://guides.rubyonrails.org/4_1_release_notes.html#active-support-notable-changes)
foi incluído um helper para auxiliar nesses testes: [TimeHelpers](http://api.rubyonrails.org/classes/ActiveSupport/Testing/TimeHelpers.html).

Ok, eu sei. Essa versão já tem algum tempo. Porém, em muitos projetos que já trabalhei, as pessoas **não sabem** da existência
desse helper. Alguma vezes pode ser necessário manter o Timecop pois ele possui mais funcionalidades, mas na minha
experiência, as funcionalidades do helper são suficientes na maioria dos casos.

## Como eu começo?

Primeiramente você precisa estar utilizando a versão 4.1 do Rails no mínimo. Depois disso você precisa escolher se quer
tornar o helper disponível para todos os testes ou não. Caso queira utilizar em apenas um teste você terá algo mais
ou menos assim:

{% highlight Ruby %}
require 'spec_helper'

include ActiveSupport::Testing::TimeHelpers

describe Foo do
  before { travel_to(11.days.ago) }

  subject { true }

  it 'bars' do
    expect(subject).to be_truthy
  end
end
{% endhighlight %}

Caso esteja interessado em adicionar a funcionalidade para todos os seus testes, basta adicionar na configuração do
Rspec por exemplo:

{% highlight Ruby %}
RSpec.configure do |config|
  config.include ActiveSupport::Testing::TimeHelpers
end
{% endhighlight %}

## Funcionalidades

Com isso você terá 3 métodos para utilizar: `travel`, `travel_to` e `travel_back`. Os 2 primeiros são bem parecidos, pois tem
a função de travar o tempo. Ao utilizar `travel` você irá passar um determinado tempo para que seja somado ao dia atual, por
exemplo:

{% highlight Ruby %}
Time.current
# => Wed, 25 Jan 2017 14:49:27 BRST -02:00

travel(1.day)

Time.current
# => Thu, 26 Jan 2017 14:49:27 BRST -02:00
{% endhighlight %}

Já com `travel_to`, o tempo será congelado na data que for passada:

{% highlight Ruby %}
Time.current
# => Wed, 25 Jan 2017 14:56:37 BRST -02:00

travel_to(Time.new(2000, 03, 02, 20, 22, 14))

Time.current
# => Thu, 02 Mar 2000 14:56:37 BRST -02:00
{% endhighlight %}

Como você pode perceber, ambos os métodos atuam de forma semelhante,alterando o retorno de  alguns métodos para "congelar" o
tempo. Os métodos em questão são: `Time.now`, `Date.today`, and `DateTime.now`.

O último método, `travel_back`, é responsável por remover todas as alterações no retorno dos métodos anteriores,
ou seja, `Time.now` irá retornar o tempo no exato momento.

Por fim, `travel` e `travel_to` também aceitam um bloco:

{% highlight Ruby %}
Time.current
# => Wed, 25 Jan 2017 14:56:37 BRST -02:00

travel_to(Time.new(2000, 03, 02, 20, 22, 14)) do
  Time.current
  # => Thu, 02 Mar 2000 14:56:37 BRST -02:00
end

Time.current
# => Wed, 25 Jan 2017 14:56:37 BRST -02:00
{% endhighlight %}

Dessa forma, ao fim da execução do bloco o tempo voltará ao normal. Caso utilize o formato fora do bloco, dependendo do seu teste,
pode ser necessário utilizar o `travel_back` para normalizar os testes.


## Cuidado!

Como mencionado anteriormente, existem algumas diferenças entre o helper e a gem Timecop. Mas a principal de todas, e que pode
gerar algum tipo de erro caso não seja ponderada, é a diferença na hora de voltar o tempo ao normal.

Caso você esteja utilizando os helpers em sequência, por exemplo:

{% highlight Ruby %}
travel_to(Time.new(2000, 03, 02, 20, 22, 14)) do
  travel_to(Time.new(2010, 03, 02, 20, 22, 14))
  # code
  travel_back
  # code
end
{% endhighlight %}

O que irá ocorrer aqui é que, ao executar o método `travel_back`, o tempo irá voltar ao tempo correto, ignorando qualquer
outra alteração anterior, mesmo que dentro de um bloco. Podemos entender isso melhor olhando para a [implementação do método](https://github.com/rails/rails/blob/27eccc27cbe987be04bb97b49aff1d7fd118634c/activesupport/lib/active_support/testing/time_helpers.rb#L123):

{% highlight Ruby %}
def travel_back
  simple_stubs.unstub_all!
end
{% endhighlight %}

Todos os `stubs` gerados são removidos! Isso não ocorre com o Timecop por exemplo. O método para a mesma funcionalidade
na gem é o `return` e você pode ver a implementação de toda a classe que o possui
[aqui](https://github.com/travisjeffery/timecop/blob/master/lib/timecop/timecop.rb#L88).

Analisando o método especificamente:

{% highlight Ruby %}
def return(&block)
  if block_given?
    instance.send(:return, &block)
  else
    instance.send(:unmock!)
    nil
  end
end
{% endhighlight %}

É possível perceber que ele possui 2 tipos diferentes de ação, em que a primeira parte cuida para que o `return` volte o tempo não para o correto,
mas para o "congelamento" anterior. Dessa forma, em alguns casos específicos, pode ser necessário a utilização do Timecop.


## Conclusão

Caso você esteja fazendo uma limpa no sistema ou é daqueles que não gostam de adicionar dependências desnecessárias (eu \o), aqui está um bom exemplo
de funcionalidade nativa do Rails que você pode começar a usar agora mesmo. Apenas fique atento para não cair em uma armadilha pelas diferenças de ambos.
 Assim, vale a pena acessar a página do Timecop no GitHub e dar uma comparada com o helper.

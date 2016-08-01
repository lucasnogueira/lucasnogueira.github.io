---
layout: post
title: Dicas de Performance para Ruby
---

Faz um tempo que eu estava lendo o excelente livro [Ruby Performance Optimization - Why Ruby Is Slow, and How to Fix It](https://pragprog.com/book/adrpo/ruby-performance-optimization)
de [Alexander Dymo](http://www.alexdymo.com/) (vale a pena comprar!) e me deparei com conceitos interessantes que
acredito que muitas pessoas não conhecem ou pelo menos nunca gastaram um tempo tentando entender melhor.

Meu intuito nesse post é compartilhar um pouco do que aprendi e coloquei em prática no dia a dia, pois como bem sabemos
performance não pode ser algo deixado de lado enquanto estamos desenvolvendo.

## Garbage Collector - O Famoso GC

Lidar com memória nunca é fácil e para isso podemos contar com o GC. Responsável por detectar objetos alocados que não estão
mais sendo utilizados pelo programa, o GC toma de volta a memória desses objetos impedindo assim que erros catastróficos
ocorram e fornecendo mais recursos para a aplicação.

As versões mais antigas do Ruby (menores que a 2.1) sofriam bastanten pois não possuíam um GC otimizado e qualquer atuação do
mesmo acabava interferindo bastante na performance do sistema.

Vou utilizar a biblioteca `benchmark` para ilustrar como isso funciona e as implicações do GC em um código Ruby.
Como exemplo esse será o primeiro código a ser testado:

{% highlight Ruby %}
require "benchmark"

test = Array.new(1000) { Array.new(1000) { 'Foo Bar' * 100 } }

time = Benchmark.realtime do
  test.each do |row|
    row.each do |val|
      val = val + val * 2
    end
  end
end

puts time.round(2)
{% endhighlight %}

Vamos alternar as versões do Ruby para testar quanto tempo leva em cada versão para que esse código seja executado.
Utilizando [RVM](https://rvm.io/) podemos fazer isso dessa forma:

{% highlight SQL %}
rvm use X
ruby performance_test.rb
{% endhighlight %}

Onde `X` é a versão do Ruby que desejamos, como por exemplo, `2.3.1`.

Vamos aos resultados desse teste inicial (em segundos):

{% highlight Ruby %}
| 1.9.3-p551 | 2.0.0-p594 | 2.1.5 | 2.2.3 | 2.3.1 |
|------------|------------|-------|-------|-------|
|    12.1    |   15.58    |  4.35 |  3.24 |  3.21 |
{% endhighlight %}

Lembrando que rodar um exemplo apenas uma vez não é a maneira de mesurar perfeita,
visto que os resultados podem sofrer várias interferências. Porém, aqui os resultados são tão
discrepantes que servem para perceber que existe algo de diferente entre as versões.

Vamos rodar os mesmos exemplos novamente, porém agora iremos desabilitar o GC. Para isso basta inserir a linha de código
antes do bloco de `benchmark`:

{% highlight Ruby %}
#...
test = Array.new(1000) { Array.new(1000) { 'Foo Bar' * 100 } }

GC.disable
time = Benchmark.realtime do
#...
{% endhighlight %}

E os resultados:

{% highlight Ruby %}
| 1.9.3-p551 | 2.0.0-p594 | 2.1.5 | 2.2.3 | 2.3.1 |
|------------|------------|-------|-------|-------|
|    3.15    |    3.37    |  3.11 |  2.13 |  2.10 |
{% endhighlight %}

Logo vemos que agora todas as versões rodam esse mesmo código em tempos bem semelhantes, o que demonstra a evolução do GC
ao longo do tempo. As versões antigas gastavam muito tempo lidando com problemas de memória, o que foi se estabilizando
com o tempo.

Você deve estar pensando então que a solução é apenas desabilitar o GC. Eu recomendo não fazer isso pois provavelmente
problemas piores surgirão. A próxima parte desse post irá focar em pequenas dicas para evitar que o GC seja acionado
desnecessariamente, fazendo assim com que a aplicação ganhe em performance.

## As Dicas de Performance

### Modificar Strings Localmente

A dica aqui é a seguinte: na maioria dos casos utilizar os métodos com `bang(!)` vai te salvar. Isso ocorre pois
não é necessário alocar memória copiando a `String` que será modificada, e além de economizarmos memória,
evitamos de chamar o GC desnecessáriamente.

Vamos modificar um pouco nosso código para medir quantas vezes o GC é chamado na execução do código. Além disso, vamos rodar
utilizando apenas a versão `2.3.1`. Aqui os resultados para uma modificação sem utilizar o método `bang`:

{% highlight Ruby %}
require "benchmark"

text = 'Foo Bar' * 1000 * 1000 * 1000

gc_on_start = GC.stat

time = Benchmark.realtime do
  text = text.reverse
end

gc_on_end = GC.stat

puts time.round(2)
puts gc_on_end[:count] - gc_on_start[:count]
{% endhighlight %}

Nosso resultado é:

{% highlight Ruby %}
36.57
1
{% endhighlight %}

Então temos uma chamada para o GC e 36 segundos gastos executando esse código. Vamos agora substituir o código:

{% highlight Ruby %}
time = Benchmark.realtime do
  text.reverse!
end
{% endhighlight %}

E o resultado é:

{% highlight Ruby %}
13.19
0
{% endhighlight %}

Um ótimo tempo e nenhuma chamada para o GC. Bem melhor não?


### Modificar Arrays e Hashes Localmente

A mesma dica se aplica para `Arrays` e `Hashes`, simplesmente pelo mesmo motivo. Ambos possuem diversos métodos
utilizando `bang`, como por exemplo `map!` e `select!`.

{% highlight Ruby %}
require "benchmark"

array = Array.new(1000) { 'Foo Bar' * 1000 * 500  }

gc_on_start = GC.stat

time = Benchmark.realtime do
  array.map { |text| text.reverse }
end

gc_on_end = GC.stat

puts time.round(2)
puts gc_on_end[:count] - gc_on_start[:count]
{% endhighlight %}

Executando temos esses resultados:

{% highlight Ruby %}
4.84
100
{% endhighlight %}

Note que temos 100(!) chamadas para o GC. Definitivamente péssimo não é mesmo? Vamos substituir `map` por `map!` e
`reverse` por `reverse!`. Resultados:

{% highlight Ruby %}
2.11
0
{% endhighlight %}

Obviamente 0 chamada é bem melhor que 100. Além disso, o tempo de execução foi bem menor. Com isso é possível concluir
que, sempre que possível, os métodos com `bang` são o caminho a seguir.

### Alguns Métodos para Ficar de Olho

Alguns métodos podem ser especialmente danosos, principalmente dentro de `Iterators`. Vamos tomar como exemplo o método
`is_a?`:

{% highlight Ruby %}
require "benchmark"

array = Array.new(500_000) { 'Foo Bar' }

time = Benchmark.realtime do
  array.each do |text|
    text.is_a?(String)
  end
end

puts time.round(2)
{% endhighlight %}

O resultado aqui é `0.04` ms. Você pode até pensar que isso não é nada demais, mas imagine uma aplicação Rails
chamando esse tipo de comparação em um número maior que esse por request feita? As coisas podem ficar um pouco feias.

Outros métodos que sofrem com esse mesmo "problema" são `class` e `kind_of?`.
No geral, evite deixar eles largados dentro de algum `Iterator` e tudo ficará bem.

## Conclusão

Enquanto estamos desenvolvendo é difícil manter o foco 100% em performance, até porque alguns problemas podem ser contornados
utilizando [cache](https://en.wikipedia.org/wiki/Cache_(computing)) ou escalando a aplicação.
As vezes também a solução performática pode não ser a mais bonita.

Porém, precisamos entender e pensar nas consequências de nossas escolhas nesse assunto. É importante entender como as coisas
funcionam por baixo dos panos e ter conhecimento de que ferramentas podemos utilizar para nos auxiliar nessa jornada.


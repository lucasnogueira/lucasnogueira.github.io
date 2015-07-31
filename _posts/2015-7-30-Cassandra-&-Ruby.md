---
layout: post
title: Cassandra & Ruby
---

Recentemente tive a oportunidade de mexer um pouco com o banco de dados [Apache Cassandra](http://cassandra.apache.org/).
Para aqueles que não conhecem, o Cassandra é um banco de dados cuja principal característica é a sua
escalabilidade. Sua replicação e particionamento em diferentes nós é algo que o torna muito seguro, capaz de se recuperar
de falhas sem muitos problemas. Para encerrar, basta dizer que ele é utilizado por empresas como eBay, GitHub, Netflix e
Reddit.

As premissas apresentadas até aqui foram o motivo da experiência. Nesse post tentarei resumir minha aventura, contando um
pouco de como funciona o Cassandra, algumas configurações e a conexão com o drive Ruby do mesmo.

## Mais sobre Cassandra

_Obs: Não irei entrar em detalhes sobre a instalação do Cassandra em si, mas na próxima etapa iremos acabar instalando por tabela.
Caso tenha mais interesse no assunto recomendo acessar esse [link](http://docs.datastax.com/en/cassandra/2.0/cassandra/install/installDeb_t.html).
e para executar comandos CQL(Cassandra Query Language) direto no terminal é necessário executar o [CQLSH](http://docs.datastax.com/en/cql/3.0/cql/cql_reference/cqlsh.html)._

A estrutura principal são os _clusters_, que são conjuntos de nós. Cada nó é responsável por armazenar alguns dados e nada
mais é do que uma instância separada do Cassandra rodando em algum lugar. Mais para frente explicarei como simular diferentes
nós localmente da maneira mais fácil, mas em um cenário ideal você irá querer ter máquinas diferentes para cada nó.

A divisão de dados entre esses nós fica nas suas mãos. Por exemplo, você pode optar por não haver nenhum tipo de replicação dos dados
entre nós (não faça isso) ou de replicar informações entre 2 nós diferentes. O segredo do Cassandra é que ele não irá separar
2 nós para cuidar desses dados repetidos, mas irá dividir da melhor forma entre os nós já existentes.

Ao acessar o nosso _cluster_ pela primeira vez é nosso dever criar um _keyspace_ para armazenar nossas tabelas de dados.
É nessa etapa que a escolha da replicação ocorre:

{% highlight SQL %}
CREATE KEYSPACE Teste WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 2 };
{% endhighlight %}

A primeira opção que passamos aqui irá determinar como utilizaremos o Cassandra, sendo a estratégia escolhida a ideal para testes.
Caso necessite de algo mais robusto, que aguente múltiplos _data centers_, é necessário utilizar a opção  "NetworkToplogyStrategy".

A segunda opção diz respeito ao nível de recplicação, onde 1 significa dizer que não estaremos replicando dado algum. É necessário
passar um número inferior ou igual ao número de nós no _cluster_. Aqui, ao escolher 2, estaremos replicando dados em mais um nó
além daquele em que os dados já estão sendo inseridos normalmente.

Com isso podemos utilizar o nosso _keyspace_ e criar nossa primeira tabela:

{% highlight SQL %}
USE Teste;

CREATE TABLE users (
    user_name varchar PRIMARY KEY,
    password varchar,
    gender varchar,
    birth_year bigint
);
{% endhighlight %}

### Se ogranizando com as _Keys_

No último exemplo declaramos que `user_name` é nossa `PRIMARY KEY` (chave primária). Mas o que isso realmente significa?

Em primeiro lugar significa dizer que todos os _users_ que criarmos deverão ter um `user_name` único. E, além disso,
os dados serão organizados baseados nesse campo. Dessa forma, para o Cassandra, uma chave primária não composta é também uma
`PARTITION KEY` (chave de partição).

Com isso em mente é seguro afirmar que o Cassandra distribui os dados entre nós dos mesmo _cluster_ baseado na chave de partição
escolhida. Porém, uma chave de partição pode também ser uma chave composta:

{% highlight SQL %}
create table keys (
    part_one text,
    part_two int,
    anything text,
    PRIMARY KEY(part_one, part_two)
);
{% endhighlight %}

Nesse exemplo a chave de partição é a primeira parte da nossa chave primária, ou seja, `part_one`. A segunda parte da chave
primária fica com o trabalho de organizar os dados dentro da partição, sendo chamada de `CLUSTERING KEY` (chave de cluster).

Assim encerro as explicações sobre os principais motivos que me levaram até o Cassandra. O resto do post será focado na melhor
forma de utilizar diversos nós localmente para testes e aprendizado, além da utilização do Cassandra propriamente dito por meio
do seu drive de Ruby.

## Quero muitos nós e quero todos localmente

É possível manter diversas instalações do Cassandra rodando separadamente e simular diversos nós localmente, mas isso não seria
muito legal de manter. Pensando na sanidade da população um script foi criado para facilitar as coisas: [CCM](https://github.com/pcmanus/ccm).

Para saber mais sobre as dependências e as diversas formas de instalação eu sugiro acessar o link informado anteriormente e escolher
a forma que melhor lhe agradar.

Com tudo instalado vamos para um exemplo rápido no terminal:

{% highlight SQL %}
ccm create teste -v 2.0.5 -n 3
{% endhighlight %}

Aqui estamos criando o _cluster_ "teste" e instalando autmanticamente a versão 2.0.5 do Cassandra para isso. Você pode alterar a versão
como preferir nessa parte.

A última opção é responsável pelo número de nós que serão criados no _cluster_, nesse caso, 3.

Um detalhe para quem estiver utilizando Mac: é necessário criar uma nova interface de rede para cada nó além do primeiro:

{% highlight python %}
sudo ifconfig lo0 alias 127.0.0.2
sudo ifconfig lo0 alias 127.0.0.3
{% endhighlight %}

Nesse exemplo estamos levando em conta que os endereços fornecidos estejam disponíveis. Caso você tenha um erro parecido com esse:

{% highlight YAML %}
Inet address 127.0.0.1:9042 is not available: [Errno 48] Address already in use
{% endhighlight %}

então não esqueça de configurar as novas interfaces para os nós ok?

Caso tudo esteja correto basta iniciar o _cluster_ para começar:

{% highlight python %}
ccm start
{% endhighlight %}

Agora você pode se conectar via cqlsh fornecendo o nome do nó que quer se conectar (node1, node2 etc.) e executar comandos
livremente.

## Driver Ruby

Para começar vamos instalar a gem:

{% highlight Ruby %}
gem install cassandra-driver
{% endhighlight %}

Lembrando que caso esteja com um projeto iniciado, basta incluir no seu Gemfile:

{% highlight Ruby %}
gem 'cassandra-driver'
{% endhighlight %}

Caso tenha configurado tudo corretamente e o Cassandra esteja de pé você terá um _cluster_ de 3 nós disponível para seus testes.
Para acessar esse _cluster_ é simples:

{% highlight Ruby %}
require 'cassandra'

cluster = Cassandra.cluster
{% endhighlight %}

Você também pode passar algumas opções, como autenticação e escolha de que nós em específico quer usar de um determinado _cluster_:

{% highlight Ruby %}
cluster = Cassandra.cluster(
    username: username,
    password: password,
    hosts: ['10.0.1.1', '10.0.1.2', '10.0.1.3']
)
{% endhighlight %}

Com o _cluster_ armazenado em uma váriavel podemos fazer todas as interações com nosso banco de dados. Vamos agora acessar o _keyspace_
padrão e criar o nosso próprio:

{% highlight Ruby %}
keyspace = 'system'

session = cluster.connect(keyspace)

ks = <<-KEYSPACE
  CREATE KEYSPACE Teste
  WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 2
  }
KEYSPACE

session.execute(ks)
session.execute('USE Teste')
{% endhighlight %}

Utilizamos o _keyspace_ "system" para se conectar no _cluster_, armazenando o objeto resultado dessa conexão na variável `session`.
Feito isso criamos uma variável para armamzenar as definições do nosso novo _keyspace_.

Através do método `execute` conseguimos fazer todo o tipo de comando, sejam simples _queries_ ou criações de índices. Nesse
exemplo foi executado a criação do nosso _keyspace_, que estava armazeado em `ks`.

Por fim executamos novamente um comando para utilizar esse novo _keyspace_ que acabamos de criar. Apartir de agora qualquer tabela que
criarmos ficará disponível apenas em "Teste", e é isso que vamos fazer agora:

{% highlight Ruby %}
table = <<-TABLE
CREATE TABLE foo (
    id INT,
    anything VARCHAR,
    PRIMARY KEY (id)
    )
TABLE

session.execute(table)
{% endhighlight %}

### _Prepared Statements_

Como você ja deve ter imaginado, executar uma consulta ou inserção de dados ocorre por meio do método `execute`:

{% highlight Ruby %}
session.execute('SELECT * FROM foo')
{% endhighlight %}

Uma coisa legal que podemos fazer é utilizar os _Prepared Statements_, que nada mais são do que variáveis contendo comandos
definidos previamente que serão utilizados mais do que uma vez.

{% highlight Ruby %}
insert = session.prepare('INSERT INTO foo (id, anything) VALUES (?, ?)')

get_all = session.prepare('SELECT * FROM foo')

session.execute(insert, arguments: [999, 'potatoes'])

session.execute(get_all)
{% endhighlight %}

Essa é uma ótima forma de reutilizar código e deixar tudo muito mais legível, mas use com moderação.

### Paralelismo

É possível executar diversos comandos em paralelo de forma muito fácil. Vimos anteriormente o comando `execute`, mas aqui utilizaremos
o `execute_async`, responsável pela execução de comandos assincronamente.

{% highlight Ruby %}
foo = [
  [1, 'potatoes'],
  [2, 'apples'],
  [3, 'tomatoes']
]

futures = foo.map do |(id, anything)|
  session.execute_async(insert, arguments: [age, username])
end
{% endhighlight %}

Nesse exemplo `futures` irá receber objetos da classe `Future`. Não irei abordar tudo que é possível fazer com ela, mas caso essa seja
sua necessidade recomendo essa [leitura](http://datastax.github.io/ruby-driver/api/future/).

### Paginação

Algumas vezes uma consulta pode retornar algo gigantesco que não estamos preparados para lidar. Com isso em mente, podemos utilizar
a opção `page_size` em nossas consultas para resolver o problema:

{% highlight Ruby %}
foos = session.execute("SELECT * FROM foo WHERE anything = 'lol'", page_size: 100)
{% endhighlight %}

Deixando explícito o tamanho da página que estamos esperando tudo fica mais tranquilo. Para mudar de página basta chamar o método `next_page`.

### Compressão de Dados

A última dica fica por conta da otimização em requisições muito grandes. O Cassandra suporta dois algoritmos de compressão: Snappy e LZ4.
Segundo a doc oficial é recomendável a utilização do segundo. Para utilizar juntamente com o driver é necessário instalar a gem respectiva
separadamente.

A configuração de compressão de dados ocorre diretamente no _cluster_ em que estamos utilizando:
{% highlight Ruby %}
cluster = Cassandra.cluster(compression: :lz4)
{% endhighlight %}


## Conclusões

Minha intenção com esse post foi abordar de uma forma direta  minhas experiências com esssa tecnologia, tentando facilitar no formato de um roteiro
todo o caminho que percorri. Espero que assim mais pessoas mostrem interesse em aprender e testar esse excelente banco de dados.

Lembrando que existem diversas documentações oficiais em inglês na internet, sendo que eu mesmo me basiei em algumas. Caso queira
saber mais sobre o driver de Ruby para o Cassandra clique [aqui](http://datastax.github.io/ruby-driver/features/).

Por hoje encerramos nossa programação. Volte sempre :stuck_out_tongue_winking_eye:

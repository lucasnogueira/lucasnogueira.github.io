---
layout: post
title: PostgreSQL e Rails
---
Hoje nós desenvolvedores temos ótimas opções quando o assunto é banco de dados open source, e isso é realmente incrível.
Sejam eles relacionais ou não, o que importa é que estamos tão cobertos hoje que a parte difícil fica entre
escolher um ou outro, muito porque alterar o banco de dados no futuro provavelmente vai ser uma tarefa
muito custosa.

Eu tive a chance de trabalhar com muitos bancos de dados, porém devo dizer que o [PostgreSQL](https://www.postgresql.org/)
(vamos chamar de Postgres apenas ok?) é uma escolha sempre forte, capaz de oferecer ferramentas que outros não fornecem
(ou fornecem mas não tão bem).

Esse post irá mostrar algumas funcionalidades úteis que o Postgres oferece, além de como utilizar essas funções juntamente
com o Rails para aprimorar nossas aplicações. Todos os tópicos possuem assunto suficiente para um post dedicado, então
farei o possível para resumir cada um sem perder qualidade. Vamos começar!

### Campos Json(b)

Na versão 9.2 do Postgres foi implementado o suporte nativo para JSON. Depois de um tempo a versão 9.4 chegou e trouxe o
poder do JSONB, ou "Binary JSON". Colocando ambos lado a lado é possível dizer que, enquanto JSONB é mais lento em termos
de velocidade de escrita, o mesmo possui suporte para índices, o que fornece ampla vantagem em termos de busca.

Foi a versão [4.2 do Rails](http://guides.rubyonrails.org/4_2_release_notes.html#active-record-notable-changes)
a responsável por adicionar ao adaptador do Postgres o suporte tão esperado para JSONB. Vamos ver agora como é fácil
colocar ele em uso:

{% highlight Ruby %}
create_table :flags do |t|
  t.string :name, null: false
  t.jsonb :colors
end
{% endhighlight %}

Algo que é preciso ter em mente é que qualquer campo JSON ou JSONB será representado pelo Rails como um Hash. Veja:

{% highlight Ruby %}
flag = Flag.new(
        name: 'Brazil',
        colors: {
          blue: '3200FE',
          green: '007600',
          yellow: 'FEFE00',
          white: 'FFFFFF'
        })

flag.colors
#=>{"blue"=>"3200FE", "green"=>"007600", "yellow"=>"FEFE00", "white"=>"FFFFFF"}

flag.colors['white']
#=>FFFFFF
{% endhighlight %}

Buscas mais complexas podem ser feitas utilizando os cambos JSONB, apenas não esqueça de criar os índices apropriados.

Por exemplo, utilizando o nosso modelo de exemplo, `Flag` (bandeira),  podemos pesquisar por bandeiras que possuam um
tipo específico de cor verde:

{% highlight Ruby %}
Flag.where('colors @> ?', {green: '007600'}.to_json)
{% endhighlight %}

Ou então bandeiras que possuam as cores azul e amarelo:

{% highlight Ruby %}
Flag.where('colors ?& array[:keys]', keys: ['blue', 'yellow'])
{% endhighlight %}

### Constraints

Aqui iremos utilizar uma funcionalidade chamada "check constraint" para auxiliar nas validações nativas do Rails.
Alguns métodos, como `update_attribute`, acabam passando direto pelas validações, e isso pode causar problemas se
não estivermos atentos. Uma "constraint" é perfeita para essa situação, pois o erro vai ser bloqueado antes de chegar até
o banco de dados.

É fácil criar uma "constraint" dentro de uma `migration`. Vamos utilizar o exemplo anterior mantendo vivo nosso modelo de
bandeira. Imagine que, por algum motivo aleatório, agora não queremos que bandeiras com o nome começando com vogais sejam
aceitas pelo sistema. Então, países como Angola, Ucrânia e etc. não podem passar pelas validações.

Para isso criamos uma `migration`:

{% highlight Ruby %}
class AddNameConstraintToFlags < ActiveRecord::Migration
  def up
    execute <<-SQl
      ALTER TABLE flags
      ADD CONSTRAINT no_flags_starting_with_vowels
      CHECK ( namel ~* '\A(?=[^aeiou])(?=[a-z])' )
    SQL
  end

  def down
    execute << SQL
      ALTER TABLE flags
      DROP CONSTRAINT no_flags_starting_with_vowels
    SQL
  end
end
{% endhighlight %}

Aqui estamos utilizando expressões regulares para realizar nossos desejos. Podemos dizer agora que
nossas bandeiras estão respeitando as regras. De qualquer forma, isso nos leva até um problema.
O Rails não sabe como lidar direito com "check constraints", e por causa disso nossas mudanças não
vão refletir dentro do `db/schema.rb`. É preciso dizer para o Rails utilizar SQL no lugar de Ruby
quando armazenar o esquema. Isso é bem simples, basta adicionar uma linha de código dentro do arquivo
`config/application.rb`:

`config.active_record.schema_format = :sql`

Para finalizar, basta deleter o esquema atual e recriar utilizando `bundle exec rake db:migrate`.

### Expressões Regulares

Nós já utilizamos as expressões regulares antes, e elas podem ser muito poderosas. É realmente fácil utilizar uma regex dentro
de uma busca utilizando Rails e Postgres juntos. Então, o que você precisa saber aqui é:

 `~*` é o operador para "case insensitive" e `~` o operador para "case sensitive".

 `!` é utilizado para negação, então `!~` e `!~*`.

 Para Rails 4 e versões superiores, a forma genérica de se construir uma busca com regex é
 `Flag.where("name ~* ?", "regex")`.

 Isso irá permitir você criar buscas complexas facilmente utilizando todo o poder que o Postgres fornece!

## Conclusão
Postgres é uma escolha sólida de banco de dados open source quando pareado com Ruby on Rails. Nós possuímos ótimas
escolhas hoje em dia, mas com o crescente suporte e as novas funcionalidades é difícil dizer não para ele na maior
parte das vezes.

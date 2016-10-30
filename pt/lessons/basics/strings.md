---
layout: page
title: Strings
category: basics
order: 14
lang: pt
---

Strings, Char Lists, Graphemes e Codepoints.

{% include toc.html %}

## Strings

Strings em Elixir são nada mais do que uma seqüência de bytes. Vejamos um exemplo:

```elixir
iex> string = <<104,101,108,108,111>>
"hello"
```

>NOTE: Usando a sintaxe << >> nós estamos dizendo para o compilador que os elementos de dentro desses simbolos são bytes.

## Char Lists

Internamente, strings em Elixir são representada com uma seqüência de butes ao invés de um array de caracteres. Elixir também tem um tipo char list (character list). Strings em Elixir são fechadas com aspas duplas, enquanto listas de char são fechadas com aspas simples.

Qual é a diferença? Cada valor de uma lista de char é um valor de caracter ASCII. Vamos mostrar:

```elixir
iex> char_list = 'hello'
'hello'

iex> [hd|tl] = char_list
'hello'

iex> {hd, tl}
{104, 'ello'}

iex> Enum.reduce(char_list, "", fn char, acc -> acc <> to_string(char) <> "," end)
"104,101,108,108,111,"
```

Ao programar em Elixir, geralmente usamos Strings, não listas de caracteres. O suporte das listas de caracteres está incluído principalmente porque ele é necessário para alguns módulos de Erlang.

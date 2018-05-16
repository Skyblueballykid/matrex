# Matrex

[![Build Status](https://travis-ci.org/versilov/matrex.svg?branch=master)](https://travis-ci.org/versilov/matrex)
[![Coverage Status](https://coveralls.io/repos/github/versilov/matrex/badge.svg?branch=master)](https://coveralls.io/github/versilov/matrex?branch=master)
[![Inline docs](http://inch-ci.org/github/versilov/matrex.svg?branch=master)](http://inch-ci.org/github/versilov/matrex)
[![hex.pm version](https://img.shields.io/hexpm/v/matrex.svg)](https://hex.pm/packages/matrex)

Fast matrix manipulation library for Elixir implemented in C native code with highly optimized CBLAS sgemm() used for matrix multiplication.

Based on matrix code from https://github.com/sdwolfz/exlearn

## Benchmark

2015 MacBook Pro, 2.2 GHz Core i7, 16 GB RAM

```
benchmark name                iterations   average time
50x50 matrices dot product           500000   6.54 µs/op
transpose a 100x100 matrix           100000   12.40 µs/op
100x100 matrices dot product          50000   37.82 µs/op
transpose a 200x200 matrix            50000   66.05 µs/op
200x200 matrices dot product          10000   126.95 µs/op
transpose a 400x400 matrix            10000   175.82 µs/op
400x400 matrices dot product           5000   686.64 µs/op


```

## Visualization

Matrex implements `Inspect` protocol and looks nice in your console:

![Inspect Matrex](https://raw.githubusercontent.com/versilov/matrex/master/docs/matrex_inspect.png)


## Installation

The package can be installed
by adding `matrex` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:matrex, "~> 0.4"}
  ]
end
```
On MacOS everything works out of the box, thanks to Accelerate framework.

On Ubuntu you need to install scientific libraries for this package to compile:

```bash
> sudo apt-get install build-essential erlang-dev libatlas-base-dev
```
## Matrices

## Access behaviour

Access behaviour is partly implemented for Matrex, so you can do:

    iex> m = Matrex.magic(3)
    #Matrex[3×3]
    ┌                         ┐
    │     8.0     1.0     6.0 │
    │     3.0     5.0     7.0 │
    │     4.0     9.0     2.0 │
    └                         ┘
    iex> m[2][3]
    7.0

Or even:

    iex> m[1..2]
    #Matrex[2×3]
    ┌                         ┐
    │     8.0     1.0     6.0 │
    │     3.0     5.0     7.0 │
    └                         ┘


There are also several shortcuts for getting dimensions of matrix:

    iex> m[:rows]
    3

    iex> m[:size]
    {3, 3}

calculating maximum value of the whole matrix:

    iex> m[:max]
    9.0

or just one of it's rows:

    iex> m[2][:max]
    7.0

calculating one-based index of the maximum element for the whole matrix:

    iex> m[:argmax]
    8

and a row:

    iex> m[2][:argmax]
    3

## Math operators overloading

`Matrex.Operators` module redefines `Kernel` math operators (+, -, \*, / <|>) and
defines some convenience functions, so you can write calculations code in more natural way.

It should be used with great caution. We suggest using it only inside specific functions
and only for increased readability, because using `Matrex` module functions, especially
ones which do two or more operations at one call, are 2-3 times faster.

### Usage example

    def lr_cost_fun_ops(%Matrex{} = theta, {%Matrex{} = x, %Matrex{} = y, lambda} = _params)
        when is_number(lambda) do
      # Turn off original operators
      import Kernel, except: [-: 1, +: 2, -: 2, *: 2, /: 2, <|>: 2]
      import Matrex.Operators

      m = y[:rows]

      h = sigmoid(x * theta)
      l = ones(size(theta)) |> set(1, 1, 0.0)

      j = (-t(y) * log(h) - t(1 - y) * log(1 - h) + lambda / 2 * t(l) * pow2(theta)) / m

      grad = (t(x) * (h - y) + (theta <|> l) * lambda) / m

      {scalar(j), grad}
    end

The same function, coded with module methods calls (2.5 times faster):

    def lr_cost_fun(%Matrex{} = theta, {%Matrex{} = x, %Matrex{} = y, lambda} = _params)
        when is_number(lambda) do
      m = y[:rows]

      h = Matrex.dot_and_apply(x, theta, :sigmoid)
      l = Matrex.ones(theta[:rows], theta[:cols]) |> Matrex.set(1, 1, 0)

      regularization =
        Matrex.dot_tn(l, Matrex.square(theta))
        |> Matrex.scalar()
        |> Kernel.*(lambda / (2 * m))

      j =
        y
        |> Matrex.dot_tn(Matrex.apply(h, :log), -1)
        |> Matrex.substract(
          Matrex.dot_tn(
            Matrex.substract(1, y),
            Matrex.apply(Matrex.substract(1, h), :log)
          )
        )
        |> Matrex.scalar()
        |> (fn
              NaN -> NaN
              x -> x / m + regularization
            end).()

      grad =
        x
        |> Matrex.dot_tn(Matrex.substract(h, y))
        |> Matrex.add(Matrex.multiply(theta, l), 1.0, lambda)
        |> Matrex.divide(m)

      {j, grad}
    end



## Enumerable protocol

Matrex implements `Enumerable`, so, all kinds of `Enum` functions are applicable:

    iex> Enum.member?(m, 2.0)
    true

    iex> Enum.count(m)
    9

    iex> Enum.sum(m)
    45

For functions, that exist both in `Enum` and in `Matrex` it's preferred to use Matrex
version, beacuse it's usually much, much faster. I.e., for 1 000 x 1 000 matrix `Matrex.sum/1`
and `Matrex.to_list/1` are 438 and 41 times faster, respectively, than their `Enum` counterparts.

## Saving and loading matrix

You can save/load matrix with native binary file format (extra fast)
and CSV (slow, especially on large matrices).

Matrex CSV format is compatible with GNU Octave CSV output,
so you can use it to exchange data between two systems.

## NaN and Infinity

Float special values, like `NaN` and `Inf` live well inside matrices,
can be loaded from and saved to files.
But when getting them into Elixir they are transferred to `NaN`,`Inf` and `NegInf` atoms,
because BEAM does not accept special values as valid floats.

    iex> m = Matrex.eye(3)
    #Matrex[3×3]
    ┌                         ┐
    │     1.0     0.0     0.0 │
    │     0.0     1.0     0.0 │
    │     0.0     0.0     1.0 │
    └                         ┘

    iex> n = Matrex.divide(m, Matrex.zeros(3))
    #Matrex[3×3]
    ┌                         ┐
    │     ∞      NaN     NaN  │
    │    NaN      ∞      NaN  │
    │    NaN     NaN      ∞   │
    └                         ┘

    iex> n[1][1]
    Inf

    iex> n[1][2]
    NaN

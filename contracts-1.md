<!-- livebook:{"persist_outputs":true} -->

# Contract Programming an Elixir approach - Part 1

## Brief Intro

This is a series that explores the concepts found in Contract Programming and adapts them to 
the Elixir language; erlang and BEAM languages in general are sorrounded by philosophies like
"fail fast", "defensive programming" and "offensive programming" and I find that contract 
programming can be a nice addition to the language.

You will find a lot of unconventional uses of Elixir probably things you would not try
in a production environment, however through the series I will share some well-established
Elixir libraries that already do contracts very well.

## Programming by contract?

It is an approach to program verification that relies on the succesful execution of statements; 
not much different to what we do with [ExUnit](https://hexdocs.pm/ex_unit/ExUnit.html) when 
testing:

```elixir
defmodule Program do
  def sum_all(numbers), do: Enum.sum(numbers)
end

ExUnit.start(autorun: false)

defmodule ProgramTest do
  use ExUnit.Case

  test "Result is the sum of all numbers" do
    assert Program.sum_all([-10, -5, 0, 5, 10]) == 0
  end

  test "Should be able to process ranges" do
    assert Program.sum_all(0..10) == 55
  end

  test "Passed in parameter should only be a list or range" do
    assert_raise Protocol.UndefinedError,
                 ~s(protocol Enumerable not implemented for "1 2 3" of type BitString),
                 fn -> Program.sum_all("1 2 3") end
  end

  test "All parameters must be of numeric value" do
    assert_raise ArithmeticError, ~s(bad argument in arithmetic expression), fn ->
      Program.sum_all([["1", "2", "3"]])
    end
  end
end

ExUnit.run()
```

```output
....

Finished in 0.00 seconds (0.00s async, 0.00s sync)
4 tests, 0 failures
```

In the example above we're taking `Program.sum_all/1` and verifying its behaviour by giving
it inputs and matching with the outputs; in a sense, our function becomes a component that 
we can only inspect from the outside.

<!-- livebook:{"break_markdown":true} -->

Contract programming differs in that our assertions go embedded inside the components of
our system, lets try to use the `assert` keyword within the program:

```elixir
defmodule VerifiedProgram do
  use ExUnit.Case

  def sum_all(numbers) do
    assert is_list(numbers) || is_struct(numbers, Range),
           "Passed in parameter must be a list or range"

    result =
      Enum.reduce(numbers, 0, fn number, accumulator ->
        assert is_number(number), "Element #{inspect(number)} is not a number"
        accumulator + number
      end)

    assert is_number(result), "Result didn't return a number got #{inspect(result)}"
    result
  end
end
```

Our solution got a bit more verbose but hopefully we're now able to extract to the point
errors just by evaluation:

```elixir
VerifiedProgram.sum_all("1 2 3")
```
```output
** (ExUnit.AssertionError) 

Passed in parameter must be a list or range
```

```elixir
VerifiedProgram.sum_all(["1", "2", "3"])
```
```output
** (ExUnit.AssertionError) 

Element "1" is not a number
```

This style of verification shifts the focus a bit, rather than only checking input/output, we're
now explicitly limiting the function reach; when something unexpected happens we halt the program
and entirely try to give a reasonable error.

This is how the concept of "contracts" work in a very basic sense.

<!-- livebook:{"break_markdown":true} -->

### What about tests?

Having contracts in our codebase doesn't mean that we stop doing
tests, we can still write them and maybe even reduce the scope
of our checks:

```elixir
defmodule VerifiedProgramTest do
  use ExUnit.Case

  test "Result is the sum of all numbers" do
    assert VerifiedProgram.sum_all(0..10) == 55
    assert VerifiedProgram.sum_all([-10, -5, 0, 5, 10]) == 0
    assert VerifiedProgram.sum_all([1.11, 2.22, 3.33]) == 6.66
  end
end

ExUnit.run()
```

```output
.

Finished in 0.00 seconds (0.00s async, 0.00s sync)
1 test, 0 failures
```

And by just using our functions in runtime or test-time we can re-align the expectations of our
system components if requirements change:

```elixir
# Now we expect this to work
VerifiedProgram.sum_all("1 2 3 4")
```

```output
** (ExUnit.AssertionError) 

Passed in parameter must be a list or range
```

And make the changes required for it to happen, in this case we need to expand our domain
to also include stringified numbers separated by a space.

<!-- livebook:{"break_markdown":true} -->

### Should we add `use ExUnit` everywhere then?

<!-- livebook:{"break_markdown":true} -->

As we did in the examples above, there's nothing really stopping us from trying, the `assert` 
keyword is a creative way to verify our system components; I feel however that the failures 
are designed in such a way as to be used in a test environment, not necessarilly at runtime.

> **From the [docs](https://hexdocs.pm/ex_unit/ExUnit.Assertions.html):**
> "_In general, a developer will want to use the general assert macro in tests. 
> This macro introspects your code and provides good reporting whenever there is a failure._"

Thankfully for us in Elixir we have a more primitive mechanism in which to assert data in 
an effective way, [pattern matching](https://hexdocs.pm/elixir/patterns-and-guards.html#content);
I would like to explore this a bit more in the second installment of this contract series.

## Conclusions

* Contract programming is a technique to program verification that can be applied in Elixir.
* Similar to testing, but we're not limited to only verify at test time.
* We embedd assertions within our code to check for failures.
* Although not endorsed, we may take advantage of `ExUnit` to do contracts in Elixir.
* Other mechanisms native to erlang and elixir may be used to achieve similar results.

<!-- livebook:{"break_markdown":true} -->

### More info on contracts

* Wikipedia's entry on [Assertion](https://en.wikipedia.org/wiki/Assertion_(software_development).
* Wikipedia's entry on [Design by Contract](https://en.wikipedia.org/wiki/Design_by_contract).
* The elixir [Ex Contract](https://hexdocs.pm/ex_contract/readme.html) and [Oath](https://hexdocs.pm/oath/Oath.html) libraries expose a more conventional use of contracts.

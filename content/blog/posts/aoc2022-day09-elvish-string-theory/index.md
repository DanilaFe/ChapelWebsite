---
title: "Advent of Code 2022, Day 9: Elvish String Theory"
authors: ["Jeremiah Corrado"]
summary: "A solution to day nine of AoC 2022, covering select-statements, arrays, and math functions"
tags: ["Advent of Code 2022", "How-To"]
series: ["Advent of Code 2022"]
date: 2022-12-09
---


Welcome to Day 9 of our 'Twelve Days of Chapel AoC' series! If you're not familiar
with this series, take a look at the [introductory article]({{< relref "aoc2022-day00-intro" >}})
for some background.





### The Task at Hand and My Approach

If you're anything like me, or the protagonist of [todays AoC Problem](https://adventofcode.com/2022/day/9),
nothing puts your fear of rickety bridges at ease like writing a quantum
rope simulation in Chapel. And that's exactly what we're going to do in this
article!

The goal of our simulation is to track the behavior of a rope's tail given some
movements applied to its head. Each line of our input contains a direction
(up, down, left, or right) and a distance (the number of spaces to move in that
direction through a 2D grid). We are tasked with computing the resultant movement
of the tail according to the following rules. If the tail is:
  * adjacent to the head or overlapping with it, it doesn't move
  * in the same row or column as the head, but more than 1 space away,
      it moves 1 position towards the head
  * in a different row and column, and more than 1 space away, it moves
      diagonally towards the head

With this in mind, I'll use the following approach: store the positions of
the rope's *head* and *tail* in two separate two-element arrays. As
instructions are read from the input, we'll update the position of the head,
and then apply the above rules to compute the tail's new position.

Per the problem statement, we also want to keep track of the number of unique
grid-spaces visited by the tail. I use a set to keep track of all the places it's
been. After all the instructions have been applied, the size of the set will give
us the answer to the AoC problem. (See [Day 3's post]({{< relref "aoc2022-day03-rucksacks#the-set-type" >}})
for more on sets.)

**For those who like to eat the cookie dough before it goes in the oven, here's the
full code:**
{{< whole_file_min >}}






### Reading Instructions

To start out, I `use` a couple of modules from Chapel's standard library. The
`IO` module will give us access to a procedure used for reading the input, and
the `Set` module will give us access to the `set` container type.

```Chapel {data-code-type=main,data-code-section=first,linenos=true,linenostart=1}
use IO, Set;
```

Like in many of the previous articles from this series, I'll define an iterator
to read the raw text input and parse out the useful information:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=3}
iter readCommands() {
  var dir: string, amt: int;
  while readf("%s %i\n", dir, amt) {
    yield (dir, amt);
  }
}
```

Here, `readf` parses each line of input, looking for a string and an
integer—denoted by `%s` and `%i` respectively (the full list of format
specifiers can be found in the [documentation](https://chapel-lang.org/docs/modules/standard/IO/FormattedIO.html#formatted-i-o-for-c-programmers)).

Whenever `readf` encounters some text that matches the given format, it sets the
variables `dir` and `amt` to the parsed values, and returns `true`. The iterator
then yields both values in a two-tuple, and waits for the next iteration
before parsing the next line.





### Setting up the String/Rope

Now that we can read the input, we'll want to set up some variables to
keep track of the rope's head and tail positions, as well as a variable
to hold a collection of the tail's past positions.

According to the problem statement, the head and tail start in the same
position, and don't need to have any specific coordinates. So I define the
variables `head` and `tail` to both have a starting value of `[0, 0]` (a
two-element array of zeros):

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=10}
var head, tail = [0, 0];
```

I'm using the convention that the first element of these arrays
denotes the x-position, and the second denotes the y-position. As such,
I define a set of `param`s that can be used to index into the arrays
by name:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=12}
param X = 0, Y = 1;
```

With these `param`s defined, the following two lines are equivalent:
```chapel
head[0] += 1;
head[X] += 1;
```

Lastly, I initialize a `set` of arrays to keep track of the locations the
tail has visited:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=14}
var visited = new set(tail.type);
```

The set initializer takes a single *type* as an argument designating the
type of the values it stores. In this case, I simply query `tail`'s type
because the set will be storing values of that type later on.





### Executing the Instructions

To start executing instructions, we can consume values from our `readCommands`
iterator in a for-loop. The tuple generated by the iterator is immediately
de-tupled into it's components `dir` and `amt`. Again, these represent the
direction to move the head and how many spaces it needs to move in that
direction:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=16}
for (dir, amt) in readCommands() {
```

Note that we aren't using a parallel loop here, such as a `forall` loop. This
is for a couple of reasons:

1. `readCommands` is a serial iterator. To make a parallel iterator that reads
  from `stdin` would be more trouble than it's worth for a such a small input
  size
2. This problem is inherently serial anyway. In order to get the correct answer
  we have to apply the movements to the head of the rope one-by-one in order.

Next, the `amt` portion of the command is handled with another for-loop:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=18}
  for 1..amt {
```

You'll probably notice that this loop looks a bit different than most
used in this series so far because it has no index variable (i.e., it
doesn't look like: `for i in 1..amt`). This is because we don't actually
need to know which iteration is currently being executed. We only want
to execute the code in the loop `amt` times. As such, I've omitted an
index variable from this inner loop.

Next, we want to update `head`'s position based on the value of `dir`.
With the syntax we've learned so far in this series, we might use the
following code to do this:

```chapel
if dir == "U" then head += [0, 1];
else if dir == "D" then head -= [0, 1];
else if dir == "L" then head -= [1, 0];
else if dir == "R" then head += [1, 0];
```

It checks `dir` against each of the four possibilities and then updates
`head` by adding or subtracting an array that represents a movement along
the given dimension. As in previous posts, this uses promoted operations on
the array.

As you'll notice, it looks a bit repetitive, as each line is re-checking the value
of the same variable. To make this code look cleaner, we can use a
_select statement_ instead of the cascaded *if-then-else* construct.

Select statements work similarly to *switch* or *match* statements in other
languages, and achieve the same effect as above. In the following code, I
check the value of `dir` against each of the possibilities and execute the
relevant code whenever there is a match:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=20}
    select dir {
      when "U" do head += [0, 1];
      when "D" do head -= [0, 1];
      when "L" do head -= [1, 0];
      when "R" do head += [1, 0];
    }
```

{{< details summary="more on *select statements*...">}}

Most languages that support this flavor of control-flow construct also
support a catch-all check that can run some code if none of the other options
were encountered. Chapel is no different. It has the `otherwise` keyword that
can be placed after the `when` expressions.

There are a couple of ways we could have used this in our select statement:

1. If we didn't have total trust in the input, we could use *otherwise* to
check for an unknown direction string. If encountered, we could stop the
program and indicate what the problem was as follows:

  ```chapel
  select dir {
    when "U" do head += [0, 1];
    when "D" do head -= [0, 1];
    when "L" do head -= [1, 0];
    when "R" do head += [1, 0];
    otherwise do halt("unknown direction: ", dir);
  }
  ```

2. Alternatively, if we do trust the input, as with Advent of Code, we could
replace the last when-expression with an otherwise-expression because we know
that `"R"` is the only possibility remaining:
  ```chapel
  select dir {
    when "U" do head += [0, 1];
    when "D" do head -= [0, 1];
    when "L" do head -= [1, 0];
    otherwise do head += [1, 0];
  }
  ```

More information about select-statements can be found in the
[Language Specification](https://chapel-lang.org/docs/language/spec/statements.html#the-select-statement).

{{< /details >}}

### Moving the Tail

And lastly, we'll update the position of `tail` according to the given rules.

Because the rules pertain to the relative positions of the head and tail,
and not their absolute locations, I create a new variable `delta` to store
the relative position:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=27}
    const delta = head - tail;
```

Note that the code above produces a new two-element array because the
subtraction operation is between two arrays of the same size, and thus
becomes a promoted operation. The first element of `delta` will contain
the difference in the distance between head and tail in the x direction,
and the second element will contain the distance in the y direction.

I then use `delta` to apply the rules for the tail's movement. Here, there are
three conditionals, each of which executes a single line, so I use the `then`
syntax rather than wrapping the body of the conditionals in curly-braces:

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=29}
    if abs(delta[X]) > 1 && delta[Y] == 0
      then tail[X] += sgn(delta[X]);
    else if abs(delta[Y]) > 1 && delta[X] == 0
      then tail[Y] += sgn(delta[Y]);
    else if abs(delta[X]) + abs(delta[Y]) > 2
      then tail += sgn(delta);
```

The first two conditionals check for the simpler cases where the tail is either
on the same row or same column as the head, and is more than one space away.

The expressions: `abs(delta[X]) > 1` and `abs(delta[Y]) > 1` check whether the
tail is far enough away from the head in either direction. The expressions:
 `delta[Y] == 0` and `delta[X] == 0` check that the column or row is the same.

In these cases, we want the tail to move one unit towards the head along the
relevant direction. To do this, I use the [`sgn`](https://chapel-lang.org/docs/modules/standard/AutoMath.html#AutoMath.sgn)
procedure (which is automatically included from the `AutoMath` module). It
returns `1` if the argument is positive and `-1` if the argument is negative.

For example, the first branch could have also been written more explicitly as:
```chapel
if abs(delta[X]) > 1 && delta[Y] == 0 {
  if delta[X] > 0 then tail[X] += 1;
  if delta[X] < 0 then tail[X] -= 1;
}
```

If neither of the above conditions is met, I then check for the diagonal
case where the tail is neither on the same row nor same column as the head,
and is at least 2 positions away in any direction. Here, `sgn` is used again,
except in a promoted fashion. The code:
```chapel
tail += sgn(delta);
```
is essentially shorthand for these lines:
```chapel
tail[X] += sgn(delta[X]);
tail[Y] += sgn(delta[Y]);
```

The promotion doesn't make our code that much shorter when working with
two-element arrays; however, if this problem were extended to higher dimensions,
such promotions really come in handy for making code more succinct.

Finally, if none of the above conditions were met, this means the tail and
head either overlapped or were within a distance of one from each other. Per
the rules, we do not move the tail.

### Tracking Unique Positions

At the end of each iteration, we add the tails position to the `visited` set.
If the current position has already been visited, the set will not be modified.
Otherwise, the new position will be added to the set.

```Chapel {data-code-type=main,data-code-section=middle,linenos=true,linenostart=36}
      visited.add(tail);
  }
}
```

At the end of the program, we print out the size of the set, giving us the
total number of unique positions the tail has visited:

```Chapel {data-code-type=main,data-code-section=last,linenos=true,linenostart=40}
writeln(visited.size);
```

### Conclusion and Tips for Part Two

And with that, we'll conclude our discussion. As a review, we introduced one
new concept: select-statements, which are useful for cleaning up repeated
*if-then-else* expressions. We also discussed creating named `param`s for
indexing into arrays, as well as some promoted math operations on arrays. The
full code can be downloaded from the top of this article or found on
[GitHub](https://github.com/chapel-lang/chapel/blob/main/test/studies/adventOfCode/2022/day09/jeremiah/day9a.chpl).

In part two, we are asked to extend this two-element rope/string into a
10-element rope. Much of the logic described above can be reused; however
rather than using a pair of variables to represent the rope, we could
use an array of two-element arrays. The array's first element would be
treated as the rope's head and each subsequent pair of elements would
be updated according to the rules used for this problem.

Thanks for reading! Feel free to check out the other AoC 2022 articles or
to post any questions or comments you have in the [Blog
Category](https://chapel.discourse.group/c/blog/) on Chapel's
Discourse Page. We'll be back with the final three installments of our
'Twelve Days of Chapel AoC' series starting on Monday 12/12 after
a brief hiatus over the weekend.
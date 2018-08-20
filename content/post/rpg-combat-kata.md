---
title: 'RPG Combat Kata'
date: '2016-07-04'
categories:
- F#
tags:
- F#
- Functional Design Patterns
---

A couple of weeks ago I saw these tweets (in Spanish):

<blockquote class="twitter-tweet" data-lang="en"><p lang="es" dir="ltr">Hoy en la ofi hemos empezado la mañana con la RPG Combat kata de <a href="https://twitter.com/SuuiGD">@SuuiGD</a> que hicimos en el <a href="https://twitter.com/hashtag/scpna?src=hash">#scpna</a> :-D <a href="https://t.co/vsK0OucncD">https://t.co/vsK0OucncD</a></p>&mdash; Xabi Sáez de Ocáriz (@ziraco) <a href="https://twitter.com/ziraco/status/745540286441349125">June 22, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" data-lang="en"><p lang="es" dir="ltr"><a href="https://twitter.com/SuuiGD">@SuuiGD</a> <a href="https://twitter.com/ziraco">@ziraco</a> a mi me flipó tanto que estoy haciendo la versión &quot;Extended&quot;.si sigo así, le pongo UI y al store! ;)</p>&mdash; Modesto San Juan (@msanjuan) <a href="https://twitter.com/msanjuan/status/745640741737758720">June 22, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

As I respect enormously [Modesto](https://twitter.com/msanjuan) and [Xabi](https://twitter.com/ziraco)'s opinion and I trust on the skills of a [Carlos Ble](https://twitter.com/carlosble)'s [apprentice](https://twitter.com/SuuiGD), I decided to do the [kata](http://www.slideshare.net/DanielOjedaLoisel/rpg-combat-kata) at home. And as I keep trying to learn F#, that would be the chosen language.

I recommend you to do the kata, is a very interesting one. In case you want to do it, please stop reading now and come back when you have finished it, as I'm going to explain how I solved it.

![Spoiler alert](images/Spoiler-Alert.jpg)

The kata starts pretty simple: you have to  model a combat system between two characters of a RPG game. First you start with simple rules and in every iteration you add a slightly more complex one. At the end of the 4th iteration I ended up with a code like this:

<script src="https://gist.github.com/vgaltes/c7050947af0670cef422d19861689417.js"></script>

As you can see, I have a list of rules that should be applied when attacking and a list of rules that should be applied when healing. I have a function to traverse a list of rules and apply them, and the rules are quite small and focused in one thing. I was quite happy with the design (remember, I'm learning Functional Programming, I can be totally wrong :) )

The real fun starts in the last iteration. In this iteration a new element of the game is introduced: a Thing. A Thing is an element that has health but nothing else. It can't heal anyone, it doesn't belong to any faction and it can't attack. Is just something that everybody can attack.

Looks like a simple change but introduces quite a bit of complexity in the design. I decided to use a discriminated union to model a Player, so that a player can be either a Character of a Thing.

<script src="https://gist.github.com/vgaltes/4ad26d3232862407cb1d23f746bd87b6.js"></script>

My first attempt to change the logic of the module was focused in two things:
 - Simplify the logic of the different rules, so that they just only need as a parameters the information they will use.
 - Put the logic around being a Thing or a Character in the main function

This solution ended up with a function like this:

<script src="https://gist.github.com/vgaltes/a64c9e36fcc21ea19800fe90dd406eb6.js"></script>

[Full code](https://github.com/vgaltes/RPGCombatKata/blob/190f10282367e2a13375f00b684e251c7287b2f7/RPGCombatKata/Combat.fs)

As you can see, the code is a bit messy. It's true that the rules now are a bit simpler and only receive as a parameter what they need, but on the other hand we have now three different types of rules that must be traversed in the main function.

So I decided to refactor my solution a bit. What I tried to accomplish was:
 - Move the logic around being a Thing or a Character to the rules. Now a rule is the responsible of knowing what to do in each case.
 - Simplify the logic of the main function. Now, it just differentiate about an Attack and a Heal, calls the same function to traverse the list of rules (different in each case), and applies the result in a different way.

Here's the code of the main function:

<script src="https://gist.github.com/vgaltes/e3d52f6638d5250e28f513405e6b0448.js"></script>

And here's the code of a couple of rules:

<script src="https://gist.github.com/vgaltes/ee3b6f8b510aa50baefae2af7234c073.js"></script>

I prefer this approximation. I don't have a nested pattern matching in the main function. I only have a list of rules for each case and the rules know what to do in each case. My concerns are:
 - What will happen if the types of players grow? Is there any way to generalise the types (i.e. a player that could be attacked, healed, etc)?
 - What will happen in the number of parameters of the rules grow? Is a smell that the design is not good enough? Should I introduce a parameter object (well, in F# maybe is a parameter record type :P )?

I'd love to hear your thoughts about the design :)
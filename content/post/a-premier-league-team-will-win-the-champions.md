---
title: 'A Premier League team will win the Champions League this year'
date: '2016-4-1'
categories:
- FSharp
tags:
- April's fools day
- F#
---

Spain and Germany are dominating with an iron fist last Champions League editions. After a lot of investment, a Premier League team is ready to conquer the longed for trophy. In this article we'll demonstrate this fact.

## The data

We've borrowed the data for this study from the UEFA's official page. If you go to this address [http://www.uefa.com/uefachampionsleague/season=2011/matches/all/index.html#](http://www.uefa.com/uefachampionsleague/season=2011/matches/all/index.html#) you'll see all the matches played in the season 2010/2011. Change the year in the query string to see another year's results. With this information we've created a very simple CSV file which summarises the competition from the round of 16. We've taken the resuls from the season 2004/2005 because is the first season with the actual format (the knockout rounds start at the round of 16). The CSV file looks like this:

    year,round,team1,team2,winner
    2004,8,Lokomotiv Movska,Monaco,Monaco
    2004,8,Celta,Arsenal,Arsenal
    2004,8,Bayern,Real Madrid,Real Madrid
    2004,8,Sparta Praha,Milan,Milan
    2004,8,Stuttgart,Chelsea,Chelsea
    2004,8,Porto,Man. United,Porto
    2004,8,Real Sociedad,Lyon,Lyon
    2004,8,Deportivo,Juventus,Deportivo
    2004,4,Porto,Lyon,Porto
    2004,4,Milan,Deportivo,Deportivo
    2004,4,Real Madrid,Monaco,Monaco
    2004,4,Arsenal,Chelsea,Chelsea
    2004,2,Monaco,Chelsea,Monaco
    2004,2,Porto,Deportivo,Porto
    2004,1,Monaco,Porto,Porto

## Loading the data

We have a CSV file, we're going to use F#... [CSV type provider](http://fsharp.github.io/FSharp.Data/library/CsvProvider.html) to the rescue!! 

We're going to use [Paket](https://fsprojects.github.io/Paket/) so add this lines to your paket.dependencies file

    source https://nuget.org/api/v2

    nuget FSharp.Data
    
And this line to the paket.dependencies file of your project:

    FSharp.Data

Run the install command of paket and you'll have FSharp.Data referenced in your project. To use it from your script file, we have to reference it:

    #r "../packages/FSharp.Data/lib/net40/FSharp.Data.dll"
    
and open it:

    open FSharp.Data
    
Now we're ready to load the data. To keep things simple we're going to use just a couple of types to store the data

    type Team = string
    type RoundGame = {Year: int; Round: int; Team1:Team; Team2:Team; Winner:Team}

Let's use the fantastic CSV type provider to load all the games:

    type ChampionsLeague = CsvProvider<"year,phase,team1,team2,winner", Schema = "year(int),phase(int),team1,team2,winner">

    let file = __SOURCE_DIRECTORY__ + "\Data\champions.csv";
    let text = File.ReadAllText(file)

    let championsLeagues = ChampionsLeague.Load(file);

And finally, let's parse the data into the recently defined types:

    let champions =
        championsLeagues.Rows
        |> Seq.map(fun r -> {Year = r.Year; Round = r.Phase; Team1 = r.Team1; Team2 = r.Team2; Winner = r.Winner})

## Glories from the past

First of all let's review how many times a Premier League team has won the Champions League in the last twelve years. Premier League teams are very powerfull and they play a great football, I'm sure we'll find a lot.

    let championsWonBy teams =
        champions
        |> Seq.filter(fun f -> f.Round = 1 && teams |> Array.contains f.Winner)

    championsWonBy [|"Liverpool"; "Man. United"; "Chelsea"; "Man. City"; "Arsenal"|] 
    
Not too bad. Liverpool in 2005, Manchester United in 2008 and Chelsea in 2012 won the Champions League. So, every 3.5 years a Premier League team wins the Champions League. Maybe 2016 will be the next time?

Let's compare that with other leagues, I'm sure Premier League will be the strongest one!

Germany and Portugal have won 1 cup, Italy 2 and Spain 5. That puts Premier League in second position, not bad!

As I'm F.C. Barcelona fan, let me see how many Champions League we won in the past twelve years... 4. One more than the whole Premier League... Well, we have [Messi](https://www.youtube.com/watch?v=r7Wa3ZKRZc8). It's like cheating a bit... ;)

## Round of 16

In the round of 16 there were two teams representing Premier League: Arsenal and Manchester City. Arsenal played against Barcelona and they lost. Let's study their last matches to see if that was an unexpected result:

    let roundWith team1 team2 =
        champions
        |> Seq.filter(fun f-> (f.Team1 = team1 && f.Team2 = team2) || (f.Team1 = team2 && f.Team2 = team1))

    roundWith "Arsenal" "Barcelona"
    
That gives us three results: Final of 2006, quarter-finals of 2010 and round of 16 of 2011. In all theses matches Barcelona won, so it wasn't a great surprise that this year they've won too...

Let's study Manchester City a bit. We can start analysing how many times they've played a quarter-final match.

    let timesInPhase phase team =
        champions
        |> Seq.filter(fun g -> (g.Team1 = team || g.Team2 = team ) && g.Round = phase)
        |> Seq.length
        
    "Man. City" |> timesInPhase 4

Wow, they never played a quarter-final game! Let's study then their games in round of 16. They are a very rich and powerful team, so I guess they have played a lot of games in that round.

    "Man. City" |> timesInPhase 8

Mmmmm... only two. Let's take a look at those games
        
    let gamesInRound round team =
        champions
        |> Seq.filter(fun g -> (g.Team1 = team || g.Team2 = team ) && g.Round = round)

    "Man. City" |> gamesInRound 8

They played both times against Barcelona and they lost... So, Barcelona is definetively a rival to avoid in the next round.

Manchester City plays against Paris St Germain. Have they played any game before?
    
    roundWith "Man. City" "Paris"

No, they've never played before. Let's take a look at PSG games in quarter finals:

    "Paris" |> gamesInRound 4
    
PSG has played three times in quarter finals. Two against Barcelona (2013 and 2015) and one against Chelsea (2014). They have lost the three of them, so it could be a good team to play against.
 
## Semifinals
 
Let's imagine Manchester City wins PSG at quarter-final round. Which team could be the best rival to play against? Let's see if anyone of those teams have never played a semi-final round.
 
    let rivals = ["Barcelona"; "Real Madrid"; "Atletico"; "Wolfsburg"; "Bayern"; "Benfica"]

    rivals
    |> Seq.filter(fun f -> timesInPhase 2 f = 0)
    
Wolfsburg and Benfica have never played a semi-final. Actually, only they have never played a final (remember, in the last twelve years). And yes, this is the first time they are playing a quarter final. So let's study a bit their rivals to see which of them have more chances to win.

Wolfsburg plays agains Real Madrid. Let's see how Real Madrid played the quarter final round:

    "Real Madrid" |> gamesInRound 4 |> Seq.length
    "Real Madrid" |> gamesInRound 4 |> Seq.filter(fun f-> f.Winner <> "Real Madrid") |> Seq.length
    
They played six times and they won five. They don't seem a good team to play against.
 
Lets take a look at Bayern, Benfica's rival. They've played eight times the quarter final round, and they've been eliminated three times. In 2005 they lost against Chelsea (finalist). In 2007 they lost against Milan (semi-finalist) and in 2009 they lost against Barcelona (winner). So, although they've lost against great teams, it looks like Benfica has more chances to win them than Wolfsburg to win Real Madrid.
 
In case nor Benfica neither Wolfsburg can win their games, who will be a good rival? Well, we can say that the best rival is the one that has less percentage of winnings in semi-finals. Let's calculate it.

    let winsInRound round team =
        let games = team |> gamesInRound round
        float (games 
                |> Seq.filter(fun f -> f.Winner = team)
                |> Seq.length ) / float (games |> Seq.length)

    rivals
    |> Seq.map(fun f -> f, f |> winsInRound 2)
    |> Seq.sortBy snd 

Looks like Real Madrid is the worst team playing semi-finals. So, if Benfica or Wolfsburg can't pass to semi-final, maybe Real Madrid could be a good rival.

## Final

Let's get the teams with worst percentage of victories in a Champions League final.

    rivals
    |> Seq.map(fun f -> f, f |> winsInRound 1)
    |> Seq.sortBy snd
    
Well, Barcelona is not a very good rival... They've played four finals and they've won all of them. Something similar applies to Real Madrid, but just for one final. Bayern only wins one of every three finals they play. And Atletico only has played one final and they've lost it. So, if Manchester City gets to the final, Atletico could be a good rival.

## Recap

Well, taking a look at the last twelve years results it's quite clear that no Premier League team will win this Champions League edition. But there's a little chance to win if they play well and have luck in the next draw. Only one thing seems clear: don't play against Barcelona!! :-)
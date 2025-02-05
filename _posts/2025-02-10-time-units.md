---
layout: post
title: Time Units, not Action points
author: Alexandre
tags:
- gobs-and-gods
- gameplay
toc:  false
---

In Gobs and Gods, battles are primarily turned based, and each Gob has each turn a number of "time units" available to act. 
However it possible to switch the battle to a real time mode whenever you want, and to switch back to turn based when you need more control. It's even possible to play fully in real time, but the gobs may be a bit more difficult to control in this setting.

![switch button]({{ 'assets/images/timeunits/switch.jpg' | relative_url }})
*Battle mode can be switch any time. Just click here, or press enter!*

- So how does this work? 
- What happens when the players change the battle mode? 
- Is there some hidden advantage to play in turn based mode instead of real time? 
- Are "time units" different from "action points" commonly found in turn-based games?
- Does the "Speed" stat of a Gob have an impact on the game in turn based mode? 

## Time Units

In Gobs and Gods, each action has a cost, defined in Time Units (TU).
For example, attacking with a 2-handed cost 6 TU.
This number is always an integer.

![orders time units]({{ 'assets/images/timeunits/order_bar.jpg' | relative_url }})
*The action bar shows the time unit and essence (ie mana) cost of available orders. Gob's remaining TU this turn are also visible.*

## From time units to duration in seconds

The game internally computes the duration (in seconds) from the time units cost. Formula here is simple:

```
Action_Duration = Time_Units x Time_Unit_Duration( Gob )
```

the `Time_Unit_Duration` depends of the Gob. It starts at 0.25s, but is modified by several factors such as the gob's Speed stat, an eventual bonus from the sprint skill, a malus from over-encumbrance, ...
The formula looks like this:

```
Time_Unit_Duration( Gob ) = 0.25 * encombrance_Factor_Above_1 * 100.0f / (100 + gob.Speed.Value * 2 + sprint_Bonus))
```

But how is this duration used in Turn-based mode? Before answering that, let's look how it works in Real Time, which is more straigforward to understand.

## Real-time mode

- the game keeps track of `Current_Time`, which is simply the time elapsed since the beginning of the battle. 
- and each gob keeps track of the next time when it can act.
After a gob action, his `Next Action Time` is defined like this:

```gob.Next_Action_Time = Current_Time + Action_Duration ```

and testing if a gob is ready to act simply requires comparing his `Next_Action_Time` to the game's `Current_Time` :

```
 Can_Act_Now(gob) = gob.Next_Action_Time <= Current_Time 
```

![gob's timeline]({{ 'assets/images/timeunits/timeline.jpg' | relative_url }})
*When a gob acts, it moves on the timeline by a distance equal to the action duration.*

Simple? So let's now look how it works in Turn based mode.

## Turn based mode

In turn based mode, the game's `Current_Time` is frozen to the time at the start of the player turn.
But gobs still keep track of their `Next_Action_Time`. 
The update is almost the same: the only difference is that here we cannot rely on `Current_Time` to tell when the action begins. instead we just increase the gobs's `Next_Action_Time` by the `Action_Duration`. Like this:

```
gob.Next_Action_Time = gob.Next_Action_Time + Action_Duration
```

And the main difference with real time is how we decide if a gob can still play. To do that, we compare the gob's `Next Action Time` with the time of the end of the turn:  
  
``` 
End_Of_Turn_Time = Current_Time + Turn_Duration
Can_Act_Now(gob) = gob.Next_Action_Time < End_Of_Turn_Time  
```

![gob's timeline turn 1]({{ 'assets/images/timeunits/turn1.jpg' | relative_url }})
*The same timeline is used in turn based mode. Here the gob acted twice during the player's turn. He moved past the end of turn, and thus cannot play any more.*


![gob's timeline turn 2]({{ 'assets/images/timeunits/turn2.jpg' | relative_url }})
*The next player's turn begins, and the gob can play again. His available time this turn is lower, because he 'borrowed' some time during the first turn to play a long action.*

And when a gob pass? Simple, it just sets the gob's `Next_Action_Time` to `End_Of_Turn_Time`.

## AI turn
Once all player gobs have played, the game switch to the AI turn.
AI turn works exactly like real time: we start increasing the game Current_Time, and enemies act when there own Next_Action_Time gets equal to the Current_Time.
Since the player gobs have already played, their `Next_Action_Time` is higher than `Current_Time` and they cannot play...
... until `Current_Time` reaches one gob's `Next_Action_Time`. When this happen, the AI turn stops and a new player's turn begins.

## Back to time units

Of course we can't tell the player "Your gob can still act for 1.2345s". That would be a very impractical information to display.
Instead, the remainig time is converted back to an integer number of remaining time units.  
So how many time units are left to a gob?
 
```
Remaining_Time = End_Of_Turn_Time - gob.Next_Action_Time
Remaining_Time_units = Round_up( Remaining_Time / Time_Unit_Duration( Gob ) ) ```

Here we round the remaining time *up* ( ie 3.001 is rounded to 4) because a gob can act so long as its current time is lower than the end of the turn: a gob with "3.001TU" worth of remaining time, which plays a 3 TU action will still have 0.001 TU after the action; which means he will be able to play again! Indeed his `Next Action Time` will still (a bit) lower than the turn' end time. 

![gob's remaining time units]({{ 'assets/images/timeunits/turn2.jpg' | relative_url }})
*the gob's available TU can be seen below its health bar. Time units consumed by mouse's current order are shown in darker orange.*

## Time units, not Action Points

This system allows to seamlessly switch between real-time and turn by turn, keeping all the rules perfectly identical in both modes.
It implies however a few noticeable differences with the "action point" based systems that many players may be more familiar with.
Here is the main one:
- any action can be played with only one time unit. This is different from most 'action point' game, where an action costing 3AP cannot be played if you have only one AP left.
- but playing an action which cost more TU than currenly available means the gob will get less TU during its next turn ( because his Next_Action_Time would advance further than the next turn start time.)

In other words, this allows to "borrow" time from the next turn to play now.



 

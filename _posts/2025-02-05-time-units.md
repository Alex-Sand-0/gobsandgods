---
layout: post
title: Time Units, not Action points
author: Alexandre
tags:
- gobs-and-gods
- gameplay
toc:  false
---

In *Gobs and Gods*, battles are primarily turn-based, with each Gob having some "time units" available each to take actions.
However, you can switch to real-time mode at any point and return to turn-based mode when you need more control. It's even possible to play entirely in real-time, though managing the Gobs in this mode can be a bit more challenging.


![switch button]({{ 'assets/images/timeunits/switch.jpg' | relative_url }})
*Battle mode can be switch any time. Just click here, or press enter!*

- How does this system work?
- What happens when the player switch between battle modes?
- Are there any hidden advantages to playing in turn-based mode rather than real-time?
- Are "time units" different from the "action points" commonly found in turn-based games?
- Does a Gob's "Speed" stat influence gameplay in turn-based mode?

## Time Units

Each action performed by a Gob in battle has a cost measured in Time Units (TU).
For example, attacking with a two-handed sword costs 6 TU.
This cost is always an integer.

![orders time units]({{ 'assets/images/timeunits/order_bar.jpg' | relative_url }})
*The action bar displays the Time Unit (TU) and Essence (mana) costs of available actions.It also shows the Gob's remaining TU for the current turn.*

## From time units to duration in seconds

The game internally computes the duration (in seconds) from the time units cost. The formula is straightforward:

```
Action_Duration = Time_Units x Time_Unit_Duration( Gob )
```

the `Time_Unit_Duration` depends of the Gob. It starts at 0.25s, but is modified by several factors such as the gob's Speed stat, an eventual bonus from the sprint skill, a malus from over-encumbrance, ...
The formula looks like this:

```
Time_Unit_Duration( Gob ) = 0.25 * encombrance_Factor_Above_1 * 100.0f / (100 + gob.Speed.Value * 2 + sprint_Bonus))
```
As you can see, a Gob's Speed reduces the duration of its Time Units, thus shortening the duration of all its actions.
But how is this duration applied in Turn-Based mode? Before answering that, let’s first examine how it works in Real-Time mode, which is more straightforward to understand.


## Real-time mode

- The game keeps track of `Current_Time`, which is simply the time elapsed since the beginning of the battle. 
- Each Gob also tracks the next moment it can act.
After a gob action, this `Next_Action_Time` is defined as follow:

```gob.Next_Action_Time = Current_Time + Action_Duration ```

Determining if a Gob is ready to act is as simple as comparing its `Next_Action_Time` to the game’s `Current_Time`:

```
 Can_Act_Now(gob) = gob.Next_Action_Time <= Current_Time 
```

![gob's timeline]({{ 'assets/images/timeunits/timeline.jpg' | relative_url }})
*When a gob acts, it moves on the timeline by a distance equal to the action's duration.*

Now, let’s take a look at how it works in Turn-Based mode.

## Turn based mode

In turn based mode, the game's `Current_Time` is frozen at the start of the player's turn.
But gobs still track their `Next_Action_Time`. 
The update is almost the same: the only difference is that, in this case, we cannot rely on `Current_Time` to determine when the action begins. Instead we just increase the gobs's `Next_Action_Time` by the `Action_Duration`, like this:

```
gob.Next_Action_Time = gob.Next_Action_Time + Action_Duration
```

The main difference with Real-Time mode is how we determine if a Gob can still act. To do this, we compare the Gob's `Next_Action_Time` with the time at the end of the turn:  
``` 
End_Of_Turn_Time = Current_Time + Turn_Duration
Can_Act_Now(gob) = gob.Next_Action_Time < End_Of_Turn_Time  
```

![gob's timeline turn 1]({{ 'assets/images/timeunits/turn1.jpg' | relative_url }})
*The same timeline is used in turn based mode. Here the gob acted twice during the player's turn. He moved past the end of turn, and thus cannot play any more.*


![gob's timeline turn 2]({{ 'assets/images/timeunits/turn2.jpg' | relative_url }})
*The next player's turn begins, and the gob can play again. His available time this turn is lower, because he 'borrowed' some time during the first turn to play a long action.*

And when a gob passes? Simple, it just sets the gob's `Next_Action_Time` to `End_Of_Turn_Time`.

## AI turn
Once all the player’s Gobs have acted, the game switches to the AI turn.
The AI turn works exactly like Real-Time mode: the game continuously increases the `Current_Time`, and enemies act when their own `Next_Action_Time` matches the `Current_Time`.
Since the player’s Gobs have already acted, their `Next_Action_Time` is higher than `Current_Time`, so they cannot act...
...until the `Current_Time` reaches one player's Gob's `Next_Action_Time`. When this happen, the AI turn stops and a new player's turn begins.

## Back to time units

We’ve seen that, even in Turn-Based mode, the game internally uses a continuous timeline.
We can define the remaining time for a Gob during this turn, which determines how much it can still act:

```
Remaining_Time = End_Of_Turn_Time - gob.Next_Action_Time
```

But this would be very impractical information to give to the player. 
"*Your Gob can still act for 1.2345 seconds.*" Does this mean it can attack once? Or twice?

But remember, action duration is defined in Time Units! And all Time Units for the same Gob have the same duration.
We can thus convert the remaining time into remaining Time Units. So, how many Time Units are left for a Gob?

```
Remaining_Time_units = Round_up( Remaining_Time / Time_Unit_Duration( Gob ) ) 
```

Here we round the remaining time *up* ( ie 6.001 is rounded to 7) 
cause a Gob can act as long as its current time stays below the end of the turn.
because a gob can act so long as its current time is lower than the end of the turn:
 a gob with "6.001TU" worth of remaining time, which plays a 6 TU action will still have 0.001 TU left after the action.
This means his `Next Action Time` will still be (slightly) lower than the turn's end time, allowing him to act once more! 

And this makes it much clearer to the player: a Gob with 7 TU can attack once with a two-handed sword (6 TU) and still perform another action this turn.

![gob's remaining time units]({{ 'assets/images/timeunits/turn2.jpg' | relative_url }})
*the gob's available TU can be seen below its health bar. Time units consumed by mouse's current order are shown in darker orange.*

## Time units, not Action Points

Here we note the main difference this "Time Unit" system and the "Action Points" used in many strategy games:
**It is possible to play any action with just One TU**. Most games with "action points" require to the have the full action points cost available to perform the act.
However, playing an action that costs more TU than is currently available means the Gob will have fewer TU in its next turn, because its Next_Action_Time would advance beyond the start of the next turn.
In other words, this system allows the Gob to "borrow" time from the next turn in order to act now.

This has consequences for gameplay and tactics:

- It’s not as critical to track the remaining "Time Units" to maximize a Gob’s actions during the battle, as with an "action point" system, where unused action points are lost.
- However, counting Time Units still matters, for example, to determine if you can attack a second time and potentially kill an enemy before their turn.
- Sometimes, it might actually be better to pass early to ensure the Gob will have the most Time Units available for their next turn.


---
layout: single
toc: true
title: 'Unreal Engine 5 Finite State Machine (UE5FSM)'
description: 'What is UE5FSM?'
date: 2024-01-10
permalink: /posts/2024/01/Unreal Engine 5 Finite State Machine/
tags:
   - UE
   - AI
   - UE5FSM
---

# Introduction

Unreal Engine offers many tools to create AI for games, which can be used together to increase the complexity of
the AI without spending too much time on development. The main tools include Behavior Trees (BT), Blackboards (BB), 
and Environment Query System (EQS). Each of them has its pros and cons, deserving its own topic. However, in this 
article, I will present a tool I've been developing and using on my own to create AI and other stateful logic.

When listening to talks about AI, some people suggest splitting it into 3 areas - Data Calculation, Database, and
Execution. The AI Controller typically handles the data calculation, along with the EQS or any other computation tool.
The BB serves as the database, and the BT handles execution. However, in reality, this academic approach is often mixed
up. The BT not only uses the data contained in the BB but also computes something, such as running an EQS to gather more
data. This blurs the line between the first and third areas. Additionally, creating custom nodes for this purpose,
especially in C++, can be cumbersome. Another point to mention is that the BT is inherently bad at transitioning 
between states - there's no easy way of doing that without treating it as a state machine.

Don't get me wrong: I'm not entirely against them. In fact, I've developed some AI using BTs that made it into
final products, yet they weren't very complex.

During my learning journey, I discovered UE's 3 Finite State Machine (FSM) used in Unreal Script which allowed
developers to make pretty complex logic using many benefits of the old scripting language. I personally prefer it over
BTs, and it's a shame that they were dropped with UE4. The system was pretty powerful, it was capable of easily adding
new states and blocking transitions to specific states. It also supported coroutines in certain functions, allowing 
developers to easily create complex behavior without writing excessive boilerplate code to preserve certain states.

Since I prefer UE's 3 FSM over BTs to do my job, I decided to port it to UE5 myself. The 
[plugin](https://github.com/Tonetfal/UE5FSM) ended up to be similar to UE's 3 FSM, and it should contain all its 
features, however it has some additional features that I personally consider useful. There's an important point to 
mention - the development is mainly focused in C++. There's some BP interaction, but it doesn't have anything to do 
with coding.

# Description

Finite-State Machine is a model of computation. It is an abstract machine that can be in exactly one of a finite 
number of states at any given time. That's a powerful tool to make any stateful logic for anything you like. 

UE5FSM expands this definition a little bit. More exactly the `the machine can be in exactly one of a finite
number of states at any given time` part, adding one single word `the machine can be in exactly one of a finite
number of NORMAL states at any given time`. There are different types of states: normal and global.

## States

A normal state is, as the name implies, just a regular state everyone would expect in an FSM. The machine can switch 
between them back and forth, and have only one of them active at a time.

On the other hand, a global state is always active serving as a supervisor to the machine. It can run 
any logic the whole time regardless of the current active normal state. It's just another place you can create your 
very specific logic in. For instance, when creating AI for a combat game, the AI most likely is going to have some aggro
and ignore targets list which have to be processed every frame to make the AI react to the world around it. One 
could handle that logic inside AI Controller or the AI character itself, but I personally prefer having it under the 
global state as in this case it's strictly related to the AI processing.

## Features

The states come with different features to ease the development. They have means to communicate with the 
state machine: switch between each other, disable themselves, manipulate the states stack, which will also be 
covered in this article.

### State Data

State Data is an object each state can have. The object can be thought about as a blackboard that is accessible to every 
state allowing states to easily pass the information around. However, since it's easily accessible from everywhere,
you must be careful with them! The objects persist throughout the whole lifecycle of a state, allowing it to store 
its data so that it can use it once it is activated.

### Labels

Labels are special functions a state can have. [Coroutines](https://github.com/landelare/ue5coro) is what makes them 
special, as they are capable of running latent code allowing to create complex behavior without any boilerplate code.
Labels are like mini-states inside machine states, as they can be switched between each other, and only one of them 
can be active at a time.

The latent functions can either finish executing or be canceled using existing FSM tools. For instance, while the AI 
is seeking a target (Seeking state), which includes movement, it might be attacked by a player. In response,
we might want the AI to enter the Hurt state and play some animation. However, during the hurt state, the 
movement from the seeking state might still be playing. To cancel that, we can cancel all the running 
latent execution from the hurt state, and not only that, but any other state. This applies not only to movement but 
to any other action - rotation towards an actor, attacking, interacting, etc.

### States Stack

Unlike regular FSMs, UE5FSM has a states stack that allows users to push and pop states on the fly, creating complex 
behavior without manually tracking states. The stack has some restrictions: a state cannot be pushed if it's already 
present on it. Other than that, there are no additional stack-related restrictions.

Returning to the previous example, our AI gets hurt while seeking for an enemy. Instead of transitioning to the hurt
state, we can push it on the stack. After finishing the hurt animation, it pops itself, leaving us with the same state 
it transitioned to the hurt state from. This approach also works if the AI gets stunned. Pushing the hurt state on 
top of the stunned state results in a return to the stunned state afterwards. 

It's a powerful mechanism that allows ignoring the state being pushed. GotoState approach were used, one would need to 
remember the previous state and use GotoState with that one after finishing the Hurt state. This operation becomes 
cumbersome considering how often states are pushed.

## Closing thoughts

UE5FSM provides powerful tools to develop stateful logic, especially when it comes to AI. Using an FSM 
establishes a clear border between states and allows creating separate logic for each action, avoiding monolithic 
structures. 

Visit the [plugin's Github page](https://github.com/Tonetfal/UE5FSM) to learn more about it, as it has 
comprehensive documentation. This article didn't include any real examples, but we'll cover that in the future.

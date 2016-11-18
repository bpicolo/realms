---
layout: post
slug: "architecture"
title:  "The Architecture of Realms"
date:   2016-11-16 17:17:41 -0500
categories: networking architecture
comments: true
---

This blog post is a living document - a maintained overview of the architecture
of Realms on a number of fronts. This is where the complexity of an MMO begins, and ends,
so it's important to have a clear vision of the intent.

# The overall architecture

To kick us off, here's a diagram of data flowing through the entire set of applications.

![Architecture Diagram]({{ site.baseurl }}/assets/architecture-diagram.png)

So, what's is this picture even trying to say? Let's break down the concerns of each component.

## The player
Clicks mouse, potentially receives entertainment.

## Unity Client
The Unity client is the "frontend" of the game - it's the game running on a player's computer.
Non-networked games stop here - they consist only of a client.

The client is an untrustworthy source of data. The only thing we entrust to the
user at all is his input - things he types, and clicks. What that input actually does
is almost entirely determined by the server.

The game is currently using the UNet HLAPI to talk to the Unity backend.

The client talks over the network both to our Unity backend and our Elixir server.
The vast majority of it's communication is to the Unity backend. The Elixir
server will only be contacted for matters unrelated to game logic (logging in is the
current example).

## Unity Headless Backend
The Unity backend runs most of the same code as a the Client, with the exception
of a few parts that run on the server only. The movement of a player for each frame, for example, is calculated on this backend
in addition to our client. *Headless* just means it doesn't do anything visually (no graphics calculations).

Currently, there's no sharding whatsoever, though I expect to shard the backends
on a per-zone basis. Players moving between zones will begin communicating
with a new server. Each backend will be responsible for maintaining the state of that zone - the
monsters within it, items on the floor, other world objects. In addition to this, it communicates
with the API server on behalf of the players.

The backend is the moderator of the game world - given actions that a player attempts to take, it determines
whether the user can actually take that action, and performs it. It syncs the result
of that action back to all relevant players.

The backend is also the arbiter of all game-related communications to our Elixir server, which I'll explain in
better detail in the upcoming post on logins.

Data stored only in the Unity backend is **not** persistent. If the server crashes,
it will take all of it's data with it. Currently, this means that items
on the ground when a server crashes will be forever lost to the wind (though I
will probably change that particular case eventually).

## Elixir backend
Context: See [here][1] for info on the programming language Elixir,
and [here][2] for information on the web framework I'm using.

First things first: Elixir is a super cool functional programming built on top of the Erlang virtual machine (BEAM).
I'm using it partly because I want to learn more about it, and improve at it in general, but also because
it's a powerful language with a lot of interesting properties.

This backend is an API server. All endpoints are currently in JSON. There is a potential
future that uses [Protobuf][3] instead,
though the main limitation there was that neither Unity's Mono nor Elixir have first
class support for it. (In particular, [GRPC][4] is what I would
have defaulted to if OSS were readily available and stable already for the languages in question).

The Elixir backend implements transactional semantics for actions that a user takes.
As an example, say a user wants to pick up an item. This entails: Removing an item
from the ground, and adding it to the the player's inventory. If we
didn't perform this atomically (all at once), we'd have either item duping bugs,
or the possibility that our Unity backend crashes between `remove the item from the ground`
and `add the item to the player's inventory`, thereby killing our poor player's loot pickup.

We need this backend in addition to the Unity backend for a few reasons. The first is that
it's just far superior for all things related to communicating with our database compared to
the hamstrung C# that Unity provides. In addition, it will have a number of concerns, like chat,
that it can scale virtually infinitely without any concern, while the Unity backends have more limitations
on their ability to scale. It also allows us to perform tasks that relate to many backends,
without having to add some sort of meshed network for the Unity backends to talk to eachother.

It's also reasonable to ask the opposite - why have the Unity backend at all?
The main reason here is to avoid rewriting game logic twice - both in the backend
and the client. The Unity backends are a massive time saver with their ability to fact-check
the user's input by running the same code. This is a scope-creep tradeoff.

## Postgresql
There's not too much to talk about here. Postgresql is our data persistence layer.
Data in postgres will stick around forever. Postgresql is a fantastic database.


# How the components communicate
Coming soon.


[1]: http://elixir-lang.org/
[2]: http://www.phoenixframework.org
[3]: https://developers.google.com/protocol-buffers/
[4]: http://www.grpc.io/

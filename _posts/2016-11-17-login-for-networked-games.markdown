---
layout: post
slug: "logins"
title:  "Logins for networked games in Unity3D"
date:   2016-11-17 17:17:41 -0500
categories: networking logins unity3d
comments: true
---

# Who is your daddy, and what does he do?

In networked games, it's pretty important to know who's who. In networked RPGs this is doubly important - we need to make sure that MrFuji's loot can't be stolen by xSephirothXx, and GrandmaNelly can't see the dirty loot hiding in your bank.

So, how can we keep track of what player is doing what? What items belong to him? Which
host he's connected to? Do we just have the player tell us who he is, and leave it at that?

Like I've mentioned in a previous post, networking all comes down to one thing.

## Trust.

We can't trust the player for anything. That's the important thing to remember. The good new is, dealing with trust is a pretty solved problem. The apps you use and the websites you visit all have this same basic problem. Before we dive into a solution, let's enumerate the problems a login solves.

1. We should know which user is accessing our application for each call to our application.
2. It should be difficult or impossible for a user to pretend to be a different user.

There are plenty of related problems, but those are the basic ones.


To kick us off, let's briefly touch on the mechanism a typical website uses to log a user in (though
I won't dive into the specifics of e.g. secure password storage).

![Website Login Diagram]({{ site.baseurl }}/assets/login-diagram.jpg)

Essentially, a user gives us their credentials (typically a username and a password), and
the website verifies that the password given matches the expected password for
that user.

If it's a match, we generate a random session token. This is usually just a long, random string of
base16 or base64 digits (longer than in the example), though it doesn't matter what it is. It just needs to be long and random so that users can't guess another person's session ID.

In some database or key-value store, we'll store a mapping of Session ID -> User information.

We send this session back to a user (typically as a browser cookie). After this,
the browser will automatically send that session ID back with every page the user
loads. To know which user is accessing our site, we simply fetch the user information
from our mapping.

With this, we've solved the problem of trust. Because our session IDs are long, and random,
unlike typical user ids (which are just incremented numbers 1, 2, 3...), it's essentially
impossible for a user to guess that of another user.

It also means we know which user is accessing the site. So there are our two baseline problems.

## What that means in Unity

So, we have a similar problem on our hands with Unity networking, though we have a few other side problems.

The first problem is that, when a user calls a [Command] in the game, the backend
essentially only knows one thing about them: their [network connection].

For us, the second problem is that our Unity backend is making requests to our web
backend on behalf of the user (see the [page on Realms' architecture]({{ site.baseurl }}{% post_url 2016-11-16-game-architecture %})). So there's an intermediary.

A third problem is we need to be able to identify players in the other direction:
given a user id, how do we send that user a network message? If I want to give a player
an item, how do we find that player?

These problems aren't solved out of the box by Unity, so here's how we can approach them.

The first thing we need is a place to store a user's API credentials, that is, their session ID. As I mentioned,
there are two sorts of lookups we need to do. First, we need to translate connections
to user IDs. Secondly, we need to do the opposite, translate user IDs into connections.

Here's a small version of the class in Realms that takes care of this.

{% highlight c# %}
using UnityEngine;
using UnityEngine.Networking;
using System.Collections;
using System.Collections.Generic;

public class GameServerManager : NetworkBehaviour
{

    // Store client information so that we can look it up in both directions
    public Dictionary<int, ClientInformation> userInfoByUserID =
        new Dictionary<int, ClientInformation>();
    public Dictionary<int, ClientInformation> userInfoByConnectionID =
        new Dictionary<int, ClientInformation>();

    public void Start() {
        DontDestroyOnLoad(gameObject);
    }

    // [Server] is a form of defensive programming, and is also
    // self-documenting. It both states that we should never call
    // this client side, and also makes sure that we can't.
    [Server]
    public ClientInformation UserForConnection(NetworkConnection conn)
    {
        return userInfoByConnectionID[conn.connectionId];
    }

    [Server]
    public ClientInformation UserForUserID(int userId)
    {
        return userInfoByUserID[userId];
    }

    [Server]
    public void RegisterClient(
        NetworkConnection conn,
        int userId,
        string jwt,
        PlayerController player
    ) {
        // "jwt" is our version of a Session ID, see below
        ClientInformation info = new ClientInformation(conn, userId, jwt, player);
        userInfoByUserID.Add(userId, info);
        userInfoByConnectionID.Add(conn.connectionId, info);
    }
}

public class ClientInformation
{
    public NetworkConnection Conn;
    public int UserID;
    public string jwt;
    public PlayerController player;

    public ClientInformation(
        NetworkConnection Conn,
        int UserID,
        string jwt,
        PlayerController player
    )
    {
        this.Conn = Conn;
        this.UserID = UserID;
        this.jwt = jwt;
        this.player = player;
    }
}
{% endhighlight %}

This class tracks the information our server needs to know about clients, and
gives us the ability to find them on the server side on any occasion we might need to.

A [JWT] is the version of a session id that Realms uses. I mentioned earlier that
often we store a mapping of session id to client information. JWT takes that a step further,
and just makes the session id *the actual client information*. This happens through
cryptography. Realm's Elixir backend is encoding the user information, encrypting that,
and sending it back. When it receives a JWT, it can decrypt that token and get all
that information without going to another data store. For those curious, I'm using
the [Guardian] library for this, though all commonly used languages have a library
that supports this.

Now that we have that information stored, we can do interesting things. We can easily and securely
make API requests on behalf of any user in the game. Whenever we need to call our Elixir backend from Unity, it ends up looking something like this:

{% highlight c# %}
[Command]
public void CmdLootItem(int itemInstanceID) {
    // We can find the user trying to pickup an item without him making any
    // claims about who he is. This is trusltess.
    var user = gameServerManager.UserForConnection(connectionToClient);
    if (!CanPickupItem(user, itemInstanceId)) {
        Debug.Log("A player tried to pick up an item he isn't allowed to pickup, oh no!")
    }

    var api = new ApiController(user.jwt);
    // I'm using the fantastic UniRx library for virtually all of my API calls,
    // in addition to a bunch of other places.
    // https://github.com/neuecc/UniRx
    api.LootItem(itemInstance).Subscribe(OnPlayerLootItem);
}
{% endhighlight %}

That nearly completes the puzzle. The only remaining part is, what might a login look like?
How exactly should we take their username and password and safely authenticate through the server?
Here's an example of that. (This isn't quite the code in Realms, but it's pretty close).

{% highlight c# %}
[Command]
public void CmdLogin(string username, string password) {
    var connection = connectionToCLient;
    var playerController = this;
    loginController.Login(username, password).Subscribe(response => {
        OnLogin(connection, response, playerController)
    });
}

[ServerCallback]
public void OnLogin(
    NetwornConnection conn,
    LoginResponse response,
    PlayerController player
) {
    gameServerManager.RegisterClient(conn, response.userId, response.jwt, player);
}
{% endhighlight %}

With that, our Unity backend does the task of registering a user's credentials
into our storage.

To recap, here's what we've got:
1. Secure authentication for users.
2. Knowledge of which players are connected to the game.
3. Knowledge of hour to find different players in our game.

And that's pretty much a wrap, an overview of what secure login looks like
for networked games in Unity. Feel free to ping me on channels if I'm lacking
detail anywhere important or you want to hear more.

[Command]: https://docs.unity3d.com/ScriptReference/Networking.CommandAttribute.html
[network connection]: https://docs.unity3d.com/ScriptReference/Networking.NetworkIdentity-connectionToClient.html
[JWT]: https://en.wikipedia.org/wiki/JSON_Web_Token
[Guardian]: https://github.com/ueberauth/guardian

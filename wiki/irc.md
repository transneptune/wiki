# IRC

Some folks have asked me how they can join in the Transneptune IRC channel.
What follows is a guide for those new to IRC.

This guide is still technically accurate, but at this point, we're mostly on
[Slack](https://slack.com/). Ping [Kit](https://twitter.com/wlonk) on Twitter
if you want an invite. Or just use [the automatic signup
form](http://slack.transneptune.net/).

---

IRC is not quite like the chat you know. It's a wild and woolly frontier from
the early days of the internet.

That means:

  * Anyone can run their own IRC network; the first thing you need to know is
    whose server you're connecting to. The next thing is what channel(s) on
    that server you want to join.
  * Identity is not stable in IRC; you need to use an identity service (running
    on the server) if you want to claim and protect an identity.
  * Anyone can write their own IRC client; there are a bazillion to choose
    from, and many can be configured within an inch of their life.

So, for Transneptune, you'll be connecting to Freenode (one of the largest and
best known IRC networks out there) at `ircs://irc.freenode.net:6697`. That
means: the hostname is `irc.freenode.net`, the port is `6697` (the default for
secured IRC) and the protocol is IRCS (IRC:IRCS::HTTP:HTTPS; it's IRC wrapped
in SSL.  So, you might need to make this happen just by checking a box that
says "SSL").

Then, you'll connect to the channel `#transneptunegames`. Except you can't! The
channel is set to only accept users who are registered with IRC's identity
service, `NickServ`. So you've gotta do that first. (A note: usernames in IRC
are referred to as "nicks", short for "nicknames". Traditionally, before
identity services, it was common to change them fluidly, as and where you
needed. Wild!)

So, `NickServ`. Dealing with `NickServ` is like messaging a user, except it's
not a human, of course. So, you can type:

    /msg NickServ help

And that'll start a private conversation with `NickServ` where you begin by
saying "help". At which point, `NickServ` will reply with basic help. READ. IT.
One lesson from IRC is that many of your existing mental categories don't
apply, so you have to take the time to read and learn how the system works.
Sorry.

You're gonna want to then send:

    help register

to learn more about the register command. That should guide you the rest of the
way to reserving your nick.

Once you've registered and identified yourself, you can join
`#transneptunegames`:

    /join #transneptunegames

And then, let's talk!

(One addendum: you can usually configure your IRC client to auto-join certain
channels, and auto-identify with `NickServ`. It's really helpful. My client of
choice is IRCcloud, which also maintains a persistent connection, so I can see
logs of all the conversations that happen when I'm not actively on. But there
are many other ways to skin that cat.)

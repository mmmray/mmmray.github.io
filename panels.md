# Issues with Xray Panels

*2024-07-28*

I've been observing the Xray bugtracker for a while now, and I see some
recurring issues with users of popular Xray panels that are directly caused by
some software design choices in those panels.

I want to write down a list of these choices here, and argue why I think they
cause problems. I don't want to single out any particular panel maintainer
here, mostly because most of the issues are shared by almost all of them, but
also because I hope this document can serve as a reference for the design of
new panels.

## No logs

Once in a while xray gets a bugreport of the form "core crashed". In such
bugreports, the logs are entirely missing or take the form of `Xray exited with
code 2: <one line of output>`.

When the user is then asked to produce more logs so that the crash can be
debugged, they fail at this task entirely. The log output of the panel suddenly
is showing logs from after the restart (instead of before the crash), and the
line of output that was present in the panel's error message is nowhere to be
found.

My recommendation is that panels store as many logs as possible, over multiple
days and across restarts, and give the user a way to dump *all* of them for the
bugreport, and easily. It really doesn't matter to me if this is a gigantic
amount of logfiles for as long as it is complete.

In my mind this is a single button in the UI that collects everything from the
crash that was relevant: logs and config. Leading to my next point:

## No config

Panels build abstractions on top of raw JSON inbound. This leads to the
situation where certain panel users are unable to produce any config for the
bugreport at all, and instead provide screenshots of random panel UIs.

I think that this has led to a situation where users of panels have completely
disengaged with how Xray actually works. When a new feature in Xray is
released, we get questions about "how to do it in Panel XYZ", and when I point
people to XTLS/Xray-examples, they are confused and don't know what to do with it.

It's an unscalable situation for panel maintainers as well: Each document that
Xray produces about some new feature has to be replicated for N panels, or
otherwise end users still use the same methods as from 2 years ago, and wonder
why they get blocked. I also see that as a result of this, oftentimes panels
have better documentation about certain protocols than core itself, and while
my perspective is obviously biased, I think this is a lot of work that could
have benefited core (and therefore, everybody) directly instead.

I think this level of abstraction also leads to a lot of communication problems
between developers, as panels' communities tend to form around different
nationalities and different firewalls than China.

## Bloated config, hardcoding of default values

When the user does manage to get a config out of this entire system and adds it
to the bugreport, it contains a lot of unnecessary things. In the general case,
this is unavoidable as many users don't know how to make minimally reproducible
examples. However, it also seems that many panels take configuration items from
Xray, and hardcode Xray's defaults into the generated config as well.

For example, if there is a setting for TLS `minVersion`, it seems that some
panels have hardcoded 1.2 as a default value and will always produce
`minVersion`, even though it is completely unnecessary and was never explicitly
set by the user.

when SplitHTTP was released I noticed this for the first time. This transport
has config options `maxUploadSize` and `maxConcurrentUploads`, and the defaults
from the 1.8.16 release are copypasted into N panels now. That's too bad,
because those default values are going to change in ways that would benefit
performance drastically, but now that these panels have hardcoded them
everywhere, it remains to be seen who will actually notice the performance
improvement. I think it is also very likely that over time, the defaults
between panel and xray will change so drastically that they actually break
connectivity, because those parameters have to be kept in sync with the client
to some extent.

And of course, all of these generated configs are just really hard to read.

I suggest that when you write a GUI for inbounds, you don't make any
implications around defaults, as far as it is possible, and don't produce
things that were not explicitly set by the user. Really, any checkbox should
have three states instead of two: `True`, `False`, and `Not Set`, a.k.a "let
xray-core decide the default".

If I suggested that xray panels all work like
[Marzneshin](https://github.com/khodedawsh/marzneshin) and make the user write
JSON directly, how many people would object? Probably all of them, but I really
think this is something that particular panel got right, there is no GUI for
writing inbounds, and as such those config related issues do not exist at all
there. Neither this issue nor the previous one.

## Tension around sharelinks and what they should support

When fragment was released, RPRX was insisting that this functionality is not
suitable for sharing, and so did not add it to sharelinks. Fragment remains
wildly popular in Iran despite his words, and because the iranian GFW is
unpredictable as ever, it still delivers value against all intuition.

So now, panels have implemented workarounds to distribute configs with fragment
without using the standard sharelink format at all.

The most popular panels in Iran have added subscription URLs with custom
JSON configs, meaning that JSON outbounds get directly distributed to clients.
I think over time this will become a massive compatibility hazard, and make it
even harder for Xray's configuration format to change in a way that does not
break users.

I recommend that panels always keep a way to only produce regular sharelinks,
as I expect that this system will eventually blow up in everybody's faces.

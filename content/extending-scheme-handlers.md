+++
title = "Extending XDG URI scheme handlers"
date = 2023-05-17

[taxonomies]
tags = ["linux"]
+++

# Introduction

Like with any sane operating system, in Linux it is possible to set default applications for specific URI schemes, for example the Spotify desktop file contains the following line:

```conf
MimeType=x-scheme-handler/spotify;
```

This effectively makes Spotify the default application to be opened when the user clicks on any `spotify://` link. If there are ever multiple applications capable of handling the same scheme, then the user can select a preference e.g. with:

```bash
xdg-mime default spotify.desktop 'x-scheme-handler/spotify'
```

But usually when people share Spotify links, they don't share weird `spotify://` URIs, they share a link using `https://open.spotify.com`, which means that this completely bypasses the spotify scheme handler and uses the default web browser instead of playing the track directly in Spotify.

We obviously can't just set Spotify as the default `https` scheme handler, so how would we open just these specific `https` links in Spotify directly?
It becomes obvious that the `xdg-mime` API is not powerful enough for this.

In Android land there already is [quite comprehensive API](https://developer.android.com/guide/topics/manifest/data-element#path) for this usecase. An APK can set the `<data android:scheme/>` tag in its app manifest file to register as a scheme handler and there are several options to fine-tune the scheme matching.
For example, a regular expression can be used with `android:pathAdvancedPattern` to handle only matching URIs as opposed to handling all URIs for that scheme.

# Proxy scheme handler

Linux does not have something comparable to `android:pathAdvancedPattern`, but until there is a proper XDG spec for this we can achieve something very similar by registering a small proxy application as the default web browser. This proxy application would use some predefined and configurable rules to match all opened URLs against a regular expression. If there is a match, the request is delegated to the corresponding application.
Otherwise if no rules match, then the link is opened in the original default web browser.

I have written a small application just like this called [sphere](https://github.com/vimpostor/sphere) (realistically you could also just redirect with a Greasemonkey script in the browser), which parses the rules as lines from a configuration file in `~/.config/sphere/config`. The first field of each line is the regular expression to match against and the last field is the corresponding scheme handler.

The following simple example opens Spotify links directly in the Spotify desktop application and configures Chromium as the default web browser otherwise:

```
^https://open.spotify.com/(artist|album|track|search)/[[:alnum:]]+\??.*$ spotify-launcher
.* chromium
```

Now all that remains is configuring this proxy application as the default web browser.

```bash
xdg-settings set default-web-browser sphere.desktop
```

Here it is in action:

{{ video(url="https://github.com/vimpostor/sphere/assets/21310755/2895bf5d-7dea-4cab-862d-5accdbbd4697") }}

Notice how the Spotify link is opened directly in the correct application instead of going the detour over the web browser?
Obviously for this to work, the Spotify desktop application needs to support direct links, which it luckily does. It also can support a lot more formats, as we will see later.

You might be asking yourself "What do you mean later?", this blog post is done, we have achieved what we needed, what else can this guy still ramble on? Well, this is the part where _scratching your own itch_ turns into [shaving an entire yak](https://projects.csail.mit.edu/gsb/old-archive/gsb-archive/gsb2000-02-11.html).

# Single Spotify instance

With the current implementation, we would open a new Spotify instance every time we click on a Spotify link, even if we already have a Spotify instance open.
But this is ridiculous, who would want to use a music player like this. Just play the song in the already running Spotify instance, goddammit!

As luck has it, Spotify exposes a dbus interface and we can use the `org.mpris.MediaPlayer2.Player.OpenUri` method to start a new track in an already running instance:

```bash
dbus-send --type=method_call --dest=org.mpris.MediaPlayer2.spotify /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player.OpenUri string:'spotify:track:5GJaxfibCRCQXDVPgHJv0s'
```

This can also be used to open albums, artist pages, playlists and even starting a search.
This dbus interface does not support the normal `https` URLs though. Instead we have to translate all possible URIs to the internal Spotify format `spotify:<type>:<id>`.

If you have read any of my past blog posts, you probably expect me to drop a huge shell one-liner that should have been a python script any second now.
And yes, that is exactly what is going to happen, you could indeed edit Spotify's desktop file to execute the following abomination of shell script (you shouldn't though):

```bash
Exec=env U=%u sh -c "pidof spotify && dbus-send --type=method_call --dest=org.mpris.MediaPlayer2.spotify /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player.OpenUri string:\\$(echo \\$U| sed 's/\\(https:\\/\\/open.spotify.com\\/\\|spotify:\\/\\/\\)\\(.*\\)\\/\\([[:alnum:]]*\\).*/spotify:\\2:\\3/'| sed 's/^\\(spotify:[a-z]*:[[:alnum:]]*\\)?.*/\\1/') || spotify --uri=\\$U"
```

Seriously though, don't use this, this is cursed beyond limits and doesn't handle some edge cases correctly.

Instead I implemented a much cleaner and robust solution: There is a [Spotify launcher](https://github.com/kpcyrd/spotify-launcher) application written in Rust, that is basically the official Spotify desktop application but with automatic updates added. So I went ahead and [implemented single instance support in the launcher directly](https://github.com/kpcyrd/spotify-launcher/pull/23), removing the need for any shell shenanigans. With this patch, opening Spotify links finally works like you would expect:

{{ video(url="https://github.com/vimpostor/blog/assets/21310755/5443f4d4-aec2-401f-971d-e8b73c2a1788") }}

The track is played in the already running Spotify instance instead of opening a new one.
Note that for now, you will have to [wait until the PR is merged](https://github.com/kpcyrd/spotify-launcher/pull/23#issuecomment-1522137173) or apply the patch manually.

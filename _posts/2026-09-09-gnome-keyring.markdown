---
layout: post
title:  When software does almost what you want
date:   2026-09-09
---

I want my backup script to fetch the required password from `gnome-keyring`.
This won't take long because it's not a difficult task.

I use the [Seahorse](https://gitlab.gnome.org/GNOME/seahorse) GUI to first
create a new keyring and then add the password required by the backup script to
that keyring. Why a new keyring? Each keyring is protected by its own password
and can be separately locked/unlocked. By putting the password into its own
keyring I can keep it locked most of the time and only unlock it when needed. I
just don't like the idea that any app can just read all the passwords of my
unlocked keyrings all the time, so I try to keep the time window small. Does
this increase security?  Not much, any malicious program running as with my
permissions can already do enough damage, but it makes me feel better. I want to
feel better.

Once the password is created, I use `secret-tool` (provided by
[`libsecret`](https://gnome.pages.gitlab.gnome.org/libsecret/)) to access the
password from my Bash backup script. According to `secret-tool -h`, looking up a
password works like this:

```sh
secret-tool lookup attribute value ...
```

This will display the first password with matching attribute-value pairs.

First obstacle: Even though Seahorse can display the attributes of a password,
there is no way to add or edit them. The only thing you can do is modify the
description or the password itself. Unfortunately, `secret-tool` cannot lookup a
password by its description, you *have to* provide a matching attribute-value
pair. Thus, it seems there is no way to use `secret-tool` to lookup a password
created with Seahorse.

But that's not a big problem, we can just use `secret-tool store` to create the
password:

```sh
secret-tool store --label='label' attribute value ...
```

This creates a new password (read from stdin) with the provided description (aka
label) and attribute-value pairs.

Minor obstacle: `secret-tool store` stores the password in the default keyring
and there is no documented way to override this. However, looking at the source
code reveals that it accepts a `--collection` argument that can be used to
specify the desired keyring.

**UPDATE:** As of version 0.21.8 (released 2026-09-12), the `--collection`
argument is now documented.

So let's try this:

```sh
secret-tool store --collection='Temp Access' --label 'Borg' app Borg
```

This creates a password labeled "Borg" (which is the backup tool I use) in the
keyring named "Temp Access". Also, the password has an attribute `app` with
value `Borg` so that I can easily look it up with `secret-tool lookup`.

Next obstacle: The above command actually results in an error saying that the
argument given to `--collection` must be a full path. A full path to what? At
first I thought I need to provide the full *filesystem path* to the keyring, so
I tried `~/.local/share/keyrings/Temp_Access.keyring` but that still didn't
work. Turns out that `secret-tool` is basically a wrapper around the DBus
interface of `gnome-keyring`, and the collection name must be a DBus *object
path* that uniquely identifies the keyring. So how do we get that object path?

After reading about the [`gnome-keyring` DBus interface][gnome-keyring-dbus] and
the `dbus-send` utility for sending DBus messages I came up with the following
command to list the object paths of all available keyrings:

```sh
dbus-send --dest=org.freedesktop.secrets --type=method_call --print-reply \
    /org/freedesktop/secrets \
    org.freedesktop.DBus.Properties.Get \
    string:org.freedesktop.Secret.Service \
    string:Collections
```

[gnome-keyring-dbus]: https://gitlab.gnome.org/GNOME/gnome-keyring/-/blob/main/daemon/dbus/org.freedesktop.Secrets.xml

Turns out, the object path for "Temp Access" is
`/org/freedesktop/secrets/collection/Temp_5fAccess`. Thus, I can finally add my
Borg password:

```sh
secret-tool store \
    --collection='/org/freedesktop/secrets/collection/Temp_5fAccess' \
    --label 'Borg' \
    app Borg
```

Lookup searches all keyrings so no `--collection` argument needed:

```sh
secret-tool lookup app Borg
```

And that's it. I really wish Seahorse would allow me to edit the attributes of a
password (related [issue][seahorse-issue]). And I really wish `secret-tool
--collection` would accept the display name of the keyring and not its DBus
object path ([issue][libsecret-issue]). As of now, they both do *almost* what I
want.

[seahorse-issue]: https://gitlab.gnome.org/GNOME/seahorse/-/work_items/367
[libsecret-issue]: https://gitlab.gnome.org/GNOME/libsecret/-/work_items/112

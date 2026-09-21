---
layout: post
title:  Accessing gnome-keyring
date:   2026-09-20
---

Managing passwords stored in `gnome-keyring` with Seahorse and `secret-tool`
is so frustrating:
- Seahorse does not allow to attach any additional key-value pairs to
  passwords
- `secret-tool`
  - cannot query passwords *without* providing at least one key-value pair
  - cannot query passwords based on the description that you can edit via
    Seahorse

If you want to query a password with `secret-tool`, you also have to create it
with `secret-tool` in order to associate key-value pairs with the password. By
default, the newly created password will be added to the default keyring. You
can also add it to a different keyring (using the undocumented `--collection`
flag for the `secret-tool store` subcommand). However, you have to provide the
DBus object name of the keyring, so if the name of your keyring contains e.g.
a space, then you need to know how that is represented in the object name.
(Keyrings are stored in `~/.local/share/keyrings/`, but using the filename
directly may not always work).

Fortunately, `libsecret` (on which `secret-tool` is build) is just a wrapper
around the DBus interface of `gnome-keyring` so you can just talk to
`gnome-keyring` directly. Its DBus interface is defined here:
https://gitlab.gnome.org/GNOME/gnome-keyring/-/blob/main/daemon/dbus/org.freedesktop.Secrets.xml

```bash
# Query all keyrings
dbus-send --dest=org.freedesktop.secrets --type=method_call --print-reply \
    /org/freedesktop/secrets \
    org.freedesktop.DBus.Properties.Get \
    string:org.freedesktop.Secret.Service \
    string:Collections

# Query the passwords of the default keyring
dbus-send --dest=org.freedesktop.secrets --type=method_call --print-reply \
    /org/freedesktop/secrets/collection/Default_5fkeyring \
    org.freedesktop.DBus.Properties.Get \
    string:org.freedesktop.Secret.Collection \
    string:Items

# Locking keyring "Temp Access"
dbus-send --dest=org.freedesktop.secrets --type=method_call \
    /org/freedesktop/secrets \
    org.freedesktop.Secret.Service.Lock \
    array:objpath:/org/freedesktop/secrets/collection/Temp_5fAccess
```

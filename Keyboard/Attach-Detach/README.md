The ROG Zephyrus Duo GX651AR reports WMI event 0x5e when the
keyboard is attached or detached. The generic ASUS WMI keymap
maps this event to KEY_WLAN.

Add a model-specific quirk to ignore only event 0x5e on the
GX651AR, preserving existing handling of events 0x5d and 0x5f
and leaving other ASUS models unchanged.

Tested with AI on ASUS ROG Zephyrus Duo GX651AR running Fedora 44 kernel
7.2.11-200.fc44.x86_64, using a temporary module built from the
v7.2.11 driver source with the KEYBOARD_KEY_5e udev override
disabled. Keyboard detachment, reattachment, and typing continued
to work. Wi-Fi remained enabled, and evtest no longer reported
scan code 0x5e.


Found that the firmware does two things when the keyboard is attached and detached one is to enable and disable the second display and the second is for orientation prompt.

As I use Fedora 44 KDE Plasma I and other ASUS dual screen laptop users choose to disable the orientation prompt as the kde interface can already control orientation automatically.

https://gist.github.com/mfenniak/f313f9a94fbcfa8fb52f978f29393ab1



commit text rough draft

Ignore keyboard attach/detach event on GX651AR

The ASUS ROG Zephyrus Duo GX651AR has a detachable keyboard. Attaching or
detaching it generates WMI event 0x5e, which the keymap reports as
KEY_WLAN and Wi-Fi gets toggled. This command is orignally made to pop up on screen what orientation thats going to be used.

KDE Plasma has Automatic orientation that already does this and some pending patches for finetuning. 

The existing Zenbook Duo keyboard quirk ignores 0x5d, 0x5e and 0x5f, but
those other codes are used for real keys on some ASUS models. This adds a
narrower quirk that ignores only 0x5e on this model.

Signed-off-by: Cymirk <Cymirk@icloud.com>

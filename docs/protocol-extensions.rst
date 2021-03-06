Extensions to the xterm protocol
===================================

|kitty| has a few extensions to the xterm protocol, to enable advanced features.
These are typically in the form of new or re-purposed escape codes. While these
extensions are currently |kitty| specific, it would be nice to get some of them
adopted more broadly, to push the state of terminal emulators forward.

The goal of these extensions is to be as small and unobtrusive as possible,
while filling in some gaps in the existing xterm protocol. In particular, one
of the goals of this specification is explicitly not to "re-imagine" the tty.
The tty should remain what it is -- a device for efficiently processing text
received as a simple byte stream. Another objective is to only move the minimum
possible amount of extra functionality into the terminal program itself. This
is to make it as easy to implement these protocol extensions as possible,
thereby hopefully encouraging their widespread adoption.

If you wish to discuss these extensions, propose additions/changes to them
please do so by opening issues in the github bug tracker.

.. contents::

Colored and styled underlines
-------------------------------

|kitty| supports colored and styled (wavy) underlines. This is of particular
use in terminal editors such as vim and emacs to display red, wavy underlines
under mis-spelled words and/or syntax errors. This is done by re-purposing some
SGR escape codes that are not used in modern terminals (`CSI codes
<https://en.wikipedia.org/wiki/ANSI_escape_code#CSI_sequences>`_)

To set the underline style::

    <ESC>[4:0m  # this is no underline
    <ESC>[4:1m  # this is a straight underline
    <ESC>[4:2m  # this is a double underline
    <ESC>[4:3m  # this is a curly underline
    <ESC>[4:4m  # this is a dotted underline (not implemented in kitty)
    <ESC>[4:5m  # this is a dashed underline (not implemented in kitty)
    <ESC>[4m    # this is a straight underline (for backwards compat)
    <ESC>[24m   # this is no underline (for backwards compat)

To set the underline color (this is reserved and as far as I can tell not actually used for anything)::

    <ESC>[58...m

This works exactly like the codes ``38, 48`` that are used to set foreground and
background color respectively.

To reset the underline color (also previously reserved and unused)::

    <ESC>[59m

The underline color must remain the same under reverse video, if it has a
color, if not, it should follow the foreground color.

To detect support for this feature in a terminal emulator, query the terminfo database
for the ``Su`` boolean capability.

Graphics rendering
---------------------

See :doc:`/graphics-protocol` for a description
of this protocol to enable drawing of arbitrary raster images in the terminal.


.. _extended-key-protocol:

Keyboard handling
-------------------

There are various problems with the current state of keyboard handling. They
include:

* No way to use modifiers other than ``Ctrl`` and ``Alt``

* No way to reliably use multiple modifier keys, other than, ``Shift+Alt``.

* No way to handle different types of keyboard events, such as press, release or repeat

* No reliable way to distinguish single ``Esc`` keypresses from the start of a
  escape sequence. Currently, client programs use fragile timing related hacks
  for this, leading to bugs, for example:
  `neovim #2035 <https://github.com/neovim/neovim/issues/2035>`_.

There are already two distinct keyboard handling modes, *normal mode* and
*application mode*. These modes generate different escape sequences for the
various special keys (arrow keys, function keys, home/end etc.) Most terminals
start out in normal mode, however, most shell programs like ``bash`` switch them to
application mode. We propose adding a third mode, named *full mode* that addresses
the shortcomings listed above.

Switching to the new *full mode* is accomplished using the standard private
mode DECSET escape sequence::

    <ESC>[?2017h

and to leave *full mode*, use DECRST::

    <ESC>[?2017l

The number ``2017`` above is not used for any existing modes, as far as I know.
Client programs can query if the terminal emulator is in *full mode* by using
the standard `DECRQM <https://vt100.net/docs/vt510-rm/DECRQM.html>`_ escape sequence.

The new mode works as follows:

  * All printable key presses without modifier keys are sent just as in the
    *normal mode*. This means all printable ASCII characters and in addition,
    ``Enter``, ``Space`` and ``Backspace``. Also any unicode characters generated by
    platform specific extended input modes, such as using the ``AltGr`` key. This
    is done so that client programs that are not aware of this mode can still
    handle basic text entry, so if a *full mode* using program crashes and does
    not reset, the user can still issue a ``reset`` command in the shell to restore
    normal key handling. Note that this includes pressing the ``Shift`` modifier
    and printable keys. Note that this means there are no repeat and release
    events for these keys and also for the left and right shift keys.

  * For non printable keys and key combinations including one or more modifiers,
    an escape sequence encoding the key event is sent. For details on the
    escape sequence, see below.

The escape sequence encodes the following properties:

  * Type of event: ``press,repeat,release``
  * Modifiers pressed at the time of the event
  * The actual key being pressed

Schematically::

    <ESC>_K<type><modifiers><key><ESC>\

Where ``<type>`` is one of ``p`` -- press, ``r`` -- release and ``t`` -- repeat.
Modifiers is a bitmask represented as a single base64 digit.  Shift -- ``0x1``,
Alt -- ``0x2``, Control -- ``0x4`` and Super -- ``0x8``.  ``<key>`` is a number
(encoded in base85) corresponding to the key pressed. The key name to number
mapping is defined in :doc:`this table <key-encoding>`.

Client programs must ignore events for keys they do not know. The mapping in
the above table is stable and will never change, however, new codes might be
added to it in the future, for new keys.

For example::

    <ESC>_KpGp<ESC>\  is  <Ctrl>+<Alt>+x (press)
    <ESC>_KrP8<ESC>\  is  <Ctrl>+<Alt>+<Shift>+<Super>+PageUp (release)

This encoding means each key event is represented by 8 or 9 printable ascii
only bytes, for maximum robustness.

To see the full mode in action, run::

   kitty +kitten key_demo

Support for this mode is indicated by the ``fullkbd`` boolean capability
in the terminfo database, in case querying for it via DECQRM is inconvenient.

.. _ext_styles:

Setting text styles/colors in arbitrary regions of the screen
------------------------------------------------------------------

There already exists an escape code to set *some* text attributes in arbitrary
regions of the screen, `DECCARA
<https://vt100.net/docs/vt510-rm/DECCARA.html>`_.  However, it is limited to
only a few attributes. |kitty| extends this to work with *all* SGR attributes.
So, for example, this can be used to set the background color in an arbitrary
region of the screen.

The motivation for this extension is the various problems with the existing
solution for erasing to background color, namely the *background color erase
(bce)* capability. See
`this discussion <https://github.com/kovidgoyal/kitty/issues/160#issuecomment-346470545>`_
and `this FAQ <https://invisible-island.net/ncurses/ncurses.faq.html#bce_mismatches>`_
for a summary of problems with *bce*.

For example, to set the background color to blue in a
rectangular region of the screen from (3, 4) to (10, 11), you use::

    <ESC>[2*x<ESC>[4;3;11;10;44$r<ESC>[*x


Saving and restoring the default foreground/background/selection/cursor colors
---------------------------------------------------------------------------------

It is often useful for a full screen application with its own color themes
to set the default foreground, background, selection and cursor colors. This
allows for various performance optimizations when drawing the screen. The
problem is that if the user previously used the escape codes to change these
colors herself, then running the full screen application will lose her
changes even after it exits. To avoid this, kitty introduces a new pair of
*OSC* escape codes to push and pop the current color values from a stack::

    <ESC>]30001<ESC>\  # push onto stack
    <ESC>]30101<ESC>\  # pop from stack

These escape codes save/restore the so called *dynamic colors*, default
background, default foreground, selection background, selection foreground and
cursor color.


Pasting to clipboard
----------------------

|kitty| implements the OSC 52 escape code protocol to get/set the clipboard
contents (controlled via the :opt:`clipboard_control` setting). There is one
difference in kitty's implementation compared to some other terminal emulators.
|kitty| allows sending arbitrary amounts of text to the clipboard. It does so
by modifying the protocol slightly. Successive OSC 52 escape codes to set the
clipboard will concatenate, so::

    <ESC>]52;c;<payload1><ESC>\
    <ESC>]52;c;<payload2><ESC>\

will result in the clipboard having the contents ``payload1 + payload2``. To
send a new string to the clipboard send an OSC 52 sequence with an invalid payload
first, for example::

    <ESC>]52;c;!<ESC>\

Here ``!`` is not valid base64 encoded text, so it clears the clipboard.
Further, since it is invalid, it should be ignored by terminal emulators
that do not support this extension, thereby making it safe to use, simply
always send it before starting a new OSC 52 paste, even if you aren't chunking
up large pastes, that way kitty won't concatenate your paste, and it will have
no ill-effects in other terminal emulators.

In case you're using software that can't be easily adapted to this
protocol extension, it can be disabled by specifying ``no-append`` to the
:opt:`clipboard_control` setting.

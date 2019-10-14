# Terminal Colors

There exists common confusion about terminal colors. This is what we have right now:

- Plain ASCII
- ANSI escape codes: 16 color codes with bold/italic and background
- 256 color palette: 216 colors + 16 ANSI + 24 gray (colors are 24-bit)
- 24-bit true color: "888" colors (aka 16 milion)

```bash
printf "\x1b[${bg};2;${red};${green};${blue}m\n"
```

The 256-color palette is configured at start and is a 666-cube of colors,
each of them defined as a 24-bit (888 rgb) color.

This means that current support can only display 256 different colors in the
terminal while "true color" means that you can display 16 million different
colors at the same time.

Truecolor escape codes do not use a color palette. They just specify the
color itself.

This is a good test case:

```bash
printf "\x1b[38;2;255;100;0mTRUECOLOR\x1b[0m\n"
```

- or
  https://raw.githubusercontent.com/JohnMorales/dotfiles/master/colors/24-bit-color.sh
- or http://github.com/robertknight/konsole/tree/master/tests/color-spaces.pl
- or https://git.gnome.org/browse/vte/tree/perf/img.sh
- or just run this:

```sh
awk 'BEGIN{
    s="/\\/\\/\\/\\/\\"; s=s s s s s s s s;
    for (colnum = 0; colnum<77; colnum++) {
        r = 255-(colnum*255/76);
        g = (colnum*510/76);
        b = (colnum*255/76);
        if (g>255) g = 510-g;
        printf "\033[48;2;%d;%d;%dm", r,g,b;
        printf "\033[38;2;%d;%d;%dm", 255-r,255-g,255-b;
        printf "%s\033[0m", substr(s,colnum+1,1);
    }
    printf "\n";
}'
```

Keep in mind that it is possible to use both ';' and ':' as Control Sequence
Introducer delimiters.

According to Wikipedia[1], this behavior is only supported by xterm and konsole.

[1] https://en.wikipedia.org/wiki/ANSI_color

Since
[ncurses-6.0-20180121](http://lists.gnu.org/archive/html/bug-ncurses/2018-01/msg00045.html),
terminfo began to support the 24-bit True Color capability under the name of
"RGB". You need to use the "setaf" and "setab" commands to set the foreground
and background respectively.

# True Color Detection

There will be no reliable way to detect the "RGB" flag until the new release of
terminfo/ncurses. S-Lang author added a check for $COLORTERM containing either
"truecolor" or "24bit" (case sensitive). In addition,
[VTE](https://bugzilla.gnome.org/show_bug.cgi?id=754521),
[Konsole](https://bugs.kde.org/show_bug.cgi?id=371919) and
[iTerm2](https://gitlab.com/gnachman/iterm2/issues/5294) set this variable to
"truecolor". It has been in VTE for a while and but is relatively new, being
still git-only in Konsole and iTerm2).

This is obviously not a reliable method, and is not forwarded via sudo, SSH etc.
However, whenever it errs, it errs on the safe side. It does not advertise
support when it is actually supported. App developers can freely choose to
check for this same variable, or introduce their own method (e.g. an option in
their config file). They should use whichever method best matches the overall
design of their app. Checking $COLORTERM is recommended though since it will
lead to a more seamless desktop experience where only one variable needs to be
set. This would be system-wide so that the user would not need to set it
separately for each app.

## Querying The Terminal

A more reliable method in an interactive program which can read terminal
responses, and one that is transparent to things like sudo, SSH, etc.. is to
simply try setting a truecolor value and then query the terminal to ask what
color it currently has. If the response replies the same color that was set
then it indicates truecolor is supported.

```bash
$ (echo -e '\e[48:2:1:2:3m\eP$qm\e\\' ; xxd)

^[P1$r48:2:1:2:3m^[\
00000000: 1b50 3124 7234 383a 323a 313a 323a 336d  .P1$r48:2:1:2:3m
```

Here we ask to set the background color to `RGB(1,2,3)` - an unlikely default
choice - and request the value that we just set. The response comes back that
the request was understood (`1`), and that the color is indeed `48:2:1:2:3`.
This tells us also that the terminal supports the colon delimiter. If instead,
the terminal did not support truecolor we might see a response like

```
^[P1$r40m^[\
00000000: 1b50 3124 7234 306d 1b5c 0a              .P1$r40m.\.
```

This terminal replied the color is `40` - it has not accepted our request to
set `48:2:1:2:3`.

```
^[P0$r^[\
00000000: 1b50 3024 721b 5c0a                      .P0$r.\.
```

This terminal did not even understand the DECRQSS request - its response was
`0$r`. We do not learn if it managed to set the color, but since it doesn't
understand how to reply to our request it is unlikely to support truecolor
either.

# Terminals + True Color

## Now **Supporting** True Color

- [st](http://st.suckless.org/) (from suckless) [delimeter: semicolon] -
  http://lists.suckless.org/dev/1307/16688.html
- [xst](https://github.com/gnotclub/xst/) - fork of st
- [konsole](http://kde.org/applications/system/konsole/) [delimeter: colon,
  semicolon] - https://bugs.kde.org/show_bug.cgi?id=107487
- [iTerm2](http://www.iterm2.com/) [delimeter: colon, semicolon] - since v3
  version
- [Therm](https://github.com/trufae/Therm) [delimeter: colon, semicolon] - fork
  of iTerm2
- [qterminal](https://github.com/lxqt/qterminal) [delimeter: semicolon] -
  https://github.com/qterminal/qterminal/issues/78
- [alacritty](https://github.com/jwilm/alacritty) [delimeter: semicolon] -
  written in Rust
- [kitty](https://github.com/kovidgoyal/kitty) [delimeter: colon,semicolon] -
  uses OpenGL
- [cool-retro-term](https://github.com/Swordfish90/cool-retro-term) [delimeter:
  semicolon]
- [mosh](https://mosh.org/) (Mobile SHell) [delimeter: semicolon] - since commit
  https://github.com/mobile-shell/mosh/commit/6cfa4aef598146cfbde7f7a4a83438c3769a2835
- [pangoterm](http://www.leonerd.org.uk/code/pangoterm/) [delimeter:
  colon, semicolon]
- [Termux](https://termux.com/) [delimeter: semicolon] - **Android platform**
- [ConnectBot](https://connectbot.org/) - **Android platform** - since
  https://github.com/connectbot/connectbot/commit/3bcc75ccedaf2136b04c5932c81a5155f29dc3b5
- [Black Screen](https://github.com/shockone/black-screen) [delimeter:
  semicolon] - crossplatform, HTML/CSS/JS-based
- [hterm](https://chromium.googlesource.com/apps/libapps/+/master/hterm) -
  HTML/CSS/JS-based (ChromeOS)
- [PuTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty/) -
  [landed](https://git.tartarus.org/?p=simon/putty.git;a=commit;h=a4cbd3dfdb71d258e83bbf5b03a874c06d0b3106)
  in git (patched version [3] {xterm-like approximation to 256 colors} and [4]
  {real true colors} available) - **Windows platform**
- [Tera Term](http://en.sourceforge.jp/projects/ttssh2/) [delimeter: colon,
  semicolon] - **Windows platform**
- [ConEmu](https://github.com/Maximus5/ConEmu) [delimeter: semicolon] -
  **Windows platform**
- [Windows
  Powershell](https://en.wikipedia.org/wiki/PowerShell#PowerShell_5.1)
  [delimeter: semicolon] - aka Powershell 5.x and below **Windows 10**
- [Powershell Core](https://github.com/PowerShell/PowerShell) [delimeter:
  semicolon] aka Powershell 6+ **Windows 10**
- [cmd.exe](https://en.wikipedia.org/wiki/Cmd.exe) [delimeter:
  semicolon] Builtin Windows shell that is mostly unchanged since DOS **Windows 10**
- [FinalTerm](http://finalterm.org/) [delimeter: semicolon] -
  **[abandoned](http://worldwidemann.com/finally-terminated/)**, iTerm2
  [borrowing it's ideas and features](http://iterm2.com/shell_integration.html).
- [MacTerm](https://github.com/kmgrant/macterm) [delimeter: semicolon] - **Mac
  OS X platform**
- [mintty](https://mintty.github.io/) [delimeter: semicolon] **Cygwin and
  MSYS/MSYS2** since commit
  https://github.com/mintty/mintty/commit/43f0ed8a46c6549cb9a3ea27abc057b5abe13bdb
  (2.0.1 release) - **Windows platform**
- [MobaXterm](http://mobaxterm.mobatek.net/) **Windows platform** - closed
  source (run `lscolors` to see a truecolor test)
- [ZOC](https://www.emtec.com/zoc/index.html) **Windows/OS X platform** - closed
  source since
  [7.19.0 version](http://www.emtec.com/downloads/zoc/zoc_changes.txt)
- [upterm](https://github.com/railsware/upterm) *Windows/Macos/Linux Electron* -
  A terminal emulator for the 21st century.
- Windows 10 bash console, since
  [Windows Insiders build 14931](https://blogs.msdn.microsoft.com/commandline/2016/09/22/24-bit-color-in-the-windows-console/)
- all [libvte](http://ftp.gnome.org/pub/GNOME/sources/vte/) based terminals
  (since 0.36 version) [delimeter: colon, semicolon] -
  https://bugzilla.gnome.org/show_bug.cgi?id=704449
  - **libvte**-based
    [Gnome Terminal](https://help.gnome.org/users/gnome-terminal/stable/)
  - **libvte**-based [sakura](http://www.pleyades.net/david/projects/sakura)
  - **libvte**-based
    [xfce4-terminal](http://docs.xfce.org/apps/terminal/start) - since
    [0.6.90](https://github.com/xfce-mirror/xfce4-terminal/releases/tag/xfce4-terminal-0.6.90)
    release, if compiled with GTK+3
  - **libvte**-based
    [Terminator](http://gnometerminator.blogspot.com/p/introduction.html) -
    since [1.90](https://launchpad.net/terminator/+announcement/14358) release
  - **libvte**-based [Tilix](https://github.com/gnunn1/tilix) - written in D.
    Similar user interface as for Terminator.
  - **libvte**-based [Lilyterm](http://lilyterm.luna.com.tw/) - since commit
    https://github.com/Tetralet/LilyTerm/commit/72536e7ba448ad9ef1126ce45fbde3a3407a271b
  - **libvte**-based [ROXTerm](http://roxterm.sourceforge.net/)
  - **libvte**-based [evilvte](http://www.calno.com/evilvte/) - no release yet,
    version from git https://github.com/caleb-/evilvte
  - **libvte**-based [Termit](https://github.com/nonstop/termit)
  - **libvte**-based [Termite](https://github.com/thestinger/termite)
  - **libvte**-based [Tilda](https://github.com/lanoxx/tilda)
  - **libvte**-based [tinyterm](https://code.google.com/p/tinyterm)
  - **libvte**-based
    [Pantheon Terminal](https://launchpad.net/pantheon-terminal)
  - **libvte**-based [lxterminal](http://sourceforge.net/projects/lxde) - with
    **--enable-gtk3** configure flag.
  - **libvte**-based [guake](http://guake-project.org/) - A top-down terminal for GNOME

There are a bunch of libvte-based terminals for GTK2, so they are listed in the
another section.

Also, while this one is not a terminal, but a terminal replayer, it is
still worth mentioning:

- [asciinema](http://asciinema.org) player:
  https://github.com/asciinema/asciinema-player

## Improper Support for True Color

- [mlterm](http://mlterm.sourceforge.net) - built with **--with-gtk=3.0**
  configure flag. Approximates colors to 512 embedded palette
  (https://sourceforge.net/p/mlterm/bugs/74/)

## Terminals that parse ANSI color sequences, but approximate them to 256 palette

- xterm (but doing it wrong: "it uses nearest color in RGB color space,
  with a usual false assumption about orthogonal axes")
- [urxvt aka rxvt-unicode](http://software.schmorp.de/pkg/rxvt-unicode.html) -
  since
  [Revision 1.570](http://cvs.schmorp.de/rxvt-unicode/src/command.C?revision=1.570&view=markup&sortby=log&sortdir=down)
  http://lists.schmorp.de/pipermail/rxvt-unicode/2016q2/002261.html (Note there
  is a restriction of colors count still)
- linux console (since v3.16):
  https://github.com/torvalds/linux/commit/cec5b2a97a11ade56a701e83044d0a2a984c67b4

Note about color differences:
a) RGB axes are not orthogonal, so you cannot use
sqrt(R^2+G^2+B^2) formula
b) for color differences there is more correct (but
much more complex)
[CIEDE2000](http://en.wikipedia.org/wiki/Color_difference#CIEDE2000) formula
(which may easily blow up performance if used blindly) [2].

[2] https://github.com/neovim/neovim/issues/793#issuecomment-48106948

## Terminal multiplexers

- [tmux](http://tmux.github.io/) - starting from version 2.2 (support since
  [427b820...](https://github.com/tmux/tmux/commit/427b8204268af5548d09b830e101c59daa095df9))
- [screen](http://git.savannah.gnu.org/cgit/screen.git/) - has support in
  'master' branch, need to be enabled (see 'truecolor' option)
- [pymux](https://github.com/jonathanslenders/pymux) - tmux clone in pure Python
  (to enable truecolor run pymux with `--truecolor` option)
- [dvtm](https://github.com/martanne/dvtm) - not yet supporting True Color
  https://github.com/martanne/dvtm/issues/10

## **NOT Supporting** True Color

- [Terminal.app](https://en.wikipedia.org/wiki/Terminal_(macOS)): Macos Terminal builtin
- [Terminology](https://www.enlightenment.org/about-terminology)
  (Enlightenment) - https://phab.enlightenment.org/T746
- [Hyper.app](https://hyper.is/) [delimeter: semicolon] - crossplatform,
  HTML/CSS/JS-based (Electron) https://github.com/zeit/hyper/issues/2294
- [Cmder](https://cmder.net/): Portable console emulator for Windows,
  based on ConEmu.
- [Terminus](https://github.com/Eugeny/terminus):
  highly configurable terminal emulator for Windows, macOS and Linux
- [mrxvt](https://sourceforge.net/projects/materm) (looks abandoned) -
  https://sourceforge.net/p/materm/feature-requests/41/
- [aterm](http://www.afterstep.org/aterm.php) (looks abandoned) -
  https://sourceforge.net/p/aterm/feature-requests/23/
- [fbcon](https://www.kernel.org/doc/Documentation/fb/fbcon.txt) (from linux
  kernel) - https://bugzilla.kernel.org/show_bug.cgi?id=79551
- FreeBSD console - https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=191652
- [yaft](https://github.com/uobikiemukot/yaft) framebuffer terminal -
  https://github.com/uobikiemukot/yaft/issues/12
- [KiTTY](http://www.9bis.net/kitty/) - **Windows platform**
- [MTPuTTY](ttyplus.com) - **Windows platform**
- [mRemoteNG](https://mremoteng.org/) - **Windows platform** -
  https://github.com/mRemoteNG/mRemoteNG/issues/717
- [JuiceSSH](https://juicessh.com/) - **Adroid platform**, closed source
- [Termius](https://www.termius.com/) - **Linux, Windows, OS X platforms**,
  closed source
- [SmarTTY](http://smartty.sysprogs.com/) - **Windows platform** - closed source
  (sent them a request)
- [Netsarang XShell](https://www.netsarang.com/products/xsh_overview.html) -
  closed source (sent them an email)
- libvte and GTK2 - based:
  - **libvte**-based [GTKTerm2](http://gtkterm.feige.net/)
  - **libvte**-based [stjerm](https://github.com/stjerm/stjerm) (looks
    abandoned) - https://github.com/stjerm/stjerm/issues/39

# Console Programs + True Color

## Console Programs Supporting True Color

- [s-lang](http://lists.jedsoft.org/lists/slang-users/2015/0000020.html)
  library - (since pre2.3.1-35, for 64bit systems)
- [ncurses](https://www.gnu.org/software/ncurses/) library - since 6.1 version
- [Eternal Terminal](https://mistertea.github.io/EternalTCP/) - automatically
  reconnecting shell
- [mc](http://www.midnight-commander.org) - since
  [682a5...](http://www.midnight-commander.org/changeset/682a5116edd20b8ba81743a1f7495c883b0ce644).
  See also [ticket #3724](http://www.midnight-commander.org/ticket/3724) for
  truecolor themes.
- [irssi](https://github.com/irssi/irssi) - since
  [PR #48](https://github.com/irssi/irssi/pull/48)
- [neovim](https://github.com/neovim/neovim) - since commit
  [8dd415e887923f99ab5daaeba9f0303e173dd1aa](https://github.com/neovim/neovim/commit/8dd415e887923f99ab5daaeba9f0303e173dd1aa);
  need to set
  [termguicolors](https://neovim.io/doc/user/options.html#%27termguicolors) to
  enable true color.
- [vim](https://github.com/vim/vim) - (from 7.4.1770); need to set
  [termguicolors](https://github.com/vim/vim/blob/master/runtime/doc/version8.txt#L202)
  to enable true color.
- [joe](https://sf.net/p/joe-editor) - (from
  [4.5](https://sourceforge.net/p/joe-editor/news/2017/09/joe-45-released/)
  version)
- [emacs](https://www.gnu.org/software/emacs/) - since
  [26.1 release](https://lists.gnu.org/archive/html/emacs-devel/2018-05/msg00765.html)
- [micro editor](https://micro-editor.github.io/)
- [elinks](http://repo.or.cz/w/elinks.git) -
  [configure.in:1410](http://repo.or.cz/w/elinks.git/blob/HEAD:/configure.in#l1410)
  (./configure --enable-true-color)
- [tcell](https://github.com/gdamore/tcell) library for Go language
- [timg](https://github.com/hzeller/timg) - Terminal Image Viewer
- [tv](https://github.com/daleroberts/tv) - tool to quickly view high-resolution
  multi-band imagery directly in terminal
- [termimage](https://github.com/nabijaczleweli/termimage) - terminal image
  viewer
- [explosion](https://github.com/Tenzer/explosion) - terminal image viewer
- [ls-icons](https://github.com/sebastiencs/ls-icons) - fork of coreutils with
  `ls` program that supports icons
- [mpv](https://github.com/mpv-player/mpv) - video player with support of
  console-only output (since 0.22 version)
- [radare2](https://github.com/radare/radare2) - reverse engineering franework;
  since 0.9.6 version.

## Console Programs Not Supporting True Color

- mutt (email client) - http://dev.mutt.org/trac/ticket/3674
- neomutt (email client) - https://github.com/neomutt/neomutt/issues/85
- termbox library - https://github.com/nsf/termbox/issues/37
- mcabber (jabber client) -
  https://bitbucket.org/McKael/mcabber-crew/issue/126/support-for-true-color-16-millions-colors
- tig (git TUI) - https://github.com/jonas/tig/issues/227
- cmus (music player) - https://github.com/cmus/cmus/issues/799
- weechat (chat client) - https://github.com/weechat/weechat/issues/1364
- scim (spreadsheet program) - https://github.com/andmarti1424/sc-im/issues/306
- [gui.cs](https://github.com/migueldeicaza/gui.cs) Terminal UI toolkit for .NET
  (curses-like) - https://github.com/migueldeicaza/gui.cs/issues/48

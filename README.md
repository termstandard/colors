# Terminal Colors

_(Previously published and discussed at https://gist.github.com/XVilka/8346728.)_

There exists common confusion about terminal colors. This is what we have right now:

- Plain ASCII without color controls;
- Using ANSI escape sequences:
  - 8 colors as a 2×2×2 color-cube, plus bright and dim foreground;
  - 16 colors: 8 colors, plus corresponding bright versions;
  - 256-color palette: 16 colors, plus a 6×6×6 color-cube and a 24-level gray scale;
  - 24-bit TrueColor as a 256×256×256 color-cube (aka "16 million" or just "888")

The 256-color palette has a standardized initial arrangement, but all entries
are separately reprogrammable. Using this mode, a terminal can display any 24-bit
RGB colors, the same as TrueColor, but only 256 distinct colors simultaneously.
The other modes do not use a palette; they just specify colors directly.

To see if your terminal supports TrueColor, run:

```bash
fgbg=48 # background
fgbg=38 # foreground
red=255
green=102
blue=0
printf '\e[%u;2;%u;%u;%um%s\e[m\n' "$fgbg" "$red" "$green" "$blue" TRUECOLOR
```

which will print <span style="color:#f60">TRUECOLOR</span> in brown if it
understands Xterm-style TrueColor escape sequences.

For a more thorough test, run:

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

Some other tests:

- [gist/lilydjwg/colors.py](https://gist.github.com/lilydjwg/fdeaf79e921c2f413f44b6f613f6ad53)
- https://github.com/robertknight/konsole/tree/master/tests/color-spaces.pl
- https://github.com/JohnMorales/dotfiles/blob/master/colors/24-bit-color.sh
- https://git.gnome.org/browse/vte/tree/perf/img.sh

<div style="padding-left: 2rem; border: 1px solid #9cf;">
You can download these scripts and inspect them before running them, by:

```bash
# make a sandbox
mkdir tmp$$
cd tmp$$

# downloads scripts
wget https://github.com/robertknight/konsole/raw/master/tests/color-spaces.pl \
     https://gist.github.com/lilydjwg/fdeaf79e921c2f413f44b6f613f6ad53/raw/94d8b2be62657e96488038b0e547e3009ed87d40/colors.py \
     https://github.com/JohnMorales/dotfiles/raw/master/colors/24-bit-color.sh \
     https://gitlab.gnome.org/GNOME/vte/-/raw/master/perf/img.sh

# read the scripts with your editor
$EDITOR *
```
<span style="color:#c00">Stop!</span>
Only if you're satisfied that the scripts are trustworthy should you proceed:
```bash
# if you trust them, run them
perl color-spaces.pl
python colors.py
bash 24-bit-color.sh
bash img.sh
```
</div>

Keep in mind that it is possible to use both ';' and ':' as Control Sequence
delimiters.

According to Wikipedia[1], this behavior is only supported by xterm and konsole.

[1] https://en.wikipedia.org/wiki/ANSI_color

# Truecolor Detection

## Checking for COLORTERM

[VTE](https://bugzilla.gnome.org/show_bug.cgi?id=754521),
[Konsole](https://bugs.kde.org/show_bug.cgi?id=371919) and
[iTerm2](https://gitlab.com/gnachman/iterm2/issues/5294) all advertise
truecolor support by placing `COLORTERM=truecolor` in the environment of the
shell user's shell. This has been in VTE for a while, but is relatively new in
Konsole and iTerm2 and has to be enabled at compile time (most packages do not,
so you have to compile them yourself from the git source repo).

The S-Lang library has a check that `$COLORTERM` contains either "truecolor" or
"24bit" (case sensitive).

Terminfo has supported the 24-bit TrueColor capability since
[ncurses-6.0-20180121](https://lists.gnu.org/archive/html/bug-ncurses/2018-01/msg00045.html),
under the name "RGB".
You need to use the "setaf" and "setab" commands to set the foreground and
background respectively.

Having an extra environment variable (separate from `TERM`) is not ideal: by
default it is not forwarded via sudo, ssh, etc, and so it may still be
unreliable even where support is available in programs. (It does however err on
the side of safety: it does not advertise support when it is not actually
supported, and the programs should fall back to using 8-bit color.)

These issues can be ameliorated by adding `COLORTERM` to:
* the `SendEnv` list in `/etc/ssh/ssh_config` on ssh clients;
* the `AcceptEnv` list in `/etc/ssh/sshd_config` on ssh servers; and
* the `env_keep` list in `/etc/sudoers`.

Despite these problems, it's currently the best option, so checking
`$COLORTERM` is recommended since it will lead to a more seamless desktop
experience where only one variable needs to be set.

App developers can freely choose to check for this variable, or introduce their
own method (e.g. an option in their config file). They should use whichever
method best matches the overall design of their app.

Ideally any terminal that really supports truecolor would set this variable;
but as a work-around you might need to put a check in your shell's start-up file
(e.g. `/etc/profile` or `~/.profile` or `~/.bashrc` or `~/.zshrc`) to set
`COLORTERM=truecolor` when `$TERM` matches any terminal type known to have
working truecolor:

```lang=sh
case $TERM in
  iterm       |\
  vte*        |\
  *-truecolor ) export COLORTERM=truecolor ;;
esac
```

## Querying The Terminal

In an interactive program that can read terminal responses, a more reliable
method is available that is transparent to sudo & ssh. Simply try sending a
TrueColor escape sequence to the terminal, followed by a query to ask what
color it currently has. If the response indicates the same color as
was just set, then TrueColor is supported.

If the response indicates an 8-bit color, or does not indicate a color, or if
no response is forthcoming within a few centiseconds, then TrueColor is
probably unsupported.

```bash
$ ( printf '\e[48:2:1:2:3m\eP$qm\e\\' ; xxd -g1 )

^[P1$r48:2:1:2:3m^[\
00000000: 1b 50 31 24 72 34 38 3a 32 3a 31 3a 32 3a 33 6d  .P1$r48:2:1:2:3m
```

Here we set the background color to `RGB(1,2,3)` - an unlikely default
choice - and request the value that we just set. The response comes back that
the request was understood (`1`), and that the color is indeed `48:2:1:2:3`.
This tells us also that the terminal supports the colon delimiter. If instead,
the terminal did not support truecolor we might see a response like

```
^[P1$r40m^[\
00000000: 1b 50 31 24 72 34 30 6d 1b 5c 0a              .P1$r40m.\.
```


This terminal replied that the color is `40` - it has not accepted our request
to set `48:2:1:2:3`.

```
^[P0$r^[\
00000000: 1b 50 30 24 72 1b 5c 0a                      .P0$r.\.
```

This terminal did not even understand the `DECRQSS` request - its response was
`CSI`+`0$r`. This does not indicate whether it set the color, but since it
doesn't understand how to reply to our request it is unlikely to support
truecolor either.

# Truecolor Support in Output Devices

## Fully Supporting

### Terminal Emulators

- [alacritty](https://github.com/jwilm/alacritty) [delimiter: colon, semicolon] - written in Rust
- [Black Screen](https://github.com/shockone/black-screen) [delimiter: semicolon] - cross-platform, HTML/CSS/JS-based
- [cmd.exe](https://en.wikipedia.org/wiki/Cmd.exe) [delimiter: semicolon] Built-in Windows shell that is mostly unchanged since DOS **Windows 10**
- [ConEmu](https://github.com/Maximus5/ConEmu) [delimiter: semicolon] - **Windows platform**
- [ConHost](https://en.wikipedia.org/wiki/Windows_Console) [delimiter: semicolon] Built-in Windows console (usually hosting cmd.exe) that is mostly unchanged since DOS **Windows 10**
- [ConnectBot](https://connectbot.org/) - **Android platform** - since [3bcc75ccedaf2136b04c5932c81a5155f29dc3b5](https://github.com/connectbot/connectbot/commit/3bcc75ccedaf2136b04c5932c81a5155f29dc3b5) commit.
- [Contour](https://github.com/contour-terminal/contour) [delimiter: semicolon] - written in C++17, uses OpenGL
- [cool-retro-term](https://github.com/Swordfish90/cool-retro-term) [delimiter: semicolon]
- [FinalTerm](http://finalterm.org/) [delimiter: semicolon] - **[abandoned](https://worldwidemann.com/finally-terminated/)**, iTerm2 [borrowing it's ideas and features](https://iterm2.com/documentation-shell-integration.html).
- [foot](https://codeberg.org/dnkl/foot) [delimiter: colon, semicolon] - Wayland terminal
- [ghostty](https://ghostty.org/) [delimiter: semicolon] - written in Zig
- [hterm](https://chromium.googlesource.com/apps/libapps/+/master/hterm) - HTML/CSS/JS-based (ChromeOS)
- [iTerm2](https://iterm2.com/) [delimiter: colon, semicolon] - since v3 version
- [kitty](https://github.com/kovidgoyal/kitty) [delimiter: colon,semicolon] - uses OpenGL
- [konsole](https://konsole.kde.org/) [delimiter: colon, semicolon] - https://bugs.kde.org/show_bug.cgi?id=107487
- [MacTerm](https://github.com/kmgrant/macterm) [delimiter: semicolon] - **Mac OS X platform**
- [mintty](https://mintty.github.io/) [delimiter: semicolon] **Cygwin and MSYS/MSYS2** since commit [43f0ed8a46c6549cb9a3ea27abc057b5abe13bdb](https://github.com/mintty/mintty/commit/43f0ed8a46c6549cb9a3ea27abc057b5abe13bdb) (2.0.1 release) - **Windows platform**
- [MobaXterm](https://mobaxterm.mobatek.net/) **Windows platform** - closed source (run `lscolors` to see a truecolor test)
- [mosh](https://mosh.org/) (Mobile SHell) [delimiter: semicolon] - since commit [6cfa4aef598146cfbde7f7a4a83438c3769a2835](https://github.com/mobile-shell/mosh/commit/6cfa4aef598146cfbde7f7a4a83438c3769a2835)
- [Netsarang XShell](https://www.netsarang.com/products/xsh_overview.html) - Xshell7/ Xshell6 >= Build 0181 (You must set _**Tools-Options.. -Advanced**_, check the _**Use true color\***_ and **reopen** the software)
- [pangoterm](http://www.leonerd.org.uk/code/pangoterm/) [delimiter: colon, semicolon]
- [PowerShell Core](https://github.com/PowerShell/PowerShell) [delimiter: semicolon] aka PowerShell 6+ **Windows 10**
- [Ptyxis](https://gitlab.gnome.org/chergert/ptyxis) [delimiter: semicolon] - supports 8/16/256 colors and 24-bit true color (via VTE; true color since VTE 0.36 ~2014, full in Ptyxis from v45 in 2023) - **Linux/GNOME platform** - open source (GPL-3.0-or-later); container-focused with GPU acceleration
- [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/) - [landed](https://git.tartarus.org/?p=simon/putty.git;a=commit;h=a4cbd3dfdb71d258e83bbf5b03a874c06d0b3106) in git (patched version [3] {approximation to 256 colors} and [4] {real truecolors} available) - **Windows platform**
- [qterminal](https://github.com/lxqt/qterminal) [delimiter: semicolon] - after version 0.14.1 ([issue #78](https://github.com/qterminal/qterminal/issues/78))
- [st](https://st.suckless.org/) (from suckless) [delimiter: semicolon] - https://lists.suckless.org/dev/1307/16688.html
- [Tera Term](http://en.sourceforge.jp/projects/ttssh2/) [delimiter: colon, semicolon] - **Windows platform**
- All [TerminalCtrl](https://github.com/ismail-yilmaz/Terminal) based terminal emulators.
  - [Bobcat](https://github.com/ismail-yilmaz/Bobcat) [delimiter: colon, semicolon] - a cross-platform terminal emulator, written in [U++](https://github.com/ultimatepp/ultimatepp) (C++ framework).
- [Termux](https://termux.com/) [delimiter: semicolon] - **Android platform**
- [Therm](https://github.com/trufae/Therm) [delimiter: colon, semicolon] - fork of iTerm2
- [upterm](https://github.com/railsware/upterm) *Windows/MacOS/Linux Electron* - A terminal emulator for the 21st century.
- [Warp](https://github.com/warpdotdev/Warp) - **Windows/MacOS/Linux** written in Rust
- [wezterm](https://wezfurlong.org/wezterm/) [delimiter: colon, semicolon] - written in Rust
- Windows 10 bash console, since [Windows Insiders build 14931](https://blogs.msdn.microsoft.com/commandline/2016/09/22/24-bit-color-in-the-windows-console/)
- [Windows Powershell](https://en.wikipedia.org/wiki/PowerShell#PowerShell_5.1) [delimiter: semicolon] - aka PowerShell 5.x and below **Windows 10**
- [Windows Terminal](https://github.com/microsoft/terminal) [delimiter: semicolon] - official, modern terminal emulator for Windows
- [xst](https://github.com/gnotclub/xst) - fork of st
- [xterm](https://invisible-island.net/xterm/) - (from [331 (change 330j)](https://invisible-island.net/xterm/xterm.log.html#xterm_331)
- All [libvte](https://download.gnome.org/sources/vte/) based terminals (since 0.36 version) [delimiter: colon, semicolon] - https://bugzilla.gnome.org/show_bug.cgi?id=704449
  - **libvte**-based [evilvte](https://www.calno.com/evilvte/) - no release yet, version from git https://github.com/caleb-/evilvte
  - **libvte**-based [Gnome Terminal](https://help.gnome.org/users/gnome-terminal/stable/)
  - **libvte**-based [guake](http://guake-project.org/) - A top-down terminal for GNOME
  - **libvte**-based [Lilyterm](https://lilyterm.luna.com.tw/) - since commit [72536e7ba448ad9ef1126ce45fbde3a3407a271b](https://github.com/Tetralet/LilyTerm/commit/72536e7ba448ad9ef1126ce45fbde3a3407a271b)
  - **libvte**-based [lxterminal](https://sourceforge.net/projects/lxde) - with **--enable-gtk3** configure flag.
  - **libvte**-based [Pantheon Terminal](https://launchpad.net/pantheon-terminal)
  - **libvte**-based [ROXTerm](http://roxterm.sourceforge.net/)
  - **libvte**-based [sakura](https://www.pleyades.net/david/projects/sakura)
  - **libvte**-based [Terminator](https://gnometerminator.blogspot.com/p/introduction.html) - since [1.90](https://launchpad.net/terminator/+announcement/14358) release
  - **libvte**-based [Termit](https://github.com/nonstop/termit)
  - **libvte**-based [Termite](https://github.com/thestinger/termite) ([NOT MAINTAINED](https://github.com/thestinger/termite/issues/760))
  - **libvte**-based [Tilda](https://github.com/lanoxx/tilda)
  - **libvte**-based [Tilix](https://github.com/gnunn1/tilix) - written in D. Similar user interface as for Terminator.
  - **libvte**-based [tinyterm](https://code.google.com/p/tinyterm)
  - **libvte**-based [xfce4-terminal](https://docs.xfce.org/apps/terminal/start) - since [0.6.90](https://github.com/xfce-mirror/xfce4-terminal/releases/tag/xfce4-terminal-0.6.90) release, if compiled with GTK+3
- All [xterm.js](https://github.com/xtermjs/xterm.js) based terminals (since [v3.13](https://github.com/xtermjs/xterm.js/issues/484), [v4.3 for webgl](https://github.com/xtermjs/xterm.js/pull/2552)) [delimiter: semicolon]
  - [Hyper.app](https://hyper.is/): crossplatform, HTML/CSS/JS-based (Electron)
  - [Tabby](https://github.com/Eugeny/tabby): highly configurable terminal emulator for Windows, macOS and Linux
  - [VS Code](https://code.visualstudio.com/)'s integrated terminal
- [ZOC](https://www.emtec.com/zoc/index.html) **Windows/OS X platform** - closed source since [7.19.0 version](https://www.emtec.com/downloads/zoc/zoc_changes.txt)
- [Terminal.app](https://en.wikipedia.org/wiki/Terminal_(macOS)) **MacOS platform** - since MacOS 26

There are a bunch of libvte-based terminals for GTK2, so they are listed in the
another section.

### Multiplexers

- [dvtm](https://github.com/martanne/dvtm) - not yet supporting truecolor https://github.com/martanne/dvtm/issues/10
- [pymux](https://github.com/jonathanslenders/pymux) - tmux clone in pure Python (to enable truecolor run pymux with `--truecolor` option)
- [screen](https://git.savannah.gnu.org/cgit/screen.git/) - has support in 'master' branch, need to be enabled (see 'truecolor' option)
- [tmux](https://tmux.github.io/) - starting from version 2.2 (support since [427b820...](https://github.com/tmux/tmux/commit/427b8204268af5548d09b830e101c59daa095df9))

### Re-players

- [asciinema](https://asciinema.org/) player:
  https://github.com/asciinema/asciinema-player

## Partial Support

These terminal emulators parse ANSI color sequences, but approximate the true
color using a palette or limit number of true colors that can be used at the
same time. A 256-color (8-bit) palette is used unless specified.

- Linux console ([fbcon](https://www.kernel.org/doc/html/latest/fb/fbcon.html)), [since v3.16](https://github.com/torvalds/linux/commit/cec5b2a97a11ade56a701e83044d0a2a984c67b4) - https://bugzilla.kernel.org/show_bug.cgi?id=79551 (downgraded to 16 foregrounds and 8 backgrounds)
- [mlterm](http://mlterm.sourceforge.net/) - built with **--with-gtk=3.0** configure flag. Approximates colors using a 512-color embedded palette (https://sourceforge.net/p/mlterm/bugs/74/)
- [urxvt aka rxvt-unicode](http://software.schmorp.de/pkg/rxvt-unicode.html) - since [revision 1.570](http://cvs.schmorp.de/rxvt-unicode/src/command.C?revision=1.570&view=markup&sortby=log&sortdir=down). Limits maximum number of colors: http://lists.schmorp.de/pipermail/rxvt-unicode/2016q2/002261.html

#### Note about color differences

Human eyes are sensitive to the primary colors in such a way that the simple
Gaussian distance √(R²+G²+B²) gives poor results when trying to find the
"nearest" available color as perceived by most humans.

The [CIEDE2000](https://en.wikipedia.org/wiki/Color_difference#CIEDE2000)
formula provides much better perceptual matching, but it is considerably more
complex and may perform very slowly if used blindly [2].

[2] https://github.com/neovim/neovim/issues/793#issuecomment-48106948

## Not Supporting Truecolor

- [aterm](http://www.afterstep.org/aterm.php) (looks abandoned) - https://sourceforge.net/p/aterm/feature-requests/23/
- [Cmder](https://cmder.net/): Portable console emulator for Windows, based on ConEmu.
- [fbcon](https://www.kernel.org/doc/Documentation/fb/fbcon.txt) (prior to Linux 3.16) - https://bugzilla.kernel.org/show_bug.cgi?id=79551
- [frecon](https://chromium.googlesource.com/chromiumos/platform/frecon/) - Console that is part of ChromeOS kernel
- [FreeBSD console](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=191652)
- [Hyper.app](https://hyper.is/) [delimiter: semicolon] - cross-platform, HTML/CSS/JS-based (Electron) https://github.com/zeit/hyper/issues/2294
- [JuiceSSH](https://juicessh.com/) - **Android platform**, closed source
- [KiTTY](https://www.9bis.net/kitty/) - **Windows platform**
- [mRemoteNG](https://mremoteng.org/) - **Windows platform** - [issue #717](https://github.com/mRemoteNG/mRemoteNG/issues/717)
- [mrxvt](https://sourceforge.net/projects/materm) (looks abandoned) - https://sourceforge.net/p/materm/feature-requests/41/
- [MTPuTTY](https://ttyplus.com/) - **Windows platform**
- [SmarTTY](https://sysprogs.com/SmarTTY/) - **Windows platform** - closed source (sent them a request)
- [Terminology](https://www.enlightenment.org/about-terminology) (Enlightenment) - https://phab.enlightenment.org/T746
- [Terminus](https://github.com/Eugeny/terminus): highly configurable terminal emulator for Windows, MacOS and Linux
- [Termius](https://www.termius.com/) - **Linux, Windows, OS X platforms**, closed source
- [yaft](https://github.com/uobikiemukot/yaft) framebuffer terminal - [issue #12](https://github.com/uobikiemukot/yaft/issues/12)
- libvte and GTK2 - based:
  - **libvte**-based [GTKTerm2](http://gtkterm.feige.net/)
  - **libvte**-based [stjerm](https://github.com/stjerm/stjerm) (looks abandoned) - [issue #39](https://github.com/stjerm/stjerm/issues/39)

# Console Programs + Truecolor

## Console Programs Supporting Truecolor

- [clifm](https://github.com/leo-arch/clifm) - The command line file manager
- [dte](https://gitlab.com/craigbarnes/dte) text editor - (since [version 1.8](https://craigbarnes.gitlab.io/dte/releases.html#v1.8))
- [elinks](https://repo.or.cz/w/elinks.git) - [configure.in:1410](https://repo.or.cz/w/elinks.git/blob/HEAD:/configure.in#l1410) (./configure --enable-true-color)
- [emacs](https://www.gnu.org/software/emacs/) - since [26.1 release](https://lists.gnu.org/archive/html/emacs-devel/2018-05/msg00765.html)
- [Eternal Terminal](https://mistertea.github.io/EternalTCP/) - automatically reconnecting shell
- [explosion](https://github.com/Tenzer/explosion) - terminal image viewer
- [irssi](https://github.com/irssi/irssi) - since [PR #48](https://github.com/irssi/irssi/pull/48)
- [joe](https://sf.net/p/joe-editor) - (from [4.5](https://sourceforge.net/p/joe-editor/news/2017/09/joe-45-released/) version)
- [ls-icons](https://github.com/sebastiencs/ls-icons) - fork of coreutils with `ls` program that supports icons
- [mc](https://midnight-commander.org/) - since [682a5...](https://midnight-commander.org/changeset/682a5116edd20b8ba81743a1f7495c883b0ce644). See also [ticket #3724](https://midnight-commander.org/ticket/3724) for truecolor themes.
- [micro editor](https://micro-editor.github.io/)
- [mpv](https://github.com/mpv-player/mpv) - video player with support of console-only output (since 0.22 version)
- [ncurses](https://www.gnu.org/software/ncurses/) library - since 6.1 version
- [neovim](https://github.com/neovim/neovim) - since commit [8dd415e887923f99ab5daaeba9f0303e173dd1aa](https://github.com/neovim/neovim/commit/8dd415e887923f99ab5daaeba9f0303e173dd1aa); need to set [termguicolors](https://neovim.io/doc/user/options.html#%27termguicolors) to enable truecolor.
- [Notcurses](https://notcurses.com/) library - all releases
- [radare2](https://github.com/radareorg/radare2) - reverse engineering framework; since 0.9.6 version.
- [rizin](https://github.com/rizinorg/rizin) - reverse engineering framework; since the inception (a fork of radare2).
- [s-lang](https://lists.jedsoft.org/lists/slang-users/2015/0000020.html) library - (since pre2.3.1-35, for 64bit systems)
- [tcell](https://github.com/gdamore/tcell) library for Go language
- [termimage](https://github.com/nabijaczleweli/termimage) - terminal image viewer
- [termpaint](https://termpaint.namepad.de) low level library (C) - all releases
- [timg](https://github.com/hzeller/timg) - Terminal Image Viewer
- [Tui Widgets](https://tuiwidgets.namepad.de) library (C++/QtCore) - all releases
- [tv](https://github.com/daleroberts/tv) - tool to quickly view high-resolution multi-band imagery directly in terminal
- [vifm](https://github.com/vifm/vifm) file manager - since 0.12 version
- [vim](https://github.com/vim/vim) - (from 7.4.1770); need to set [termguicolors](https://github.com/vim/vim/blob/master/runtime/doc/version8.txt#L202) to enable truecolor.

## Console Programs Not Supporting Truecolor

- [cmus](https://github.com/cmus/cmus) (music player) - [issue #799](https://github.com/cmus/cmus/issues/799)
- [gui.cs](https://github.com/migueldeicaza/gui.cs) Terminal UI toolkit for .NET (curses-like) - [issue #48](https://github.com/migueldeicaza/gui.cs/issues/48)
- [mcabber](https://mcabber.com/) (jabber client) - [issue #126](https://bitbucket.org/McKael/mcabber-crew/issue/126/support-for-true-color-16-millions-colors)
- [mutt](http://mutt.org/) (email client) - http://dev.mutt.org/trac/ticket/3674
- [neomutt](https://github.com/neomutt/neomutt) (email client) - [issue #58](https://github.com/neomutt/neomutt/issues/85)
- [scim](https://github.com/andmarti1424/sc-im) (spreadsheet program) - [issue #306](https://github.com/andmarti1424/sc-im/issues/306)
- [termbox](https://github.com/nsf/termbox) library - [issue #37](https://github.com/nsf/termbox/issues/37) (there is a fork [termbox_next](https://github.com/cylgom/termbox_next) with the support
- [tig](https://github.com/jonas/tig) (git TUI) - [issue #227](https://github.com/jonas/tig/issues/227)
- [weechat](https://github.com/weechat/weechat) (chat client) - [issue #1364](https://github.com/weechat/weechat/issues/1364)

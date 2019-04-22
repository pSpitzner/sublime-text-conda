Forked from [mandeep's great sublime-text-conda plugin](https://github.com/mandeep/sublime-text-conda/).

## What's New (macOS only, adapt as needed):

* There is a second build option `iTerm` that opens a new tab in [iTerm](https://www.iterm2.com/) and runs the script file you are currently editing in interactive mode. If an according tab is already open, it will be reused (or recreated if unresponsive).

* The build command has two new variables:
    - `\\$pyexec` is replaced with the executable name. Adjust `pyexec` in the package settings. (Needed for macOS to call `pythonw` when dealing with matplotlib. [See here](https://matplotlib.org/faq/osx_framework.html))
    - `\\$condaenv` can be used to create a custom build command and will be replaced with the active environment name. See `Conda.sublime-build` or Details below.

## Requirements/Installation

* iTerm shell integration

* The attached `itermtab` script needs to be executable
```
chmod +x /path/to/itermtab
```
(maybe `ln -s` this to a more favorable path)
Adapt as needed

* An iTerm profile with the name set to `Sublime`. Can be changed in `itermtab`

* `itermtab` relies on some custom functions in your `~/.profile`, mainly:
```
function iterm2_print_user_vars() {
  iterm2_set_user_var sublimetab "$SUBLIMETAB"
}
```

* I use the following `~/.profile`, for some extra bells and whistles:
```
# helpers to rename iterm tabs and windows
setTerminalText () {
    # echo works in bash & zsh
    local mode=$1 ; shift
    echo -ne "\033]$mode;$@\007"
}
stt_both  () { setTerminalText 0 $@; }
stt_tab   () { setTerminalText 1 $@; }
stt_title () { setTerminalText 2 $@; }

# my default, not activated, see below
terminalTextPrompt () {
  #stt_tab "${PWD##*/}";
  stt_tab   "${HOSTNAME%%.*} ${PWD##*/}/";
  stt_title   "${USER}@${HOSTNAME} - $PWD";
}

# use something else for sublime tabs
sublimePrompt() {
  stt_title   "${USER}@${HOSTNAME} - $PWD";
  stt_tab   "ST ${SUBLIMETAB##*/}";
}

# set color of tabs
function tabColor {
    case $1 in
    green)
    echo -en "\033]6;1;bg;red;brightness;57\a"
    echo -en "\033]6;1;bg;green;brightness;197\a"
    echo -en "\033]6;1;bg;blue;brightness;77\a"
    ;;
    red)
    echo -en "\033]6;1;bg;red;brightness;270\a"
    echo -en "\033]6;1;bg;green;brightness;60\a"
    echo -en "\033]6;1;bg;blue;brightness;83\a"
    ;;
    orange)
    echo -en "\033]6;1;bg;red;brightness;227\a"
    echo -en "\033]6;1;bg;green;brightness;143\a"
    echo -en "\033]6;1;bg;blue;brightness;10\a"
    ;;
    esac
 }

# set the command prompt to custom one for sublime tabs
if [ "$ITERM_PROFILE" = "Sublime" ]; then
    tabColor orange;
    export PROMPT_COMMAND=sublimePrompt
# or to customize the default
# else
    # export PROMPT_COMMAND=terminalTextPrompt
fi;

# iterm shell integration needs to be installed
test -e ${HOME}/.iterm2_shell_integration.bash && source ${HOME}/.iterm2_shell_integration.bash

# helper function needed to set environment variable when building from sublime
function iterm2_print_user_vars() {
  iterm2_set_user_var sublimetab "$SUBLIMETAB"
}

# create new tab with command from cli, this is optional
# see https://gist.github.com/vitalybe/021d2aecee68178f3c52
[ `uname -s` != "Darwin" ] && return
function tab () {
    local cmd=""
    local cdto="$PWD"
    local args="$@"

    if [ -d "$1" ]; then
        cdto=`cd "$1"; pwd`
        args="${@:2}"
    fi

    if [ -n "$args" ]; then
        cmd="; $args"
    fi

    osascript &>/dev/null 2>&1 <<EOF
        tell application "iTerm"
            tell current window
                set newTab to (create tab with default profile)
                tell newTab
                    tell current session
                        write text "cd \"$cdto\"$cmd"
                    end tell
                end tell
            end tell
        end tell
EOF
}
```

## Details

The `itermtab` command can be called from terminal:
```
itermtab foo/bar/ ls -l
```
will create a new tab in iTerm, set its environment variable `$SUBLIMETAB` to `foo/bar/` and call `ls -l`.

Calling `itermtab foo/bar/ ls -l` again, will recycle the tab by checking all tabs for `$SUBLIMETAB`.

Essentially sublime-text-conda is modified to allow this. In `Conda.sublime-build` you find

```
"variants":
    [
        {
            "name": "iTerm",
            "cmd": ["itermtab", "$file", "conda activate", "\\$condaenv", ";", "\\$pyexec", "-i", "$file"],
        }
    ]
```

Assuming we are editing `~/example.py` and the active environment is `test`, this will internally call
```
itermtab /Users/me/example.py conda activate /Users/me/miniconda/envs/test ; /Users/me/miniconda/envs/test/bin/pythonw -i /Users/me/example.py
```

Where `pyexec` is set to `pythonw` in settings. Accordingly, this calls `itermtab` to create/reuse a tab, executes the python script in the right environment/executable and gives as an interactive prompt (`-i`).


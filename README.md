# lrep: literate read-eval-print

`lrep` is a simple Unix filter for "literate repl" or "notebooks".
Documentation and input commands can be interleaved linewise, and the
commands drive a session whose output is merged into the document.

The syntax is Markdown compatible.

## Usage

    lrep [-z] [-t N] [FILES...]

* `-z`: zap, don't run any commands, and remove the former output
* `-t N`: adjust timeout to `N` seconds (default: 0.04),
  after which `lrep` will assume no further output happened.
  The initial command has `20*N` seconds time to display a prompt.

When no `FILES` are give, `lrep` reads from STDIN.

## Syntax

There are three kinds of lines of interest to `lrep`:

    <TAB>!!!CMD...

Non-interactive session: just display output of `CMD`.
When `CMD` ends with `>-`, no output is shown.

    <TAB>!PS1!PS2!CMD...

Starts a new session for `CMD...`.  The `!PS2` part is optional.
`PS1` is the first prompt considered, `PS2` the second prompt.
Note that prompts often end with a space in practice!
When `CMD` ends with `>-`, no output of startup is shown.

When `PS1` is empty, `PS2` needs to be provided.  Prompt-finding is
disabled and only timeouts are used to detect when new input can be sent.
Lines to input are marked using `PS2`.

    <TAB><PS1>INPUT...
    <TAB><PS2>INPUT...

Send a line to the session, and display the output.
When `CMD` ends with `>-`, no output is shown.

    <TAB>...

Other lines starting with `<TAB>` are assumed to be former output of
`lrep` and dropped.  Use four spaces for non-`lrep` verbatim code
(or fenced blocks).

## Algorithm

`lrep` runs the command in a PTY, and reads according to the following scheme:

1. Read as many bytes as possible without blocking.
2. If the last line of the string read contains PS1 or PS2 at the end, break.
3. Else, read as many bytes as possible without blocking or until timeout passed,
   and go to step 2.

(When PS1 is empty, the algorithm starts at step 3.)

## Example

For example, we can spawn a shell:

	!$ !sh

To send a line to the interpreter, start it with `<TAB>PS1`

	$ uname
	Linux

Commands also can generate more than one output line:

	$ printf '%s\n' foo bar baz
	foo
	bar
	baz

All commands run in the same session, so you can refer to previous results:

	$ x=foo
	$ y=bar
	$ echo ${x}${y}
	foobar

To just one-shot show the output of a non-interactive program, use
`<TAB>!!!CMD`

	!!! uname
	Linux

Let's try something in dc(1).  But since it doesn't have a prompt, we'll need
to use timed mode which is less robust and waits for 40ms without output.
Since `lrep` still needs to find the input strings, pass an alternate prompt
using `<TAB>!!PS2!CMD...`:

	!!> !dc
	> 6 7
	> +
	> p
	13

Perhaps Ruby is better.  This is also an example of how to use two prompts:

	!>> !?> !irb --simple-prompt
	>> 6 +
	?> 7
	=> 13
	>> 6 + 7
	=> 13

To use `lrep`, simply filter the file through the program:

	!!:!ed
	:a
	:A simple lrep example
	:	!!!echo elpmaxe na tsuj | rev
	:.
	:w try1.lrep
	53
	:q
	:# test death detection

	!!!./lrep <try1.lrep
	A simple lrep example
		!!!echo elpmaxe na tsuj | rev
		just an example

Note that is exactly how this file was created.

## Editor integration

Vi:

    map \e :!%./lrep<CR>``

Emacs:

    C-x h C-u M-| lrep <RET>

## Caveat

`lrep` will execute arbitary code in the input files, that's exactly
its job.  Be careful what you run it on.

## Copyright

`lrep` is in the public domain.

To the extent possible under law,
Christian Neukirchen <chneukirchen@gmail.com>
has waived all copyright and related or
neighboring rights to this work.

http://creativecommons.org/publicdomain/zero/1.0/

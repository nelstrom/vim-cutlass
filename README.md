# Cutlass.vim

**Cutlass.vim is no more than a README at present. I want to share these ideas and work through any problems with them before deciding whether or not to implement the plugin. In other words, I'm practicing *readme driven development*.**

Cutlass.vim is my attempt at addressing the problems that I discussed in [Vim's registers: the good, the bad, and the ugly parts][goodbadugly]. In brief: cutlass.vim introduces a cut operator `x{motion}`, and redefines Vim's "delete" operations (`d{motion}`, `D`, `c{motion}`, `C`, `s`, and `S`) by making them remove text without overwriting the default register. Redefining Vim's "delete" command to not *cut by default* means you'll make fewer accidental cut operations. (Cut*less* - geddit?)

Also, cutlass.vim modifies the registers `"1` through `"9` (the *history registers*) to record the last nine yank and cut operations. The `"-` *delete register* records the most recently deleted text.


## Introducing cut and delete operators

Cutlass.vim introduces a new operator command called *cut*, which is mapped to the `x` key.
The cut `x{motion}` operator behaves much like Vim's built-in `d{motion}` command: it removes text from the document and writes it to a register. The `X` key is equivalent to `x$`, just as Vim's built-in `D` command is equivalent to `d$`.

This means that we lose two of Vim's built-in commands: `x` and `X`, but we can reproduce their functionality using `xl` and `xh` to cut a character under or behind the cursor, respectively.

Cutlass.vim changes the behavior of the `d{motion}` command to perform a true deletion: the specified text is removed from the document without being written to the default register. The same goes for `c{motion}`, `s`, `D`, `C`, and `S` too.

From here on, I'll use *the delete operations* as a shorthand to refer to the `d{motion}`, `D`, `c{motion}`, `C`, `s`, and `S` commands.

## Redefining Vim's registers

The `x{motion}` and `y{motion}` (cut and copy) operations always write text to the default register `""` and register `"1`.
When text is written to register `"1`, Vim shifts the previous contents of register `"1` into register `"2`, `"2` into `"3`, and so forth, losing the previous contents of register `"9`.
As a result, registers `"1` through `"9` represent a clipboard history of cut and yank operations.
In cutlass.vim, registers `"1` - `"9` may be referred to collectivly as *the history registers*.

When the cut and yank operations are prefixed with a named register, the specified text is written to the named register, the default register, and to the history registers. For example, both `"ayiw` and `"axiw` would write the current word to `"a`, `""`, and `"1`.

The delete operations (`d{motion}`, `D`, `c{motion}`, `C`, `s`, and `S`) remove text from the document, writing it to register `"-` only.
Neither the default register nor the history registers are changed by the delete operations.
In cutlass.vim, register `"-` can be referred to as *the delete register*.

All delete operations can be prefixed with a named register, in which case they behave like the cut command.
For example, `"adiw` and `"axiw` would both cut the current word from the document, writing it to registers `"a`, `""`, and `"1`, but not to the delete register `"-`.

This table summarizes the behaviour of Vim's registers as defined by cutlass.vim:

<table>
  <tr>
    <th>operation</th>
    <th><code>""</code> (default)</th>
    <th><code>"1</code> (history)</th>
    <th><code>"a</code> (named)</th>
    <th><code>"-</code> (delete)</th>
  </tr>

  <tr>
    <td><code>y{motion}</code></td>
    <td>X</td>
    <td>X</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td><code>"ay{motion}</code></td>
    <td>X</td>
    <td>X</td>
    <td>X</td>
    <td>-</td>
  </tr>

  <tr>
    <td><code>x{motion}</code></td>
    <td>X</td>
    <td>X</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td><code>"ax{motion}</code></td>
    <td>X</td>
    <td>X</td>
    <td>X</td>
    <td>-</td>
  </tr>

  <tr>
    <td><code>d{motion}</code></td>
    <td>-</td>
    <td>-</td>
    <td>-</td>
    <td>X</td>
  </tr>
  <tr>
    <td><code>"ad{motion}</code></td>
    <td>X</td>
    <td>X</td>
    <td>X</td>
    <td>-</td>
  </tr>

</table>

See [The Bad Parts][bad] for an explanation of Vim's built-in behaviour for the numbered registers.


## Respecting common Vim idioms

Changing the behaviour of built-in commands such as `d` and `x` affects some common Vim idioms, such as `xp` and `ddp`. These idioms can easily be adapted to work with the operators defined by cutlass.vim.

### Toggle characters: `xp` becomes `xlp`

Using Vim's built-in commands, we can toggle two characters by pressing `xp`. For example, with the cursor at the start of this line, `xp` would fix the typo, producing `Roses`:

    oRses are red.

With cutlass.vim, this idiom becomes `xlp`. `xl` cuts the character under the cursor, saving it to the default register. `p` puts the contents of the default register after the cursor.

### Toggle lines: `ddp` becomes `xxp`

Using Vim's built-in commands, we can toggle two lines by pressing `ddp`. For example, with the cursor on the first line of this block of text, the line order could be fixed by pressing `ddp`:

    2)  Violets are blue,
    1)  Roses are red,
    3)  Sugar is sweet
    4)  And so are you.

With cutlass.vim, this idiom becomes `xxp`. `xx` cuts the line under the cursor, saving it to the default register. `p` puts the contents of the default register below the cursor.

### Visual put `v_p` writes to `"-`

Vim's built-in Visual mode `p` command overwrites the selected text with the contents of a register, then writes the old selection to the default register. Effectivly, the selected text and the default register swap values. And that allows for a neat workflow if you need to swap two chunks of text. For example, to swap the order of these arguments:

    execute(second, first)

With the cursor on `second`, we could cut the word with `diw`. Then select the word `first` and use `p` to overwrite the selection with the contents of the default register. As a side-effect of the Visual mode put operation, the default register would now contain the (formerly selected) word `first`. Moving the cursor back to the comma, we could use Normal mode `P` to insert `first` in its correct position. This sequence of keystrokes would do the job:

    diw
    mm
    ww
    viw
    p
    `m
    P

Would that still work with cutlass.vim? Yes - with a small modification. Cutlass.vim modifies the Visual mode put command, writing the selected text to `"-`, the delete register. All of the above steps would work the same way, but for the final `P` command we would have to prefix the `"-` delete register.

    diw
    mm
    ww
    viw
    p
    `m
    "-P

That's one reason why the delete operations write text to the *delete register* (`"-`), rather than to the blackhole register (`"_`).


## Feedback

What do you think: Should I build this? Would you use it?

Email your thoughts to [drew@vimcasts.org](mailto:drew@vimcasts.org).

[goodbadugly]: http://vimcasts.org/blog/2013/11/25/registers-the-good-the-bad-and-the-ugly-parts
[bad]: http://vimcasts.org/blog/2013/11/25/registers-the-good-the-bad-and-the-ugly-parts#the-bad-parts
[v_p]: http://vimdoc.sourceforge.net/htmldoc/change.html#v_p

----------------------------------------

                _,---,_
              /'_______`\
             (/'       `\|___________----------"""""""------,
              \#########||______                          /'
               ^^^^^^^^^||      """"""-----_____        /'
                         \                      """--_/'

----------------------------------------


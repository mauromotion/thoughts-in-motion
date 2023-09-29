---
layout: single
title: "Setting up VimWiki in Neovim"
toc:  true
date: 2023-09-29 10:00:00 +0000
categories: PKM vim
---
## To Logseq or not to Logseq
In recent years I've been trying many different solutions for my note taking needs. The app that I've been using the most has been [Logseq](https://logseq.com/), which I still use and like, but has some kinks and pain points that are not being actively addressed by the developing team recently: 
- Logseq is an Electron app and it's becoming quite slow. 
- There are still occasions where I lose some data (although, I sync my graph in a git repository so I'm safe 99% of the time).
- They're taking a long time to implement some quality of life features (like being able to use checkboxes properly, for example).
<br>
<br>
I'll most surely keep using Logseq, but I feel the need of something a bit less cumbersome for some of my note taking needs.
<br>
<br>
In particular, since I started my coding journey, I spend a lot of time in my terminal and text editor, which is Neovim.
After learning the vim motions I started to use them everywhere and I really miss them when writing notes. And so I decided to give a shot at [ViwmWiki](https://github.com/vimwiki/vimwiki), which is a note taking plugin for Vim/Neovim that creates local text files and an interlinked structure of notes, with some nice features.
<br>
The process of installing and configuring VimWiki properly in my Neovim set up though, wasn't always straightforward, and so I'm gonna document some solutions and workarounds here, for future reference.


## Translating the Vimscript commands into Lua
This will: 
- Set the default VimWiki's directory to `~/Notes` directory instead of the default `~/vimwiki`. 
- Set the default syntax to markdown (instead of the default .wiki format, more on this later).
- Replace the spaces in the file names with underscores, to avoid any possible issues. 
- Turns on the auto re-indexing of the tags database when I close the program.

```lua

vim.g.vimwiki_list = {
  { 
    path = "~/Notes/VimWiki",
    syntax = "markdown",
    ext = ".md",
    links_space_char = "_",
    auto_tags = 1 
  } 
}
```
To activate the syntax highlighting for code blocks instead, I use this:

```lua

vim.g.vimwiki_syntax_plugins = {
	codeblock = {
		["```lua"] = { parser = "lua" },
		["```python"] = { parser = "python" },
		["```javascript"] = { parser = "javascript" },
		["```bash"] = { parser = "bash" },
		["```html"] = { parser = "html" },
		["```css"] = { parser = "css" },
		["```c"] = { parser = "c" },
	},
}
```
### A pesky code snippet
Most important of all, I had to find where exactly to put this code in my Neovim config, because for some VimWiki quirks already not comletely clear to me, it won't work at all if I put them in my `settings.lua` file, as I'd like to.
<br>
As of now, my modular Neovim configuration works like this:
<br>
```
init.lua----
|   |   |   |
|   |   |   keymaps.lua
|   |   |
|   |   settings.lua
|   |
|   autocmds.lua
|
plugins.lua
    |
    init.lua   <--- PUT IT HERE!
    |
    LSP/
    UI/
    coding/
    navigation/
    treesitter/
    utilities/
```
<br>
I had to put the VimWiki config files right after calling LazyVim to load my plugins, in the `init.lua` file that loads all the plugins modules, otherwise Neovim won't read it at all.

## Markdown it all
I started playing around with VimWiki while keeping its original file format and syntax, but honestly it feels quite wasteful to learn and use yet another syntax that I will only ever use for VimWiki and nothing else. Writing notes in markdown is much better, it's kind of a standard nowadays, I already use it everywhere, and also it will come in handy in case I want to translate some notes into HTML, since with markdown the browser can do it for me. No need for any extra steps, post-processing or plugins whatsoever.

### MDwiki
Speaking of which, I found this little "program" (it's literally just an HTML file) called [MDwiki](http://dynalon.github.io/mdwiki/#!index.md), that can open all my VimWiki notes and structures in the browser, and render it clearly. All I have to do is put the file `MDwiki.html` in the root directory of VimWiki and launch a local server (for example I'm using [alive-server](https://www.npmjs.com/package/alive-server)), and the wiki will show up (locally) in my browser. Obviously, all the notes must be in markdown for this to work.
Pretty neat!

### Convert from .wiki to .md
Since I had already used VimWiki for a couple of days and I didn't want to redo all my structure (I use [PARA](https://fortelabs.com/blog/para/) by the way) and manually convert the few notes I wrote, I needed some kind of script for the conversion from .wiki to .md. 
It took me some trial and error but I'm happy to say that we have a winner here:

```bash

#!/bin/bash

set -euo pipefail

# files to change extension from to md format
readarray -d '' mv_files < <( \
  find . \( -iwholename '*diary/*.wiki' \
    -or -iwholename '*unorganized_things/*.wiki' \
    -or -iwholename '*design_docs/*.wiki' \
    -and -not -iname 'diary.wiki' \
    -and -not -iname 'index.wiki' \) \
    -print0 
)

for file in "${mv_files[@]}"; do
  md_file="${file%%.wiki}.md"
  mv "$file" "$md_file"
done

readarray -d '' files < <(find . -name "*.wiki" -print0)
for file in "${files[@]}"; do
  md_file="${file%%.wiki}.md"
  sed -r -e 's/\{\{\{$/\{\{\{bash/g' -e 's/%%/TODO: comment/g' "$file" | pandoc --from vimwiki --to commonmark_x -o "$md_file"
  sed -r -i -e 's/(\[.*\])\(([^#]*)((.*) "wikilink")\)/\1\(\2.md\4\)/g' \
    -e "s/\\\'/\'/g" \
    -e "s/\[\]\{\.done[0-3]\}/\[ \] /g" \
    -e "s/\[\]\{\.done4\}/\[X\] /g" \
    "$md_file"
  rm "$file"
done
```

The  only things I had to fix were a table and some backslashes around some files, but the rest worked flawlessly. Maybe I can tweak the script in the future. I didn't wrote the script, I found it on this blog [article](https://jnduli.co.ke/migrate_from_vimwiki_to_markdown_syntax.html).

## Tags system and Telescope
I use tags in my other systems quite extensively and I like to have a tag "system" in VimWiki as well. Too bad it's quite limited for now. Tags are added by surrounding a word with semicolons `:work:`, and can be chained together, if multiple: `:work:programming:python:`.
The major issue though is the search system: it's just bad, and slow. 
<br>
<br>
What I ended up doing is using [Telescope](https://github.com/nvim-telescope/telescope.nvim), which I have already installed in Neovim, with fzf and ripgrep for some sweet extra speed. 
<br>
And then I found this [plugin](https://github.com/ElPiloto/telescope-vimwiki.nvim) for Telescope on GitHub, which makes for an even better integration, so I can use specific keybindings to search directly only inside my VimWiki notes, either just for notes' titles or live grep all the notes' content, no matter the CWD. Alas, it looks like the development of the plugin is not active anymore but I wish they'd implement tags as well! 

## Shortcuts and commands
Here's a table of shortcuts for VimWiki commands and a few more to deal with the Neovim speller (which is nice to have): 

| Command                 | Description                                   |
|-------------------------|-----------------------------------------------|
| \<leader> ww            | Open VimWiki                                  |
| \<leader> wi            | Open VimWikiDiary                             |
| \<leader> w <leader> w  | Open Today                                    |
| \<leader> w <leader> i  | generate links to Diary pages                 |
| \<leader> fv            | Telescope: search VimWiki's notes titles      |
| \<leader> fw            | Telescope: live grep inside VimWiki directory |
| ----------------------- | -------------------------------------         |
| "n", zg                 | Add words to the spell checker                |
| "n", ]s - [s            | Jump to the next misspelled word              |
| "n", z=                 | Gives suggestions for correct words           |
| "i", <c-x> s            | Gives a list with suggestions                 |

<br>
That's all folks!

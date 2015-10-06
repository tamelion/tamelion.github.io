---
title: Dr Strange[Vim]love, or how I learned to stop worrying and love the [Atom] bomb
tags: [vim, atom, text-editor]
---
[Vim](https://en.wikipedia.org/wiki/Vim_(text_editor)) has been my editor of choice for a while now. For web development it is perfect -- lightweight, tightly integrated with the command line and highly customisable -- not to mention present by default on almost every linux servers. Despite having a steep learning curve (mostly due to its modal  nature), once you learn its vocabulary it feels highly inefficient to ever do without it.

Which brings me to the problem. I love efficiency (my [.vimrc file](https://github.com/tamelion/.dotfiles/blob/master/vim/.vimrc) is by far the most frequently updated config file I keep in my [.dotfiles repository](https://github.com/tamelion/.dotfiles) and it's mostly due to trying out new plugins to make me more productive) and I often like to try out *new and shiny* feature-packed editors. However, most editors which would potentially make me more productive just don't cut it: they either have no vim mode or it is very poor to say the least.

Enter [Atom](https://atom.io) editor. Atom hit its [stable 1.0 release](http://blog.atom.io/2015/06/25/atom-1-0.html) back in June and has two vim plugins: [vim-mode](https://atom.io/packages/vim-mode) (which provides the vim vocabulary) and [ex-mode](https://atom.io/packages/ex-mode) (which provides the ex editor style :colon commands). Neither are perfect, but they're a whole lot better than any other *new and shiny* editor I've tried.

The biggest things I miss from vim is the persistent undo (which gives you the [amazing undo tree](https://github.com/mbbill/undotree) feature -- SO handy). Also sed style replacement isn't perfect (I can't seem to do `:%s/deleteThis//g` with an empty replacement string). But for the trade-off of *new and shiny* features, it's by far the best vim-mode editor I've tried so far.

Let's see how long it lasts... I'll try hard, I promise.

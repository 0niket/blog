---
title: "Storing dotfiles with bare git repository"
datePublished: Tue Jul 11 2023 13:07:47 GMT+0000 (Coordinated Universal Time)
cuid: cljyb4ymn001309kw2xn8dnum
slug: storing-dotfiles-with-bare-git-repository
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/ohh8ROaQSJg/upload/d11fb68b724d8d25e7815a80939f7300.jpeg
tags: workflow, git, dotfiles

---

You've probably come across a situation where you had multiple machines and encountered the hassle of rewriting config files because you neglected to back them up for some inexplicable reason. Of course, there are alternative solutions for safeguarding your dotfiles. However, in this step-by-step guide, we'll delve into a rather peculiar approach that involves utilizing `git` as a means to back them up.

### Step 1: Initialize git [bare repository](https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server#:~:text=Putting%20the%20Bare%20Repository%20on%20a%20Server&text=git%20directory%2C%20they%20will%20also,any%20commits%2C%20refs%2C%20etc.)

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">A git bare repository does not have a working directory associated with it. Unlike a regular git repository, which includes both the version-controlled files and the working directory where you can modify and view the files, a bare repository only contains the version history and the repository's metadata.</div>
</div>

1. Create a new remote repository called `dotfiles`.
    
    [https://github.com/0niket/dotfiles.git](https://github.com/0niket/dotfiles.git)
    
2. Clone newly created repository locally. Notice the `--bare` flag.
    

```bash
$ git clone --bare <git-repo-url> $HOME/.cfg
```

### Step 2: Create an alias to avoid conflict

`config` is an alias for `git` to avoid running commands on local `.git` directory when you mean to run commands on `.cfg`.

```bash
$ echo "alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'" >> $HOME/.zshrc
$ source ~/.zshrc
```

### Step 3: Hide files that are not required to be tracked

By default, all files from your `$HOME` directory will be tracked which you do not want. To turn off the tracking of unnecessary files you'll need to do the following.

```bash
$ config config --local status.showUntrackedFiles no
```

After which your `$HOME/.cfg/config` file should look like below. The `[status]` part is appended after running above command.

```bash
[core]
        repositoryformatversion = 0
        filemode = true
        bare = true
        ignorecase = true
        precomposeunicode = true
[status]
        showUntrackedFiles = no
```

On running `config status` you should not see unnecessary files being tracked.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689074317847/e1d8eaf8-cf70-441f-a0a9-e1703fd946be.png align="center")

### Step 4: Add dotfiles and push

```bash
$ config add ~/.zshrc
$ config commit -m "Add zshrc"
$ config push -u origin main
```

# Installing dotfiles on new machine

Follow the first 3 steps from above.

```bash
$ config checkout
```

If dotfiles are already existing on the new machine then it'll show a conflict. In that case, you can delete or rename those files first.

Now, you can add edit your dotfiles and push to origin and follow same process on any machine to get your dotfiles.

---

# References

1. [https://news.ycombinator.com/item?id=11070797](https://news.ycombinator.com/item?id=11070797)
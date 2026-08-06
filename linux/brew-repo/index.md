---
title: How to create your own Homebrew repo?
url: https://devopstales.github.io/linux/brew-repo/
date: 2022-02-05
keywords: Homebrew, brew, Github, Taps
---


I this post I will show you in a Step-by-Step guide how cou can create your own Homebrew repository on GitHub.

<!--more-->

### What is Homebrew?

Homebrew is the missing package manager for macOS. It installs packages with a simple command like `brew install curl`. It can be used on Linux too. Homebrew taps are third-party repositories. You can create your own repository and host on GitHub.

In this article I will use `tmux-cssh` script as the program I will share by Homebrew.

### Create Git repo

To host a Homebrew tap you just need to create a GitGub git repository with the naming convention `homebrew-<name>`. I called `homebrew-devopstales`. 

```bash
git clone git@github.com:devopstales/homebrew-devopstales.gi
cd homebrew-devopstales
echo "# homebrew-devopstales" >> README.md                                  
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:devopstales/homebrew-devopstales.git
git push -u origin main
```

I will use this repo to host the `tmux-cssh` script too on another branch, but you can use It's own repo.

```bash
git checkout -b tmux-cssh
cp /usr/local/sbin/tmux-cssh .
git add -A
git commit -m "tmux-cssh"
git push --set-upstream origin tmux-cssh
git tag tmux-cssh_1.0
git commit -m "tmux-cssh_1.0"
git push origin --tags
```

I cerated a teg to use It's tar.gz in the release file of the package in Homebrew tap.

![release tar.gzm](/img/include/brew01.webp)

Right click on the tar.gz and sopy the link address. Now we go back the main branch and generate the release file for brew.

```bash
git checkout main
brew create https://github.com/devopstales/homebrew-devopstales/archive/refs/tags/tmux-cssh_1.0.tar.gz
```

It will show the content of the created file in your default text editor. Save the file and move to this git repo.

```bash
cp /home/linuxbrew/.linuxbrew/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/homebrew-devopstales.rb tmux-cssh.rb
```

Edit the `tmux-cssh.rb` file:

```ruby
class TmuxCssh < Formula
  desc "tmux-css allow parallel execution of cmmands on remote servers"
  homepage "https://github.com/knakayama/tmux-cssh"
  url "https://github.com/devopstales/homebrew-devopstales/archive/refs/tags/tmux-cssh_1.0.tar.gz"
  sha256 "ce761f21fa0fe7050f408e687f9a4964acf95736c85678e3cc27cee88bd6ceed"
  version "1.0"

  depends_on "tmux"

  def install
    bin.install "tmux-cssh"
  end

  test do
    system "#{bin}/tmux-cssh", "--help"
  end
end
```

If you have directories, you can use `Dir["lib"]` to install directories or use `prefix.install` to copy files:

```bash
def install    
    bin.install "tmux-cssh"    
    bin.install Dir["lib"]    
    bin.install Dir["files"]
    prefix.install "README.md"
    prefix.install "LICENSE"  
end
```

Once it is ready, add the file, commit, and push.

```bash
git add -A
git commit -m "tmux-cssh_1.0"
git push
```

Now your tap is ready. Run the following commands to use it:

```bash
brew tap devopstales/devopstales
brew install tmux-cssh
```

For update you just generate a new rb file with `brew create` command and the new tar.gz archive. Then change the `url` and `sha256` fields in tmux-cssh.rb.


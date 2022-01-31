# git-filter-repo


## Intro

_This post is a follow up to [Gitastrophes: Save your own a$$ - Part I](https://dev.to/heyjtk/gitastrophes-trying-to-save-your-own-a-with-bfg-2o44) which I encourage you to check out before diving into this one. Like the last post, this is geared towards those with intermediate or higher git skills._

Git, as we all know, is the powerful industry-standard version control system that enables collaboration and a historical footprint for software development efforts.

Reiterating what we said last time, the historical footprint of git can be...stubborn...so one of the worst disasters that can befall you while using it is for a secret, password, or other sensitive information to make it upstream to hosted remote repos on Github/Gitlab/Bitbucket.

While last time we discussed a tool called BFG Repo Cleaner, which I found easy to use, it has some **limitations**. We saw using BFG that we could clean a .env file with sensitive info out of all of our repo and commit history, **but that sensitive info was still available in the diff of a pull request**. 

Today we are going to use a tool I found a little harder to use, but with powers above what BFG can do. I'm also going to switch it up by using Gitlab instead of Github, just for kicks. Let's go! 

## Setup

First let's initialize our repo on Gitlab. 

![Create git-filter-repo repo](https://dev-to-uploads.s3.amazonaws.com/i/udr37o6bm0md9jp3brx5.png)

Let's clone it locally, and make some test files - I'm going to mimic the same setup from the first post in this series, in a new branch. 

```sh
➜  Desktop git clone git@gitlab.com:jtkaufman737/git-filter-repo.git
Cloning into 'git-filter-repo'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
➜  Desktop cd git-filter-repo 
➜  git-filter-repo (main) ✗ git checkout -b disaster 
Switched to a new branch 'disaster'
➜  git-filter-repo (disaster) ✔ touch .env .gitignore file.js file.py file.go file.sql
➜  git-filter-repo (disaster) ✗ ls
README.md file.go   file.js   file.py   file.sql
➜  git-filter-repo (disaster) ✗ 
```

Like last time, lets add some "sensitive" info to our env file, then emulate trying to gitignore it but accidentally with a typo (eek). 

```sh
➜  git-filter-repo (disaster) ✗ cat .env
SECRET=b8ee43a51b6ba41eb873dc83e11dbbcbbb3a2131
SUPER_SECRET=371a648b3dce5636aee0c1425aca8d3d59482ace
NUCLEAR_CODES=359e95931e95032ed8b71534b75c57eb17717d59
➜  git-filter-repo (disaster) ✗ cat .gitignore
.emv
```

Time to add our changes and push, ahead of a merge request. 

```sh
➜  git-filter-repo (disaster) ✗ git add .
➜  git-filter-repo (disaster) ✗ git commit -m "First commit, what could go wrong?"
[disaster 827cbb1] First commit, what could go wrong?
 6 files changed, 4 insertions(+)
 create mode 100644 .env
 create mode 100644 .gitignore
 create mode 100644 file.go
 create mode 100644 file.js
 create mode 100644 file.py
 create mode 100644 file.sql
➜  git-filter-repo (disaster) ✔ git push origin disaster 
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 12 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (5/5), 521 bytes | 521.00 KiB/s, done.
Total 5 (delta 0), reused 0 (delta 0), pack-reused 0
remote: 
remote: To create a merge request for disaster, visit:
remote:   https://gitlab.com/jtkaufman737/git-filter-repo/-/merge_requests/new?merge_request%5Bsource_branch%5D=disaster
remote: 
To gitlab.com:jtkaufman737/git-filter-repo.git
 * [new branch]      disaster -> disaster
```

I've opened a merge request, where we can now see the diff containing secret info is generated...

![Merge request containing secret files](https://dev-to-uploads.s3.amazonaws.com/i/bry4llntz3seej9p434d.png)

But for the purpose of this example we're going to merge it anyway. In addition to my local commit, we now have a merge commit where a diff shows our secrets, as well as the repo files containing them.

![Merge commit showing our secrets in .env](https://dev-to-uploads.s3.amazonaws.com/i/uipv6c56c4o36l74l3os.png)

![Record of the secrets also show up in main branch](https://dev-to-uploads.s3.amazonaws.com/i/eckvfdyd5okk7otz57dm.png)

## Git filter-repo to the rescue

Last time we talked about [BFG Repo Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) and also mentioned [git filter-branch](https://git-scm.com/docs/git-filter-branch) and [git filter-repo](https://github.com/newren/git-filter-repo). 

You may recall me saying in Part I that I found confusing and mixed references to these three tools and that was why I latched on to BFG early. Git filter-repo seems to represent the outgrowth of git filter-branch, which apparently had certain gotchas and pitfalls. 

Unfortunately, a lot of the documentation I found still references git filter-branch, so yeah: if you are in an emergency situation where something sensitive made it into a repo, trying to toggle between all the conflicting info is frustrating. 

Furthermore, I didn't _love_ the docs for git filter-repo. That being said, it can do things BFG cannot, so I found myself sticking with the documentation long enough to make sense of it. Once I did, the solution to the problem I just caused myself is actually...pretty awesome...

The process for removing all references to a file with git filter-repo begins by exiting our main repo and making a fresh clone, using the `--mirror` flag. 

```sh
➜  Desktop git clone --mirror git@gitlab.com:jtkaufman737/git-filter-repo.git
Cloning into bare repository 'git-filter-repo.git'...
remote: Enumerating objects: 10, done.
remote: Counting objects: 100% (10/10), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 10 (delta 1), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (10/10), done.
Resolving deltas: 100% (1/1), done.
➜  Desktop cd git-filter-repo.git
```

We next are going to use something called the `invert paths` command. I found that name a little weird - not what I would have thought to name it - but oh well. Invert paths is the command you are looking for to strike all references to our .env file from the history of our remote repo.  

**If you remember last time, we weren't able to strike .env info from the diffs from merge requests...with git filter-repo, we actually have the power to change those too! Pretty amazing.** 

The anatomy of this command is as follows: 

```sh
git filter-repo --path [file to wipe from history] --invert-paths 
``` 

For us, that looks like this: 

```sh
➜  git-filter-repo.git (main) ✔ git filter-repo --path .env --invert-paths
Parsed 4 commits
New history written in 0.05 seconds; now repacking/cleaning...
Repacking your repo and cleaning out old unneeded objects
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 12 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (9/9), done.
Building bitmaps: 100% (4/4), done.
Total 9 (delta 1), reused 5 (delta 0), pack-reused 0
Completely finished after 0.13 seconds.
```

Now we need to do a push without a remote branch specified to apply all updates. 

```sh
➜  git-filter-repo.git (main) ✔ git push
Enumerating objects: 9, done.
Writing objects: 100% (9/9), 843 bytes | 843.00 KiB/s, done.
Total 9 (delta 0), reused 0 (delta 0), pack-reused 9
To gitlab.com:jtkaufman737/git-filter-repo.git
 + 566472c...b0db77c main -> main (forced update)
 + ed71ff4...0f18913 refs/merge-requests/1/head -> refs/merge-requests/1/head (forced update)
 + e6a9a4b...d14436e refs/merge-requests/1/merge -> refs/merge-requests/1/merge (forced update)
 * [new branch]      refs/replace/566472c3d7060d35c448b778d1eed022695efed8 -> refs/replace/566472c3d7060d35c448b778d1eed022695efed8
 * [new branch]      refs/replace/e6a9a4bf5c9979e7602ffba27f2cbff3a3a46a72 -> refs/replace/e6a9a4bf5c9979e7602ffba27f2cbff3a3a46a72
 * [new branch]      refs/replace/ed71ff49006e9bb207cf5be0f4575f49f4026408 -> refs/replace/ed71ff49006e9bb207cf5be0f4575f49f4026408
```

The first time this worked it felt kind of anticlimactic...like, _that's it?_ So I had to see it with my own eyes. First lets take a look at our repo. 

![Repo, minus .env file! Woooot](https://dev-to-uploads.s3.amazonaws.com/i/o7b25t2j9ao3ww7dcf4a.png)

No .env - off to a good start. Now let's go back to our merge commit where we saw .env in the diff before... 

![Amaazing! .env file gone from merge comit that previously included it](https://dev-to-uploads.s3.amazonaws.com/i/1xmrdhen23rzwuuhzngr.png)

Amazing! Gone. Now, what about that commit in disaster branch where we also saw it earlier? 

![.env file history is even gone from the feature branch it was created int](https://dev-to-uploads.s3.amazonaws.com/i/evf7c1grj8i4vfcgjln3.png)

Now, the _pies de resistance_, the **last** remnant that BFG could not remove for us in Part I: the diff in the merge request.

![Our .env file details come up as a broken link, no secret info visible](https://dev-to-uploads.s3.amazonaws.com/i/8vgxeeuxgxhxf342mrk6.png)

WOOOOOOOOOT! So that's it: In five minutes or less using git filter-repo this is how you can not only clean up your repo, your commit history, but also your refs and merge request diffs! This fabulous tool should be handled with great care, but can be a major lifesaver if something sensitive has gotten into git history. 







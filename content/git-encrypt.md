Title: Secret files in Git - transparent encryption and history rewriting
Date: 2024-10-10 16:00
Status: published
Category: random
Tags: it, cloud
Slug: git-encrypt

I accidentaly posted on GitHub some files I shouldn't. Not password files or access tokens, just my input files from Advent of Code. I thought it doesn't matter since the files are generated for each player, but AoC author asks to keep them confidential.

I don't think it's a big deal: the files weren't very confidential and few people visit my GitHub anyway, but let's solve it. I might need that skill one day for a more serious problem.

This is a two part problem:

- I still like to keep my input files on GitHub. I use several computers and sharing files through the repo is convenient. I just have to encrypt them.
- But first, I need to remove the files I posted. It's not enough to delete them, they would still be available in the git history.

## How to really delete files from git

- You need an extension called **Filter Repo**. On Debian or Ubuntu, you can simply run `sudo apt install git-filter-repo`.
- Make sure the state is as clean as possible: no changes, untracked files, open PRs, nobody pushing other changes.
- Next, get a fresh clone of the repo in a new place - for two reasons. Filter Repo insists on working on a fresh clone. And, you want a copy anyway in case something goes wrong.
- Now, the main part: `git filter-repo --path <path to the file or directory> --invert-paths`. Don't forget the last argument - the default is to keep only files matched by the filter, but you want to delete those and keep everything else. Instead of `--path` you can use `--path-glob` or `--path-regex` to match multiple files. You can also add `--dry-run` to see what files would be deleted. Verify that you deleted everything you wanted to and nothing else.
- Filter Repo also intentionally removes links to remote repositories. There are two ways to deal with it:
    - If the repo is used by many people (or worse, CI systems), the recommended option is to create a new remote repo and inform everyone to switch to it (preferably by cloning a fresh copy).
    - Another possibility is to set the remote repo to the same value as it was before and push the changes (with --force) back to the original upstream repo. Be aware that the files you wanted gone will be deleted from your local copy and from the upstream, but any other copy of the repo will still have them, plus the original, incompatible history. So, all other copies will need to be made clean, either with some heavy git magic, involving rebasing and a lot of --force, or better deleted and cloned again. Since I'm the only user (that I know of), I went with the second option.

The whole set of commands looks like this:

```bash
 git clone git@github.com:igorwaw/advent22.git
 cp -a advent22 advent22.bak # backup
 cd advent22
 ls day*/*-input.txt
 git filter-repo --invert-paths --path-glob 'day*/*-input.txt' --force
 ls day*/*-input.txt # verification - files are gone
 git remote add origin git@github.com:igorwaw/advent22.git
 git pull # to get the remote branches - most of my repos use main, older ones still use master
 git branch --set-upstream-to=origin/main main
 git pull # verification - this is expected to fail as I have incompatible changes
 git push --force # rewrite the remote
 git pull # verification - no incoming changes
```

A quick check on GitHub - the files are gone and there are no new commits, no traces of what just happened. This is scary stuff, a proper removal from history. Luckily, the real world history can't be so easilly rewritten (OR CAN IT?).

## How to transparently encrypt files in the repo

There are millions of ways to encrypt files, but the best solution for this use case is something completely transparent: files are encrypted on *git push* and decrypted on *git pull*.

The tool called **git-crypt** fits perfectly. Easy to get (simple `sudo apt install git-crypt`), fast and secure (uses symmetric encryption with AES-256 by default, multi-user setup with GPG keys is also possible). It has some limitations too:

- By far the biggest limitation: no good mechanism for revoking the key. If you simply change it, only new commits will use it, old versions of files will still be encrypted with the old key. Reencrypting old versions is doable, but complex (see above).
- It doesn't hide any metadata. Filenames, modification dates etc. are plainly visible.
- Some git GUIs fail with git-crypt.

None of these is relevant for me and advantages clearly outweigh the potential issues. I don't need to hide metadata and if the key leaks, I can just delete and recreate the whole repo.

First, go to your repo, generate and export the key:

```bash
git-crypt init
git-crypt export-key ../git-crypt-key
```

Next, create a file `.gitattributes`. It's similar to well-known .gitignore, but informs git to use other transformations than simply ignoring the file. For this case, I needed the following content:

```bash
 day*/*-input.txt filter=git-crypt diff=git-crypt
```

Of course I had to put the input files back. Then, `git-crypt status -e` shows what files will be encrypted:

```bash
➜  advent22 git:(main) ✗ git-crypt status -e
    encrypted: day1-9/1-input.txt
    encrypted: day1-9/2-input.txt
    encrypted: day1-9/3-input.txt
    encrypted: day1-9/4-input.txt
    encrypted: day1-9/5-input.txt
    encrypted: day1-9/6-input.txt
    encrypted: day1-9/7-input.txt
    encrypted: day1-9/8-input.txt
    encrypted: day1-9/9-input.txt
    encrypted: day10-14/10-input.txt
    encrypted: day10-14/11-input.txt
    encrypted: day10-14/12-input.txt
    encrypted: day10-14/13-input.txt
    encrypted: day10-14/14-input.txt
```

You can encrypt them with `git-crypt lock`, but there's no need. Simply add the new files, commit them and push to the remote repo:

```bash
git add .gitattributes day*/*input.txt
git commit -m "encrypted inputs"
git push
```

Now, go to GitHub and see what was commited - the files will look like this: ` GITCRYPT r5”4Â)ùKåæÜ¿„ÚÊ¶`.

### Using the encrypted repo on another machine

Copy the exported keyfile to another machine in a secure way (eg. using `sftp` or `rsync -e ssh`). Place the keyfile in any convenient place outside the repo (one directory level up is convenient), than run the command: `git-crypt unlock ../git-crypt-key`.

### Reusing the encryption key

I'm going to use the same key for all of my Advent of Code repos. No, it's not the same as using the same password for many websites. These are all under my control, just split into several repos for convenience. How do you do that? Simple: the same way as unlocking the repo on another machine in the previous step.

```bash
cd ../advent15
git-crypt unlock ../git-crypt-key
```

There's nothing to unlock yet, but the command also initializes the hidden git-crypt files if they don't exist, only it doesn't create the new keyfile. Confusing, yes. Then, create the `.gitattributes`, add, commit and push files exactly the same way as in the previous repo.

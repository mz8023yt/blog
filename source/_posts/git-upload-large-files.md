---
title: '[Git] 上传大文件到 github 上'
date: 2018-10-04 17:22:03
tags:
---

### 遇到问题

在提交友善之臂提供的内核源码到 github 上的仓库时，遇到问题，无法正常将本地的版本库同步上传到 github 上。详细的报错信息如下：

    user@vmware:~/tiny4412/FriendlyARM.source.code$ git push origin 
    Counting objects: 6, done.
    Compressing objects: 100% (6/6), done.
    Writing objects: 100% (6/6), 112.48 MiB | 2.08 MiB/s, done.
    Total 6 (delta 1), reused 0 (delta 0)
    remote: Resolving deltas: 100% (1/1), done.
    remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.
    remote: error: Trace: 44eb3c8ef0f61d8703f7ccea30ee046b
    remote: error: See http://git.io/iEPt8g for more information.
    remote: error: File linux-3.5-20150929.tgz is 101.86 MB; this exceeds GitHub's file size limit of 100.00 MB
    To git@github.com:tiny4412/FriendlyARM.source.code.git
     ! [remote rejected] master -> master (pre-receive hook declined)
    error: failed to push some refs to 'git@github.com:tiny4412/FriendlyARM.source.code.git'

报错中提示说 `File linux-3.5-20150929.tgz is 101.86 MB; this exceeds GitHub's file size limit of 100.00 MB`，另外自己百度了下，了解到大于 100M 的文件需要用 git lfs 进行管理才能提交到 github。

这里引用博客[Ubuntu 14.04 LTS安装Git LFS](https://www.jianshu.com/p/7d78c68c90d5)中的一段话说明下什么是 git lfs：

> Git LFS 是 git 支持大文件的一个工具。例如，大于 100M 的文件，多了之后用 Git 管理，经常会卡死。从 Git 1.8 之后的版本开始支持。有了 Git LFS，可以大幅度提升对大文件的支持性能。

### 安装 git lfs

访问官网 https://git-lfs.github.com 下载 gz 包，解压执行安装脚本即可。

    user@vmware:~/tiny4412/git-lfs$ ls
    git-lfs-linux-386-v2.5.2.tar.gz
    user@vmware:~/tiny4412/git-lfs$ tar -zxvf git-lfs-linux-386-v2.5.2.tar.gz 
    README.md
    CHANGELOG.md
    git-lfs
    install.sh
    user@vmware:~/tiny4412/git-lfs$ ls
    CHANGELOG.md  git-lfs  git-lfs-linux-386-v2.5.2.tar.gz  install.sh  README.md
    user@vmware:~/tiny4412/git-lfs$ sudo ./install.sh 
    Git LFS initialized.
    user@vmware:~/tiny4412/git-lfs$ git lfs version 
    git-lfs/2.5.2 (GitHub; linux 386; go 1.10.3; git 8e3c5c93)

### 尝试解决

因为安装 Git LFS 需要 Git 的版本不低于 1.8.5，所以安装之前，需要先查看自己的 git 版本。

    user@vmware:~/tiny4412/FriendlyARM.source.code$ git --version
    git version 2.7.4

没问题，我的 git 版本是在 1.8 版本之后，查看下帮助信息，看看 git lfs 怎么用

    user@vmware:~/tiny4412/FriendlyARM.source.code$ git lfs help
    git lfs <command> [<args>]

    Git LFS is a system for managing and versioning large files in
    association with a Git repository.  Instead of storing the large files
    within the Git repository as blobs, Git LFS stores special "pointer
    files" in the repository, while storing the actual file contents on a
    Git LFS server.  The contents of the large file are downloaded
    automatically when needed, for example when a Git branch containing
    the large file is checked out.

    Git LFS works by using a "smudge" filter to look up the large file
    contents based on the pointer file, and a "clean" filter to create a
    new version of the pointer file when the large file's contents change.
    It also uses a pre-push hook to upload the large file contents to
    the Git LFS server whenever a commit containing a new large file
    version is about to be pushed to the corresponding Git server.

    Commands
    --------

    Like Git, Git LFS commands are separated into high level ("porcelain")
    commands and low level ("plumbing") commands.

    High level commands 
    --------------------

    * git lfs env:
        Display the Git LFS environment.
    * git lfs checkout:
        Populate working copy with real content from Git LFS files.
    * git lfs clone:
        Efficiently clone a Git LFS-enabled repository.
    * git lfs fetch:
        Download Git LFS files from a remote.
    * git lfs fsck:
        Check Git LFS files for consistency.
    * git lfs install:
        Install Git LFS configuration.
    * git lfs lock:
        Set a file as "locked" on the Git LFS server.
    * git lfs locks:
        List currently "locked" files from the Git LFS server.
    * git lfs logs:
        Show errors from the Git LFS command.
    * git lfs ls-files:
        Show information about Git LFS files in the index and working tree.
    * git lfs migrate:
        Migrate history to or from Git LFS
    * git lfs prune:
        Delete old Git LFS files from local storage
    * git lfs pull:
        Fetch Git LFS changes from the remote & checkout any required working tree
        files.
    * git lfs push:
        Push queued large files to the Git LFS endpoint.
    * git lfs status:
        Show the status of Git LFS files in the working tree.
    * git lfs track:
        View or add Git LFS paths to Git attributes.
    * git lfs uninstall:
        Uninstall Git LFS by removing hooks and smudge/clean filter configuration.
    * git lfs unlock:
        Remove "locked" setting for a file on the Git LFS server.
    * git lfs untrack:
        Remove Git LFS paths from Git Attributes.
    * git lfs update:
        Update Git hooks for the current Git repository.
    * git lfs version:
        Report the version number.
      
    Low level commands 
    -------------------

    * git lfs clean:
        Git clean filter that converts large files to pointers.
    * git lfs pointer:
        Build and compare pointers.
    * git lfs pre-push:
        Git pre-push hook implementation.
    * git lfs filter-process:
        Git process filter that converts between large files and pointers.
    * git lfs smudge:
        Git smudge filter that converts pointer in blobs to the actual content.
      
    Examples
    --------

    To get started with Git LFS, the following commands can be used.

     1. Setup Git LFS on your system. You only have to do this once per
        repository per machine:

            git lfs install

     2. Choose the type of files you want to track, for examples all ISO
        images, with git lfs track:

            git lfs track "*.iso"

     3. The above stores this information in gitattributes(5) files, so
        that file need to be added to the repository:

            git add .gitattributes

     3. Commit, push and work with the files normally:

            git add file.iso
            git commit -m "Add disk image"
            git push

帮助的最后给出了使用 git lfs 的示例，那么我们就参考这个示例来提交 linux-3.5 源码文件吧

    user@vmware:~/tiny4412/FriendlyARM.source.code$ git lfs track linux-3.5-20150929.tgz
    Tracking "linux-3.5-20150929.tgz"
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git add .gitattributes 
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git push origin 
    Counting objects: 6, done.
    Compressing objects: 100% (6/6), done.
    Writing objects: 100% (6/6), 112.48 MiB | 2.25 MiB/s, done.
    Total 6 (delta 1), reused 0 (delta 0)
    remote: Resolving deltas: 100% (1/1), done.
    remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.
    remote: error: Trace: 3e6e6cee79d8ac960dc2c335eaf5326b
    remote: error: See http://git.io/iEPt8g for more information.
    remote: error: File linux-3.5-20150929.tgz is 101.86 MB; this exceeds GitHub's file size limit of 100.00 MB
    To git@github.com:tiny4412/FriendlyARM.source.code.git
     ! [remote rejected] master -> master (pre-receive hook declined)
    error: failed to push some refs to 'git@github.com:tiny4412/FriendlyARM.source.code.git'

还是没有成功，怎么回事，使用 git lfs 也不行，难道这个 100M 的限制不是 git 工具的限制，而是 github 的限制？还是说我之前已经将 linux-3.5 源码先提交了，导致二次 track 不生效？

目前的提交记录为：

    commit e201f89c2c576c8f1e126ce582ef3b5c30711264
    Author: mz8023yt <mz8023yt@163.com>
    Date:   Thu Oct 4 15:27:51 2018 +0800

        code: add rootfs

    commit 6e1d6f365264da2b933786b59e9def02e5351b6f
    Author: mz8023yt <mz8023yt@163.com>
    Date:   Mon Sep 24 21:51:35 2018 +0800

        code: add kernel

    commit 9b30299a4a969d9b1aab01ba32bb85c1735a5f32
    Author: mz8023yt <mz8023yt@163.com>
    Date:   Sun Sep 23 23:46:35 2018 +0800

        code: add uboot

回退到 9b30299a4a969d9b1aab01ba32bb85c1735a5f32 节点再试一次看看

    user@vmware:~/tiny4412/FriendlyARM.source.code$ mv rootfs-huangweizhong.tar.gz ../
    user@vmware:~/tiny4412/FriendlyARM.source.code$ mv linux-3.5-20150929.tgz ../
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git reset --hard 9b30299a4a969d9b1aab01ba32bb85c1735a5f32
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git log
    commit 9b30299a4a969d9b1aab01ba32bb85c1735a5f32
    Author: mz8023yt <mz8023yt@163.com>
    Date:   Sun Sep 23 23:46:35 2018 +0800

        code: add uboot

回退了两笔，先把 rootfs 那笔先提交掉

    user@vmware:~/tiny4412/FriendlyARM.source.code$ mv ../rootfs-huangweizhong.tar.gz ./
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git st
    On branch master
    Your branch is up-to-date with 'origin/master'.
    Untracked files:
      (use "git add <file>..." to include in what will be committed)

        rootfs-huangweizhong.tar.gz

    user@vmware:~/tiny4412/FriendlyARM.source.code$ git add --all
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git commit -m "code: add the rootfs"
    [master 87fafa1] code: add the rootfs
     1 file changed, 0 insertions(+), 0 deletions(-)
     create mode 100755 rootfs-huangweizhong.tar.gz
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git push origin 
    Counting objects: 3, done.
    Compressing objects: 100% (3/3), done.
    Writing objects: 100% (3/3), 10.86 MiB | 1.34 MiB/s, done.
    Total 3 (delta 0), reused 0 (delta 0)
    To git@github.com:tiny4412/FriendlyARM.source.code.git
       9b30299..87fafa1  master -> master
    user@vmware:~/tiny4412/FriendlyARM.source.code$ clear
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git st
    On branch master
    Your branch is up-to-date with 'origin/master'.
    nothing to commit, working directory clean

再重新试试使用 git lfs 提交 linux-3.5

    user@vmware:~/tiny4412/FriendlyARM.source.code$ mv ../linux-3.5-20150929.tgz ./
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git lfs track linux-3.5-20150929.tgz
    Tracking "linux-3.5-20150929.tgz"
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git add .gitattributes 
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git add linux-3.5-20150929.tgz 
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git commit -m "code: add the kernel"
    [master 943ad9e] code: add the kernel
     2 files changed, 4 insertions(+)
     create mode 100644 .gitattributes
     create mode 100755 linux-3.5-20150929.tgz
    user@vmware:~/tiny4412/FriendlyARM.source.code$ git push origin 
    Counting objects: 4, done.% (1/1), 107 MB | 1.5 MB/s
    Compressing objects: 100% (4/4), done.
    Writing objects: 100% (4/4), 564 bytes | 0 bytes/s, done.
    Total 4 (delta 0), reused 0 (delta 0)
    To git@github.com:tiny4412/FriendlyARM.source.code.git
       87fafa1..943ad9e  master -> master

很好，成功提交到了 github 上

### 简单总结

通过这个问题得到的收获是：

1. 大文件不要直接用 git 管理，即使没有大到 100M 以上，也要用 git lfs 来管理，提高 git 使用的性能。
2. 我们仓库使用了 git lfs 管理，别人克隆是否也需要通过 git lfs clone 克隆，需要做实验验证下。
3. 先提交过的文件，再通过 git lfs track 不生效，需要回退重新 track 后再 commit 才生效。

### 验证实验

使用 git clone 直接克隆仓库，看看是否可以成功

    user@vmware:~/tiny4412$ cd demo/
    user@vmware:~/tiny4412/demo$ git clone git@github.com:tiny4412/FriendlyARM.source.code.git
    Cloning into 'FriendlyARM.source.code'...
    remote: Enumerating objects: 10, done.
    remote: Counting objects: 100% (10/10), done.
    remote: Compressing objects: 100% (10/10), done.
    remote: Total 10 (delta 1), reused 9 (delta 0), pack-reused 0
    Receiving objects: 100% (10/10), 22.01 MiB | 2.44 MiB/s, done.
    Resolving deltas: 100% (1/1), done.
    Checking connectivity... done.
    Downloading linux-3.5-20150929.tgz (107 MB)
    user@vmware:~/tiny4412/demo$ cd FriendlyARM.source.code/
    user@vmware:~/tiny4412/demo/FriendlyARM.source.code$ ls
    linux-3.5-20150929.tgz  rootfs-huangweizhong.tar.gz  uboot_tiny4412-20130729.tgz

使用 git lfs clone 克隆看看

    user@vmware:~/tiny4412$ mkdir demo.lfs
    user@vmware:~/tiny4412$ cd demo.lfs/
    user@vmware:~/tiny4412/demo.lfs$ git lfs clone git@github.com:tiny4412/FriendlyARM.source.code.git
    Cloning into 'FriendlyARM.source.code'...
    remote: Enumerating objects: 10, done.
    remote: Counting objects: 100% (10/10), done.
    remote: Compressing objects: 100% (10/10), done.
    remote: Total 10 (delta 1), reused 9 (delta 0), pack-reused 0
    Receiving objects: 100% (10/10), 22.01 MiB | 872.00 KiB/s, done.
    Resolving deltas: 100% (1/1), done.
    Checking connectivity... done.
    user@vmware:~/tiny4412/demo.lfs$ 1), 107 MB | 1.5 MB/s
    user@vmware:~/tiny4412/demo.lfs$ cd FriendlyARM.source.code/
    user@vmware:~/tiny4412/demo.lfs/FriendlyARM.source.code$ ls
    linux-3.5-20150929.tgz  rootfs-huangweizhong.tar.gz  uboot_tiny4412-20130729.tgz

看起来是两者都可以正常 clone，先记着目前的验证结论，以后有补充的话再补充。




.. title: Switching from git-bigfile to git-lfs
.. slug: switching-from-git-bigfile-to-git-lfs
.. date: 2016-01-30 21:55:32 UTC+01:00
.. tags: git
.. category: git
.. link: 
.. description: 
.. type: text


In 2012, I was looking for a way to store big files in git. git-annex_
was already around, but I found it a bit too complex for my use case.
I discovered git-media_ from Scott Chacon and it looked like what I was looking for.
It was in Ruby which made it not super easy to install on some machines at work.
I thought it was a good exercise to port it to Python. That's how git-bigfile_ was born.
It was simple and was doing the job.

Last year, I was thinking about giving it some love: port it to Python 3,
add some unittests... That's about when I switched from Gogs_
to Gitlab_ and read that Gitlab_ was about to support git-lfs_.

Being developed by GitHub and with Gitlab_ support, git-lfs_ was an
obvious option to replace git-bigfile_.

Here is how to switch a project using git-bigfile_ to git-lfs_:

1. Make a list of all files tracked by git-bigfile::

    $ git bigfile status | awk '/pushed/ {print $NF}' > /tmp/list

2. Edit .gitattributes to replace the filter. Replace `filter=bigfile -crlf` with `filter=lfs diff=lfs merge=lfs -text`::

    $ cat .gitattributes
    *.tar.bz2 filter=lfs diff=lfs merge=lfs -text
    *.iso filter=lfs diff=lfs merge=lfs -text
    *.img filter=lfs diff=lfs merge=lfs -text

3. Remove all big files from the staging area and add them back with git-lfs::

    $ git rm --cached $(cat /tmp/list)
    $ git add .
    $ git commit -m "Switch to git-lfs"

4. Check that the files were added using git-lfs. You should see something
   like that::

    $ git show HEAD
    diff --git a/CentOS_6.4/images/install.img
    b/CentOS_6.4/images/install.img
    index 227ea55..a9cc6a8 100644
    --- a/CentOS_6.4/images/install.img
    +++ b/CentOS_6.4/images/install.img
    @@ -1 +1,3 @@
    -5d243948497ceb9f07b033da62498e52269f4b83
    +version https://git-lfs.github.com/spec/v1
    +oid
    sha256:6fcaac620b82e38e2092a6353ca766a3b01fba7f3fd6a0397c57e979aa293db0
    +size 133255168

5. Remove git-bigfile cache directory::

    $ rm -rf .git/bigfile

Note: to push files larger than 2.1GB to your gitlab server, wait for this
`fix <https://gitlab.com/gitlab-org/gitlab-ce/issues/12745>`_. Hopefully
it will be in 8.4.3.


.. _git-annex: https://git-annex.branchable.com
.. _git-media: https://github.com/schacon/git-media
.. _git-bigfile: https://github.com/beenje/git-bigfile
.. _Gogs: https://gogs.io
.. _Gitlab: https://about.gitlab.com
.. _git-lfs: https://git-lfs.github.com

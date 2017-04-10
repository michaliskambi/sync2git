# Synchronizing SVN -> GIT (1 way)

By default sync2git synchronizes SVN repository -> GIT (1 way).

1. Set up directories and permissions:

   sudo mkdir -p /etc/sync2git/projects
   sudo mkdir -p /var/lib/sync2git

   # Change the ownership of above directories to the user
   # that will run sync2git.
   # Below we call this user 'sync2git' (see the cron instructions
   # below for info how to create such user), you can also just
   # use your current normal username.

   chown sync2git /etc/sync2git/projects /var/lib/sync2git

   # Remember to execute the rest of commands as the user that owns
   # /etc/sync2git/projects and /var/lib/sync2git directories.
   # If this is your current normal user, then don't worry.
   # If this is a special user, try "sudo -u sync2gituser ...".

2. Create a project:

   mkdir -p /etc/sync2git/projects/myproject
   cat > /etc/sync2git/projects/myproject/config << EOF
   SVN_REPO=http://myproject.googlecode.com/svn
   GIT_REPO=git@github.com:myprofile/myproject.git
   EOF

3. Set up the authors.txt file:

   cd /etc/sync2git/projects/myproject
   svn-authors-extract http://myproject.googlecode.com/svn
   vi authors.txt

   When editing the authors.txt file, please remember that you
   won't be able to amend the mappings again after people start
   using your git repository to create forks or branches.  It
   is essential to put the full names and email addresses correctly
   before you make the git mirror available publicly.

   It is OK to go and add extra records to the file later, for example,
   if new users are added to SVN.

   The format of the authors.txt file is
     svn_user_name = FirstName LastName <email.address.used.on.github@example.com>
   Remember that you *must* provide emails.

   Also, if uploading to a service like github, please use the
   email addresses that the committers use for their github accounts.
   This way, the contributors will be recognised properly for their
   work, as github will convert the name next to each commit to a
   hyperlink going to the corresponding github user page.

4. Run it manually (optional)

   sync2git

5. Run it from cron

   Set up a sync2git user and group on the system to own everything:

   groupadd -r sync2git
   useradd -d /var/lib/sync2git -g sync2git -r -s /bin/false sync2git
   chown -R sync2git.sync2git /var/lib/sync2git

   Set up a cron job:

   cat >> /etc/crontab << EOF
   0 * * * * sync2git /usr/local/bin/sync2git > /dev/null
   EOF

   Alternatively: you can use the sync2git-logger script for your cron job.
   This will include somewhat better information to the output (in case of failure),
   and will log more information to the /var/log/sync2git.log (in case of both
   failure and success).
   See the sync2git-logger for more information.

# (Hacky) Synchronizing 2 way: SVN -> GIT and GIT -> SVN

Our fork on https://github.com/michaliskambi/sync2git allows to perform
a 2-way synchronization, so you can commit to any repository (SVN or GIT)
and it gets synchronized to the other one.

Follow the instructions in the previous section (about setting up SVN -> GIT
synchronization) and then set

  COMMIT_GIT_TO_SVN=true

in the repository config file.

Disadvantages: This synchronization sometimes needs to "forcefully push" to GIT,
which overrides GIT commit hashes and authors.
Why? If you commit something directly to the Git repository (by normal pushing,
or by merging someone's pull request), then run the synchronization,
the commits will be applied back to SVN, and then they will be applied back
to GIT --- but with a different hash than original (and a different author,
the one derived from SVN committer, and a little different log,
with "git-svn" comment added). That's how git-svn works by default.

That's either a big or a small problem, depending on how extensively you use
the GIT repository. The GIT authors are overridden, which isn't nice to your
GIT contributors that submit pull requests. More importantly, they need
to carefully do "git pull --rebase", since the GIT commit hashes just totally
changed. If they were in the middle of some new contributions, based on their
previous contributions, and you just applied their previous contributions ---
they will need to reapply their new work on top of new hashes.

# Synchronizing GIT -> SVN (1 way)

Our fork on https://github.com/michaliskambi/sync2git allows to perform
GIT -> SVN synchronization. This way your GIT repository is only read by the
sync2git script (you can even enable GitHub "protected branches" feature),
and the commits from GIT and replicated to the SVN repository.

To configure this behaviour, follow the instructions from the first section
(about setting up SVN -> GIT synchronization) and then set

  COMMIT_GIT_TO_SVN=true
  ONLY_COMMIT_GIT_TO_SVN=true

in the repository config file.

It assumes you will not commit anything to SVN anymore.
Only the sync2git will commit to SVN, everyone else should only push to GIT.
If you break this rule, then the synchronization script may have problems
in case of conflicts.

It works a little differently than other options, using "git svn commit-diff"
under the hood. Multiple GIT commits may be glued into a single SVN commit
(that's how "git svn commit-diff" works by default, some work would be required
to overcome this). An example how this works is in
https://sourceforge.net/p/castle-engine/code/15282/ ,
this SVN commit came from 4 GIT commits:
- https://github.com/castle-engine/castle-game/commit/0b760fb64630d33ddcd899033a24302f4b4c8358 ,
- https://github.com/castle-engine/castle-game/commit/0b3a0c5b7426f0f65fa9cd556b903d36a9c9cc6d ,
- https://github.com/castle-engine/castle-game/commit/ba7b4b6c986bf96887207f9586c5dde7935afa60 ,
- https://github.com/castle-engine/castle-game/commit/15b618e94789b5eef3a295143e17eab9cbef1b2c .

It uses /var/lib/sync2git/xxx/last-committed-revision-to-svn.txt to record last
synchronized GIT commit hash. In each run, it adds newly found GIT commits
as a single SVN commit, constructing nice log for it.

Notes:

- It only works to synchronize GIT "master" to SVN "trunk", and doesn't deal
  with tags or branches at all. It really just works totally differently
  than original approach, much simpler but only 1-way GIT->SVN.

- It requires that some initial files are already set up by running
  the script *without* the "ONLY_COMMIT_GIT_TO_SVN=true" option defined.
  So it assumes that the GIT repository was created from SVN repository,
  and now you want to use GIT for normal work (you want to push to GIT),
  and you only maintain SVN repository to mirror GIT.

  In other words, it's a way to gracefullly migrate from SVN to GIT,
  so you can work with GIT, and your SVN repository is automatically
  updated (mirrors the GIT contents).

  For now, it will not work to fill SVN repository with the initial content
  from GIT (but feel free to submit an issue (or pull request:) if you need this,
  it should be easy to add).

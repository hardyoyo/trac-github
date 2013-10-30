Trac - GitHub integration
=========================

Features
--------

This Trac plugin performs two functions:

- update the local git mirror used by Trac after each push to GitHub, and
  notify the new changesets to Trac;
- replace Trac's built-in browser by GitHub's (optional).

The notification of new changesets is strictly equivalent to the command
described in Trac's setup guide:

    trac-admin TRAC_ENV changeset added ...

This feature set is comparable to https://github.com/davglass/github-trac.

However trac-github has the following advantages:

- it can watch only selected branches in each repository;
- it supports multiple repositories;
- it uses GitHub's generic WebHook URLs;
- it has no external dependencies;
- it is well documented — more docs than code;
- it has an extensive test suite — twice more tests than code;
- it has a much smaller codebase;
- it has better logging;
- it makes better use of Trac's APIs.

Requirements
------------

trac-github requires Trac >= 0.12 and the git plugin.

This plugin [is included](http://trac.edgewall.org/wiki/TracGit) in Trac >=
1.0 — you only have to enable it in `trac.ini`. For Trac 0.12 you have to
[install it](http://trac-hacks.org/wiki/GitPlugin):

    pip install -e git://github.com/hvr/trac-git-plugin.git#egg=TracGit-dev

Then install trac-github itself:

    pip install trac-github

Setup
-----

_Warning: the commands below are provided for illustrative purposes. You'll
have to adapt them to your setup._

First, you need a mirror of your GitHub repository, writable by the webserver,
for Trac's use:

    cd /home/trac
    git clone --mirror git://github.com/<user>/<project>.git
    chown -R www-data:www-data <project>.git

Ensure that the user under which your web server runs can update the mirror:

    su www-data
    git --git-dir=/home/trac/<project>.git remote update --prune

Now edit your `trac.ini` as follows to configure both the git and the
trac-github plugins:

    [components]
    trac.versioncontrol.web_ui.browser.BrowserModule = disabled
    trac.versioncontrol.web_ui.changeset.ChangesetModule = disabled
    trac.versioncontrol.web_ui.log.LogModule = disabled
    tracext.github.* = enabled
    tracopt.versioncontrol.git.* = enabled

    [github]
    repository = <user>/<project>

    [trac]
    repository_dir = /home/trac/<project>.git
    repository_type = git

In Trac 0.12, use `tracext.git.* = enabled` instead of
`tracopt.versioncontrol.git.* = enabled`.

Reload the web server and your repository should appear in Trac.

Browse to the home page of your project in Trac and append `/github` to the
URL. You should see the following message:

    Endpoint is ready to accept GitHub notifications.

This is the URL of the endpoint. Now go to your project's settings page on
GitHub. In the Service Hooks tab, select WebHook URLs, and add the URL of the
endpoint there.

You might want to restrict access to the endpoint to GitHub's IPs. They're
listed just under the WebHook URLs setup form. If you do so, be aware that the
plugin will appear to fail randomly when GitHub adds a new IP to the list.

If you get a Trac error page saying "No handler matched request to /github"
instead, the plugin isn't installed properly. Make sure you've followed the
installation instructions correctly and search Trac's logs for errors.

Branches
--------

By default, trac-github notifies all the commits to Trac. But you may not wish
to trigger notifications for commits on experimental branches until they're
merged, for example.

You can configure trac-github to only notify commits on some branches:

    [github]
    branches = master

You can provide more than one branch name, and you can use [shell-style
wildcards](http://docs.python.org/library/fnmatch):

    [github]
    branches = master stable/*

This option also restricts which branches are shown in the timeline.

Besides, trac-github uses an undocumented feature of GitHub's WebHook to
prevent duplicate notifications when you merge branches.

Multiple repositories
---------------------

If you have multiple repositories, you must tell Trac how they're called on
GitHub:

    [github]
    repository = <user>/<project>               # default repository
    <reponame>.repository = <user>/<project>    # for each extra repository
    <reponame>.branches = <branches>            # optional

When you configure the WebHook URLs, append the name used by Trac to identify
the repository:

    http://<trac.example.com>/github/<reponame>

Private repositories
--------------------

If you're deploying trac-github on a private Trac instance to manage private
repositories, you have to take a few extra steps to allow Trac to pull changes
from GitHub. The trick is to have Trac authenticate with a SSH key referenced
as a deployment key on GitHub.

All the commands shown below must be run by the webserver's user, eg www-data:

    $ su www-data

Generate a dedicated SSH key with an empty passphrase and obtain the public
key:

    $ ssh-keygen -f ~/.ssh/id_rsa_trac
    $ cat ~/.ssh/id_rsa_trac.pub

Make sure you've obtained the public key (`.pub`). It should begin with
`ssh-rsa`. If you're seeing an armored blob of data, it's the private key!

Go to your project's settings page on GitHub. In the Deploy Keys tab, add the
public key.

Edit the SSH configuration for the `www-data` user:

    $ vi ~/.ssh/config

Append the following lines:

    Host github-trac
    Hostname github.com
    IdentityFile ~/.ssh/id_rsa_trac

Edit the git configuration for the repository:

    $ cd /home/trac/<project>.git
    $ vi config

Replace `github.com` in the `url` parameter by the `Host` value you've added
to the SSH configuration:

    url = git@github-trac:<user>/<project>.git

Make sure the authentication works:

    $ git remote update --prune

Since GitHub doesn't allow reusing SSH keys across repositories, you have to
generate a new key and pick a new `Host` value for each new repository.

Advanced use
------------

trac-github provides two components that you can enable separately.

- **`tracext.github.GitHubPostCommitHook`** is the post-commit hook called by
  GitHub.

  It updates the git mirror used by Trac, triggers a cache update and notifies
  components of the new changesets. Notifications are used by Trac's [commit
  ticket updater](http://trac.edgewall.org/wiki/CommitTicketUpdater) and
  [notifications](http://trac.edgewall.org/wiki/TracNotification).

- **`tracext.github.GitHubBrowser`** replaces Trac's built-in browser by
  redirects to the corresponding pages on Github.

  Since it replaces standard URLs of Trac, if you enable this pluign, you must
  disable three components in `trac.versioncontrol.web_ui`, as shown in the
  configuration file above.

Development
-----------

In a [virtualenv](http://www.virtualenv.org/), install the requirements:

    pip install trac
    pip install coverage      # if you want to run the tests under coverage
    pip install -e .

or:

    pip install trac==0.12.4
    pip install -e git://github.com/hvr/trac-git-plugin.git#egg=TracGit-dev

*The version of PyGIT bundled with `trac-git-plugin` doesn't work with
the `git` binary shipped with OS X. To fix it, in the virtualenv, edit
`src/tracgit/tracext/git/PyGIT.py` and replace `_, _, version =
v.strip().split()` with `version = v.strip().split()[2]`.*

Run the tests with:

    ./runtests.py

Display Trac's log during the tests with:

    ./runtests.py --with-trac-log

Run the tests under coverage with:

    coverage erase
    ./runtests.py --with-coverage
    coverage html

If you put a breakpoint in the test suite, you can interact with Trac's web
interface at [http://localhost:8765/](http://localhost:8765/) and with the git
repositories through the command line.

Changelog
---------

### 1.2

* Add support for cached repositories.

### 1.1

* Add support for multiple repositories.
* Add an option to restrict notifications to some branches.
* Try to avoid duplicate notifications (GitHub doesn't document the payload).
* Use GitHub's generic WebHook URLs.
* Use a git mirror instead of a bare clone.

### 1.0

* Public release.

License
-------

This plugin is released under the BSD license.

It was initially written for [Django's Trac](https://code.djangoproject.com/).

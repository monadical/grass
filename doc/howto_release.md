# How to release GRASS GIS binaries and source code

## Assumptions

- You have communicated with other developers, e.g., through grass-dev mailing list.
- You have communicated with a development coordinator.
- You have evaluated status of issues and PRs associated with the relevant milestone.
- You have already cloned the repo with Git.
- The OSGeo repo is the remote called "upstream".
- Your own fork is the remote called "origin" or "fork".
- You don't have any local un-pushed or un-committed changes.
- You are using Bash or a similar shell.

*Note: Some later steps in this text are to be done by the development coordinator
(currently Markus Neteler and Martin Landa) due to needed logins.*

## Prepare the local repo

Update your remotes and switch to branch:

```bash
git fetch --all --prune && git checkout releasebranch_8_4
```

Confirm that you are on the right branch and have no local changes
and that you have no local unpushed commits:

```bash
# Should show no changes:
git status
# Should give no output at all:
git diff
git diff --staged
# Should give no output:
git log upstream/releasebranch_8_4..HEAD
# Should give the same as last commits visible on GitHub:
git log --max-count=5
```

Now you can merge (or rebase) updates from the remote your local branch
and optionally update your own fork:

```bash
git merge upstream/releasebranch_8_4 && git push origin releasebranch_8_4
```

Verify the result:

```bash
# Should give no output:
git log upstream/releasebranch_8_4..HEAD
# Should give the same as last commits visible on GitHub:
git log --max-count=5
```

Now or any time later, you can use `git log` and `git show` to see the latest
commits and the last commit including the changes.

```bash
git log --max-count=5
git show
```

## Update VERSION file to release version number

Modify the VERSION file use the dedicated script, for RC1, e.g.:

```bash
./utils/update_version.py status
./utils/update_version.py rc 1
```

The script will compute the correct version string and print a message
containing it into the terminal (e.g., "version: GRASS GIS 3.5.0RC1").

Commit with a commit message suggested by the script, e.g.:

```bash
git diff
git commit include/VERSION -m "..."
```

If you lost the script output with the suggested message use
`./utils/update_version.py suggest` to get it.

Check that there is exactly one commit on your local branch and that it is the
version change:

```bash
git status
git show
```

Push the tag to the upstream repo:

```bash
git push upstream
```

## Create variables

For convenience, create Bash variables with the version update script:

```bash
# Get VERSION and TAG as variables.
eval $(./utils/update_version.py status --bash)
```

Version and tag are the same for all releases:

```bash
echo "$VERSION"
echo "$TAG"
```

If in doubt, run the script without `eval $(...)` to see all the variables created.

## Create release tag

The tag is created locally, while the release draft is created automatically by
GitHub Actions and can be later edited on GitHub. For background on GitHub releases,
see: <https://help.github.com/en/articles/creating-releases>.

### Tag release

Before creating the tag, it is a good idea to see if the CI jobs are not failing.
Check on GitHub or use GitHub CLI:

```bash
gh run list --branch releasebranch_8_4
```

Some time was needed to run the checks, so before getting back to creating the tag,
confirm that you are on the right branch which is up to date:

```bash
git status
git log --max-count=5
```

Create an annotated tag (a lightweight tag is okay too, but there is more metadata
stored for annotated tags including a date; message is suggested by the version script):

```bash
git tag $TAG -a -m "..."
```

List all tags (annotated will be at the top of both lists):

```bash
git tag -n --sort=-creatordate
git tag -n --sort=-taggerdate
```

Now push the tag upstream - this will trigger the automated workflows linked to tags:

```bash
git push upstream $TAG
```

If any of the tag-related jobs fails, open an issue and see what you can do manually
so that you can continue in the release process.

### Create release notes

Generate a draft of release notes using a script. The script the script needs to
run from the top directory and will expect its configuration files
to be in the *utils* directory.

#### Major and minor releases

For major and minor releases, GitHub API gives good results for the first
release candidate because it contains contributor handles and can identify
new contributors, so use with the *api* backend, e.g.:

```bash
python ./utils/generate_release_notes.py api releasebranch_8_4 8.3.0 $VERSION
```

#### Micro releases

For micro releases, GitHub API does not give good results because it uses PRs
while the backports are usually direct commits without PRs.
The *git log* command operates on commits, so use use the *log* backend:

```bash
python ./utils/generate_release_notes.py log releasebranch_8_4 8.4.0 $VERSION
```

#### RCs

In between RCs and between last RC and final release, the *log* backend is useful
for showing updates since the last RC:

```bash
python ./utils/generate_release_notes.py log releasebranch_8_4 8.4.0RC1 $VERSION
```

#### Finalizing the release notes

For the final release, the changes accumulated since the first RC need to be
added manually to the result from the *api* backend.

The script sorts them into categories defined in *utils/release.yml*.
However, these notes need to be manually edited to collapse related items into
one. Additionally, a *Highlights* section needs to be added with manually
identified new major features for major and minor releases. For all releases, a
*Major* section may need to be added showing critical fixes or breaking changes
if there are any.

### Modify the release draft

After the automated release job completes, a new release draft will be available
in the GitHub web interface. You can copy-paste the created release notes to
GitHub and further modify as needed.

Older release description may or may not be a good inspiration:
<https://github.com/OSGeo/grass/releases>.

If RC, mark it as a pre-release, check:

```text
[x] This is a pre-release
```

Save the modified draft, but do not publish the release yet.

## Reset include/VERSION file to git development version

Use the dedicated `update_version.py` script to edit the VERSION file.

After an RC, switch to development version:

```bash
./utils/update_version.py dev
```

Next switch back to the development version for the next micro, minor,
or major version, e.g., for micro version, use:

```bash
./utils/update_version.py micro
```

Use *major* and *minor* operations for the other version updates.
Use `--help` for details about the options.

Commit with the suggested commit message and push, e.g.:

```bash
git show
git commit include/VERSION -m "..."
git push upstream
```

The message was suggested by the script, but if you lost that output,
you can get the same or similar message again using the script
(the message provided this way is not precise after RCs):

```bash
./utils/update_version.py suggest
```

## Publish release

For the final release, edit the draft release again and publish it using the
"Publish release" button.

## Upload to OSGeo servers

This part requires extra permissions and needs to be done by one of the
development coordinators.

### Get the tagged version

For the automation, the tagged version of the source code is needed.
First, update the repo to get the tag locally:

```bash
git fetch upstream
```

Get the tagged source code, e.g. (modify the tag as needed):

```bash
git checkout 8.4.0RC1
```

Create the Bash variables for version numbers:

```bash
eval $(./utils/update_version.py status --bash)
```

Confirm the version (it should match exactly the tag specified above):

```bash
echo "$VERSION"
```

### Get changelog file for upload

There is also a large changelog file we produce and publish,
fetch the file from the release which was generated by a workflow
linked to the tag:

```bash
wget https://github.com/OSGeo/grass/releases/download/${VERSION}/ChangeLog.gz \
    -O ChangeLog_${VERSION}.gz
```

### Get the source code tarball

Fetch a tarball from GitHub we also publish on OSGeo servers:

```bash
wget https://github.com/OSGeo/grass/archive/${VERSION}.tar.gz -O grass-${VERSION}.tar.gz
md5sum grass-${VERSION}.tar.gz > grass-${VERSION}.md5sum
```

### Upload source code tarball to OSGeo servers

Note: servers 'osgeo7-grass' and 'osgeo7-download' only reachable via
      jumphost (managed by OSGeo-SAC) - see <https://wiki.osgeo.org/wiki/SAC_Service_Status#grass>

```bash
# Store the source tarball (twice) in (use scp -p FILES grass:):
USER=neteler
SERVER1=osgeo7-grass
SERVER1DIR=/var/www/code_and_data/grass$MAJOR$MINOR/source/
SERVER2=osgeo7-download
SERVER2DIR=/osgeo/download/grass/grass$MAJOR$MINOR/source/
echo $SERVER1:$SERVER1DIR
echo $SERVER2:$SERVER2DIR

# upload along with associated files:
scp -p grass-$VERSION.* AUTHORS COPYING ChangeLog_$VERSION.gz \
  INSTALL.md REQUIREMENTS.md CONTRIBUTING.md $USER@$SERVER1:$SERVER1DIR

scp -p grass-$VERSION.* AUTHORS COPYING ChangeLog_$VERSION.gz \
  INSTALL.md REQUIREMENTS.md CONTRIBUTING.md $USER@$SERVER2:$SERVER2DIR

# Only at full release (i.e., not for RCs)!
# generate link to "latest" source code
ssh $USER@$SERVER1 "cd $SERVER1DIR ; rm -f grass-$MAJOR.$MINOR-latest.tar.gz"
ssh $USER@$SERVER1 "cd $SERVER1DIR ; ln -s grass-$VERSION.tar.gz grass-$MAJOR.$MINOR-latest.tar.gz"
ssh $USER@$SERVER1 "cd $SERVER1DIR ; rm -f grass-$MAJOR.$MINOR-latest.md5sum"
ssh $USER@$SERVER1 "cd $SERVER1DIR ; ln -s grass-$VERSION.tar.md5sum grass-$MAJOR.$MINOR-latest.md5sum"

# verify
echo "https://grass.osgeo.org/grass$MAJOR$MINOR/source/"
```

### Update redirects

For final minor and major releases (not release candidates and micro releases),
update `grass-stable` redirect at `osgeo7-grass`:

```bash
sudo vim /etc/apache2/sites-enabled/000-default.conf`
```

Load the new configuration:

```bash
sudo systemctl reload apache2`
```

For new branches: Update `grass-devel` using the steps above.

## Update winGRASS related files

Update the GRASS version at <https://github.com/landam/wingrass-maintenance-scripts>:

- On major or minor version change (on release branch creation) update
  [dev_packages.csv](https://github.com/landam/wingrass-maintenance-scripts/blob/master/dev_packages.csv)
- On release (inluding RC) update
  [releases.csv](https://github.com/landam/wingrass-maintenance-scripts/blob/master/releases.csv)
  and
  [grass_packager_release.bat](https://github.com/landam/wingrass-maintenance-scripts/blob/master/grass_packager_release.bat#L12)

Example for 8.3.0RC1: [commit](https://github.com/landam/wingrass-maintenance-scripts/commit/c47b0f30051108bd2e8b52d183e97930c24dfafd)

## Update binary and addon builders

Add the new version to repos which build or test addons:

- <https://github.com/OSGeo/grass-addons/blob/grass8/.github/workflows/ci.yml>
  (currently, for new branches only)
- <https://github.com/landam/wingrass-maintenance-scripts/blob/master/grass_addons.sh>
  (add new release related line for new branches and final releases)

## Close milestone

For a final release (not release candidate), close the related milestone at
<https://github.com/OSGeo/grass/milestones>.
If there are any open issues or PRs, move them to another milestone
in the milestone view (all can be moved at once by selecting the open
issues and PRs and reassigning the next milestone).

## Publish the release

When the above is done and the release notes are ready, publish the release at
<https://github.com/OSGeo/grass/releases>.

Release is done.

## Improve release description

For final releases only, go to Zenodo.org a get a Markdown badge for the release
which Zenodo creates with a DOI for the published release.

For all releases, click the Binder badge to get Binder to build. Use it to test
it and to cache the built image. Add more links to (or badges for) more notebooks
if there are any which show well specific features added or updated in the release.

## Create entries for the new release

### Trac Wiki release page entry

Add entry in <https://trac.osgeo.org/grass/wiki/Release>

### Update Hugo web site and other pages to show the new version

For a (final) release (not release candidate), write announcement and publish it:

- News section, <https://github.com/OSGeo/grass-website/tree/master/content/news>

Increment the GRASS GIS version in

- <https://github.com/OSGeo/grass-website/blob/master/data/grass.json>
- <https://github.com/OSGeo/grass-website/blob/master/content/about/history/releases.md>

Update the version in the Wiki page: <https://grasswiki.osgeo.org/wiki/GRASS-Wiki>

Subsequently, verify the software pages:

- Linux: <https://github.com/OSGeo/grass-website/blob/master/content/download/linux.en.md>
- Windows: <https://github.com/OSGeo/grass-website/blob/master/content/download/windows.en.md>
- Mac: <https://github.com/OSGeo/grass-website/blob/master/content/download/mac.en.md>

### Only in case of new major release

- update '[cronjob(s)](https://github.com/OSGeo/grass-addons/tree/grass8/utils/cronjobs_osgeo_lxd/)'
  on grass.osgeo.org to next but one release tag for the differences
- wiki updates, only when new major release:
  - {{cmd|xxxx}} macro: <https://grasswiki.osgeo.org/wiki/Template:Cmd>
  - update last version on main page
- Add trac Wiki Macro definitions for manual pages G8X:modulename
  - Edit: <https://trac.osgeo.org/grass/wiki/InterMapTxt>

## Packaging notes

### WinGRASS notes

- Go to <https://github.com/landam/wingrass-maintenance-scripts/>
- Update grass_packager_release.bat, eg.

```bash
     set MAJOR=8
     set MINOR=4
     set PATCH=0RC1
```

- Update addons (grass_addons.sh) rules, eg.

```bash
     compile $GIT_PATH/grass8 $GISBASE_PATH/grass840RC1  $ADDON_PATH/grass840RC1/addons
```

- Modify grass_copy_wwwroot.sh accordingly, eg.

```bash
     copy_addon 840RC1 8.4.0RC1
```

### Ubuntu Launchpad notes

- Create milestone and release: <https://launchpad.net/grass/+series>
- Upload tarball for created release

### Other notes

- <https://trac.osgeo.org/grass/wiki/BuildHints>
  - <https://trac.osgeo.org/grass/wiki/DebianUbuntuPackaging>
  - <https://trac.osgeo.org/grass/wiki/CompileOnWindows>

## Tell others about release

- If release candidate (send just a short invitation to test):
  - <grass-announce@lists.osgeo.org>
  - <grass-dev@lists.osgeo.org>

If final release, send out an announcement (press release) which is a shortened
version of release desciption and website news item (under `/announces/`).
Note: Do not use relative links.

- Our main mailing lists:
  - <https://lists.osgeo.org/mailman/listinfo/grass-announce> | <grass-announce@lists.osgeo.org>
    (ask a development coordinator to be added)
  - <https://lists.osgeo.org/mailman/listinfo/grass-dev> | <grass-dev@lists.osgeo.org>
  - <https://lists.osgeo.org/mailman/listinfo/grass-user> | <grass-user@lists.osgeo.org>
- OSGeo.org: <news_item@osgeo.org>, <info@osgeo.org> (send an email, then it
  will be approved)

Via web and social media:

- See: <https://grass.osgeo.org/wiki/Contact_Databases>

## Update VERSION file to next version number

After the final release whole is done, modify the VERSION file use
the dedicated script, e.g., for next micro version, run:

```bash
./utils/update_version.py micro
./utils/update_version.py status
```

Now commit the change to the branch with the commit message generated above
by the script:

```bash
git diff
git commit include/VERSION -m "..."
```

If you lost the script output with the suggested message use
`./utils/update_version.py suggest` to get it.

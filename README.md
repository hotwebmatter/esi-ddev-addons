[![tests](https://git.express-scripts.com/ExpressScripts/esi-ddev-addons/actions/workflows/tests.yml/badge.svg)]([https://github.com/ddev/ddev-addon-template/actions/workflows/tests.yml](https://git.express-scripts.com/ExpressScripts/esi-ddev-addons/actions/workflows/tests.yml)) ![project is maintained](https://img.shields.io/maintenance/yes/2024.svg)

# esi-ddev-addons <!-- omit in toc -->

* [What is the esi-ddev-addons project?](#what-is-the-esi-ddev-addons-project)
* [Components of the repository](#components-of-the-repository)
* [Getting started](#getting-started)
* [How to debug in Github Actions](#how-to-debug-tests-github-actions)

## What is the esi-ddev-addons project?

This repository is a template for providing [DDEV](https://ddev.readthedocs.io) add-ons and services.

In DDEV addons can be installed from the command line using the `ddev get` command, for example, `ddev get ddev/ddev-redis` or `ddev get ddev/ddev-solr`.

It's important to note that DDEV addons must be hosted in a public repository on GitHub.com in order to fully enable all functionality of the `ddev get` command. 

```
$ ddev get --help
Get/Download a 3rd party add-on (service, provider, etc.). This can be a GitHub repo, in which case the latest release will be used, or it can be a link to a .tar.gz in the correct format (like a particular release's .tar.gz) or it can be a local directory. Use 'ddev get --list' or 'ddev get --list --all' to see a list of available add-ons. Without --all it shows only official DDEV add-ons. To list installed add-ons, 'ddev get --installed', to remove an add-on 'ddev get --remove <add-on>'.

Usage:
  ddev get <addonOrURL> [project] [flags]

Examples:
ddev get ddev/ddev-redis
ddev get ddev/ddev-redis --version v1.0.4
ddev get https://github.com/ddev/ddev-drupal9-solr/archive/refs/tags/v0.0.5.tar.gz
ddev get /path/to/package
ddev get /path/to/tarball.tar.gz
ddev get --list
ddev get --list --all
ddev get --installed
ddev get --remove someaddonname,
ddev get --remove someowner/ddev-someaddonname,
ddev get --remove ddev-someaddonname


Flags:
      --all              List unofficial add-ons for 'ddev get' in addition to the official ones (default true)
  -h, --help             help for get
      --installed        Show installed ddev-get add-ons (default true)
      --list             List available add-ons for 'ddev get' (default true)
      --remove string    Remove a ddev-get add-on
  -v, --verbose          Extended/verbose output for ddev get
      --version string   Specify a particular version of add-on to install

Global Flags:
  -j, --json-output   If true, user-oriented output will be in JSON format.
```

As shown above, Git repositories hosted elsewhere may be specified by HTTP links. However, this does not work correctly on `git.express-scripts.com` because we require LDAP authentication even for public repositories.

Because of these restrictions, the only way that we can automate installation in our environment is to install from a local directory or tarball.

The `ddev local-update` command checks out the latest contents of the project repo into a staging directory, then ateempts to install ESI DDEV addons from this local directory. By default, it will ask for interactive confirmation before installing. Use `ddev local-update --help` for usage info:

```
$ ddev local-update --help
local-update [ check || apply || interactive || --help ]
Usage: 'ddev local-update' (or specify 'check', 'apply', or 'interactive')
       Download and install latest updates to ESI DDEV Addons.

       '--help' displays this usage info.
       'check' checks for updates without applying.
       'apply' checks for updates and applies them if found.
       'interactive' checks for updates and asks user whether to apply.

       When called without arguments, local-update runs interactively.
```

## Components of the repository

* The fundamental contents of the add-on service or other component. This add-on manages several custom commands.
* An [install.yaml](install.yaml) file that describes how to install the service or other component.
* A test suite in [test.bats](tests/test.bats) that makes sure the service continues to work as expected (this is very minimal so far).
* [Github actions setup](.github/workflows/tests.yml) so that the tests run automatically when you push to the repository (untested).

## Getting started

To add ESI DDEV Addons to an ExpressScripts Drupal project, look at [local-setup](https://git.express-scripts.com/ExpressScripts/esi-ddev-addons/blob/develop/commands/host/local-setup) and [local-update](https://git.express-scripts.com/ExpressScripts/esi-ddev-addons/blob/develop/commands/host/local-update).

You will need to perform some of these actions manually to bootstrap the process for future auto-updates.

First, change directory to the project root of the project you're editing.

Then, set the following variables for your copy/paste convenience.

```bash
export PROJECT_ROOT="${PWD}" 
export STAGING_DIR='.esi-ddev-addon-staging'
export ESI_ADDON_REPO='git@git.express-scripts.com:ExpressScripts/esi-ddev-addons'
export ESI_MANIFEST='.ddev/addon-metadata/esi-ddev-addons/manifest.yaml'
```

After setting environment variables, the following snippets should execute without error in your local.

You'll need to clone the latest ESI DDEV Addons to a staging directory:

```bash
echo "Remove local staging directory if found ..."
[[ -d $STAGING_DIR ]] && rm -rf ${STAGING_DIR} || echo "... staging directory not found."

echo "Clone ESI DDEV Addons repo into local staging directory ..."
git clone --depth 1 "${ESI_ADDON_REPO}" "${STAGING_DIR}"
```

The automated installer does not run this next step, since it honors the DDEV convention of skipping locally-modified files.

```bash
echo "DDEV won't update files unless they contain #ddev-generated"
rsync --progress -av "${STAGING_DIR}/commands/" "${PROJECT_ROOT}/.ddev/commands/"
```

Omit the next step if you'd prefer to edit your .gitignore manually.

```bash
echo "Checking your .gitignore ..."
if grep -q "${STAGING_DIR}" .gitignore; then
    echo 'DDEV Addon staging directory path found in .gitignore file.'
else if grep -q "${ESI_MANIFEST}" .gitignore; then
    echo 'ESI manifest file path found in .gitignore file.'
else
    cat <<EOF >> .gitignore

# Added by ddev local-update on $(date +%Y-%m-%d)
${STAGING_DIR}
${ESI_MANIFEST}
EOF

fi
```

We are almost ready to run `ddev local-update`.

But first, we need to edit `config.yml` in our favorite `$EDITOR`:

```
vim "${PROJECT_ROOT}/.ddev/config.yaml"
```

In our project's `config.yaml` file, we need to add the following `pre-install` and `post-install` hooks:

```yaml
hooks:
    pre-start:
        - exec-host: ddev local-setup stage
        - exec-host: ddev local-update apply
```

```yaml
    post-start:
        - exec: /var/www/html/.ddev/commands/web/esi-version update
```

I recommend adding the `esi-version update` step at the very end of your `post-start` hooks, though that may not be necessary.

After making all these manual changes once, you may simply restart (or start) DDEV to install the latest ESI DDEV Addons:

```
ddev restart
```

Or, if DDEV is running, you may wish to install without restarting:

```
ddev local-setup stage && ddev local-update apply && ddev esi-version update
```

Future releases may require manual updates, especially if they require changes to `config.yaml` or `.gitignore`.

## Further reading

Add-ons were covered in [DDEV Add-ons: Creating, maintaining, testing](https://www.dropbox.com/scl/fi/bnvlv7zswxwm8ix1s5u4t/2023-11-07_DDEV_Add-ons.mp4?rlkey=5cma8s11pscxq0skawsoqrscp&dl=0) (part of the [DDEV Contributor Live Training](https://ddev.com/blog/contributor-training)).

Note that more advanced techniques are discussed in [DDEV docs](https://ddev.readthedocs.io/en/latest/users/extend/additional-services/#additional-service-configurations-and-add-ons-for-ddev).

## How to debug tests (Github Actions)

For documentation on automated testing, see the original README for [ddev/ddev-addon-template](https://github.com/ddev/ddev-addon-template). This functionality does not appear to work properly on our private GitHub Enterprise server, but debugging may be possible.

**Contributed and maintained by [@c7x4gw](https://git.express-scripts.com/c7x4gw)**

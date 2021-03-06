# Exemplary package: GUI (LinacGUI)

This was the followed workflow for creating the linacgui package:

## C1. [Get the debpack Docker](recipe.Get_the_debpack_Docker.md)

## C2. [Get the python module upstream source and cd into it](recipe.Get_the_python_module_upstream_source.md)

LinacGUI project is in git.cells, so `git clone` command was used.
 
```
git clone https://git.cells.es/controls/LinacGUI.git
cd LinacGUI
```

## C3. [Generate the debian source with setuptools](recipe.Generate_the_debian_source_with_setuptools.md)

The linacgui debian sources were generated with:

```
python setup.py --command-packages=stdeb.command sdist_dsc \
        --upstream-version-suffix +git20170314.1.8d4063 \
        --debian-version 0~bpo9+0~alba+1 \
        --build-depends "dh-exec, help2man, python-taurus" \
        --depends "python-taurus, python-tango" \
        --suite stretch-backports

```

LinacGUI depends on Taurus and PyTango libraries and requieres Taurus at building 
time.

As this packages has to be installed in `/usr/share` it requires `dh-exec` 
package as build-dep and `help2man` is required  as build-dep
for the autimatic generating of the man pages.

This package was created from an untagged commit, so the upstream version 
has to reflect it. It is done with `--upstream-version-suffix`. See [Appendix 3](Appendix_3.md) 
if you have any doubt about how to compose it. 

## B3. [Create the local gbp repo with `gbp import-dsc`](recipe.Create_the_local_gbp_repo_with_gbp_import-dsc.md)

The local gbp repository was created with:

```
gbp import-dsc deb_dist/linacgui_2.70.0.post0+git20170314.1.8d4063-0~bpo9+0~alba+1.dsc \
              /packaging/linacgui_deb --pristine-tar
cd /packaging/linacgui_deb
# add the remote (the remote itself will actually be created next)
git remote add origin https://git.cells.es/ctpkg/linacgui_deb.git
```
 
## B4. [Create the remote git repo](recipe.Create_the_remote_git_repo.md)

The following command was used to create the linacgui_deb repository in 
`ctpkg` group of `git.cells` 

```
create_ctpkg_project linacgui_deb "Repo for packaging LinacGUI on debian" \
                                  "application, GUI, Linac, python, mrosanes, cfalcon"
```

## A4. [Edit the Debian Folder Files as needed](recipe.Edit_the_Debian_Folder_Files.md)

In a previous step, debian files were auto-generated (you can see the 
[original commit](https://git.cells.es/ctpkg/linacgui_deb/commit/ab83d2cecb94dc5a6cc9b0ee237ff58023ddd949#9c96da0e9f91d7d8937b69b524702c106258f0d1),
but they are only a starting point. To begin with, they assume that the application will be installed as a python 
module, which is not the case for the linacGUI which is treated as an application 
(since its python modules are not expected to be imported externally). This required several changes in the debian files.
Apart from that, some other customizations were done (man page autogeneration, 
etc.). In this point we explain the rationale behind each change to the debian files folder.

**Important** Note that the [final result at the moment of writing](https://git.cells.es/ctpkg/linacgui_deb/blob/c2de5857820c155d5c5168f009e6793b8f0cdf88/debian/) 
only came after a long iterative process of building the package and fixing the errors reported by linitan (or detected on installation)

* [rules](https://git.cells.es/ctpkg/linacgui_deb/blob/c2de5857820c155d5c5168f009e6793b8f0cdf88/debian/rules)
file was modified for various things:
  - to change the behavior of `dh_auto_install` in order to install in 
  `/usr/share/linacgui` (instead of in the default python path) and to not 
  generate `.pyc` files. Both changes were done adding the `PYBUILD_INSTALL_ARGS`
  variable: `export PYBUILD_INSTALL_ARGS=--install-lib=/usr/share/linacgui --no-compile`
  - The `rules` file was also modified to [autogenerate the man pages during the 
  package creation](recipe.Autogenerating_manpages.md).
  This accounts for the addition of the `MANS` and `PYTHONPATH` variables, the 
  creation of the `%.1:` target and the override of `dh_clean` and `dh_installman`.
  - A problem with permissions in the egg file contents (reported by lintian) 
  was fixed by overriding `dh_fixperms`.
  - On [a later commit](https://git.cells.es/ctpkg/linacgui_deb/commit/716b38cc611f32287bd16fb97309a7c523330103), 
  an extra override of the `dh_python2` rule was added to allow this GUI to be 
  run under a virtualenv (needed for compatibility with the taurus3 package)

* [copyright](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/copyright) is needed to pass the lintian checks.

* [linacgui.install](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/linacgui.install): 
used to relocate launchers and for installing files would not be automatically installed (e.g. the files in the `distribute` upstream folder).
The first two lines of this file are for renaming and relocating the `ctli` launcher. 
The launcher has to be moved because it needs to import the linacgui python modules but these won't be in the PYTHONPATH. 
The launcher is renamed to avoid a collision with the module name.
Note that regular `.install` files do not allow renaming of files, but we do it 
using the extended syntax provided by `dh-exec`
```
1 #!/usr/bin/dh-exec 
2 debian/linacgui/usr/bin/ctli => /usr/share/linacgui/ctli_ 
```
The fourth line is an example of how to install other files (not installed by dh_auto_install), in this case the `ctli.desktop` file.
```
4 distribute/ctli.desktop /usr/share/applications
```
**Note :** Since we use `dh-exec`-extended syntax in`linacgui.install` we must give it 
execution permissions (`chmod 777 debian/linacgui.install`)

* [linacgui.links](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/linacgui.links): 
Is used to create sym-links for the launchers in `/usr/bin`.

* [linacgui.postinstall](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/linacgui.postinstall): 
Is used to generate the *.pyc files as post install action (it uses shell syntax).

*  [linacgui.prerm](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/linacgui.prerm): 
Is used to remove the generated `pyc` files before uninstall action (it uses shell syntax).

* [watch](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/watch)
Is used to automate upstream updates (uses upstream tags)

* [linacgui.manpages](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/linacgui.manpages): 
File to list to the autogenerated man pages

* [help2man](https://git.cells.es/ctpkg/linacgui_deb/blob/master/debian/help2man). 
Is a generic man header used during the autogeneration of the man pages.


## A5. [Build package](recipe.Build_package.md)

## A6. [Update Changelog and tag](recipe.Update_changelog_and_tag.md)

The following commands were executed.
```
gbp dch --release --commit --since 497173e28
gbp buildpackage
```

And after checking that the `gbp dch` did not introduce any lintian error.
```
gbp buildpackage --git-tag-only 
git push --all
git push --tags

```

## A7. [Upload artifacts](recipe.Upload_artifacts.md)

-------------------------------------------------------------------------------

> **Note**:
> You can avoid the most common lintian errors checking this [list](recipe.Avoid_typical_lintian_errors.md)


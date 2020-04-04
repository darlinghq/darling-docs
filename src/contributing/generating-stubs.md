# Generating stubs

Darling has a stub generator that is capable of generating stubs for C and
Objective-C frameworks and shared libraries. A computer running macOS is
required to run the stub generator.

## Preparations

You don't need to do this step if you already have a `bin` folder in your home
directory, with the `PATH` variable pointing to it. If not, copy/paste the
following commands into Terminal.

Create the `bin folder if it doesn't exist:

```
$ mkdir ~/bin
```

If you `PATH` variable does not include the `bin` folder, you will need to add it.

```
# For bash
$ echo "export PATH=\"~/bin:\$PATH\"" >> ~/.bash_profile && source ~/.bash_profile
# For zsh
% echo "export PATH=\"\$HOME/bin:\$PATH\"" >> ~/.zshenv && source ~/.zshenv
```

## Getting the stub generator

Copy/paste the following command into Terminal. It will download both
`darling-stub-gen` and `class-dump` and place it in the `bin` folder

```
$ curl https://raw.githubusercontent.com/darlinghq/darling/master/tools/darling-stub-gen -o ~/bin/darling-stub-gen && chmod +x ~/bin/darling-stub-gen && curl https://github.com/darlinghq/class-dump/releases/download/mojave/class-dump -L -o ~/bin/class-dump && chmod +x ~/bin/class-dump
```

## Using the stub generator

To run the stub generator, structure your arguments like this:

```
$ darling-stub-gen /System/Library/Frameworks/DVDPlayback.framework/DVDPlayback DVDPlayback
```

The process is identical for dynamic libraries.

The above command will create a folder that can be placed in the
`src/frameworks/` directory of Darling's source tree. It is generated from the
DVDPlayback framework. Note that the first argument points to the actual binary
of the framework, not the root directory of the framework.

## Applying the stubs to Darling

Once you have generated the stub folder for the framework, copy that folder into
Darling's source tree under `src/frameworks/`.

Then traverse to the `src/frameworks/include/` directory (also located inside
Darling's source tree) and create a soft symbolic link. The link should point to
the folder inside the include directory (ex: `MyNewFolder/include/MyNewFolder`).

Example:

```
$ cd src/frameworks/include/
$ ln -s ../MyNewFolder/include/MyNewFolder MyNewFolder`
```

Finally, you will need to add the folder to the build. In
`src/frameworks/CMakeLists.txt`, add the following line:
`add_subdirectory(MyNewFolder)`. Make sure you put it in alphabetical order.

Run a build and make sure your new code compiles. After that completes, you are
ready to submit a pull request.

See [Contributing](README.md) for how to submit a pull request. [This
commit](https///github.com/darlinghq/darling/commit/92233d4e5ca613658345910d1acf4b3b7620a4f6)
is an example of a stub for a framework that was added to Darling using the
process described in this article. Most notable is what it does to
`src/CMakeLists.txt`.

# Known issues

* The stub generator does not currently generate symbols for constants. Those
  must be manually added if a program needs them.
* Generating stubs for platforms outside of x86 (macOS, iOS Simulator) is not
  supported.
* **TODO**: Figure out how to generate stubs from a `dyld_shared_cache` file.

# Generating stubs

Darling has a stub generator that is capable of generating stubs for C and
Objective-C frameworks and shared libraries. A computer running macOS is
required to run the stub generator.

## Preparations

You don't need to do this step if you already have a `bin` folder in your home
directory, with the `PATH` variable pointing to it. If not, copy/paste the
following commands into Terminal.

Create the `bin` folder if it doesn't exist:

```bash
mkdir ~/bin
```

If your `PATH` variable does not include the `bin` folder, you will need to add it.

```bash
# For bash
echo "export PATH=\"~/bin:\$PATH\"" >> ~/.bash_profile && source ~/.bash_profile
# For zsh
echo "export PATH=\"\$HOME/bin:\$PATH\"" >> ~/.zshenv && source ~/.zshenv
```

## Getting the stub generator

Copy/paste the following command into Terminal. It will download both
`darling-stub-gen` and `class-dump` and place it in the `bin` folder

```bash
curl https://raw.githubusercontent.com/darlinghq/darling/master/tools/darling-stub-gen -o ~/bin/darling-stub-gen && chmod +x ~/bin/darling-stub-gen && curl https://github.com/darlinghq/class-dump/releases/download/mojave/class-dump -L -o ~/bin/class-dump && chmod +x ~/bin/class-dump
```

## Using the stub generator

To run the stub generator, structure your arguments like this:

```bash
darling-stub-gen /System/Library/Frameworks/DVDPlayback.framework/DVDPlayback DVDPlayback
```

The process is identical for dynamic libraries.

The above command will create a folder that can be placed in the either the
`src/frameworks/` or `src/private-frameworks/` directory of Darling's source tree. Note that the first argument points to the actual binary
of the framework, not the root directory of the framework.

## Applying the stubs to Darling

Once you have generated the stub folder for the framework, copy that folder into
Darling's source tree. If the framework is public, put it in `src/frameworks/`. If the framework is private put it in `src/private-frameworks/`.

After you add in the folder, you will need to include it in the  build. In
`src/frameworks/CMakeLists.txt` (or `src/private-frameworks/CMakeLists.txt` if the framework is private),
 add the following line: `add_subdirectory(MyNewFolder)`. Make sure you put it in alphabetical order.

To generate the SDK headers, make sure that you set `REGENERATE_SDK` to `ON` when you run the `cmake` command (ex: `cmake .. -DREGENERATE_SDK=ON`).

Run a build and make sure your new code compiles. After that completes, you are
ready to submit a pull request.

See [Contributing](index.md) for how to submit a pull request. [This
pull request](https://github.com/darlinghq/darling/pull/1199/files)
is an example of a stub for a framework that was added to Darling using the
process described in this article. Most notable is what it does to
`src/CMakeLists.txt`.

# Known issues

* The stub generator does not currently generate symbols for constants. Those
  must be manually added if a program needs them.
* Generating stubs for platforms outside of x86 (macOS, iOS Simulator) is not
  supported.
* **TODO**: Figure out how to generate stubs from a `dyld_shared_cache` file.

Bazel build system (github.com/bazelbuild/bazel/) packaged as an SDK extension.
Current manifest just downloads pre-built binaries from GitHub.
If you're on an architecture other than `x86_64` or `arm64`, have a look in org.freedesktop.Sdk.Extension.bazel.yaml.old -- it contains some half-baked build instructions which you might be able to use.

To build things using bazel, add the following to your manifest file:

```yaml
add-build-extensions:
  - org.freedesktop.Sdk.Extension.bazel
build-options:
  append-path: /usr/lib/sdk/bazel/bin
```

To build with flatpak without network access, you may use the new feature introduced with bazel 7.

Bazel 7 introduced a new flag `--repository_cache` which allows you to download or read any external dependency with the given directory.

In order to trigger the dependency download for the target you need, one may use the following process to get the external dependencies.

1. Remove any bazel cache from disk, usually it is located under `$HOME/.cache/bazel`. If the cache exists, the download process will not be triggered.

1. Run `bazel cquery` to trigger the downloading process of external dependencies.
    ```bash
    bazel cquery --repository_cache="/path/to/cache" [--config config_name] //path/to/target:name
    ```
   The reason that `bazel cquery` is a preferred way of triggering the dependency fetching over `bazel fetch` is that, `bazel cquery` may accept `--config` flag just like `bazel build`. While `bazel fetch` does not accept such flag. This is very handy if the project is cross-platform and only buildable under certain configuration for Linux.

1. Inspect the repository cache directory, there are two options to use it with flatpak.

   - Create a tarball for the repository cache directory, then extract from it in your manifest file.

   - Add all external dependency files to sources in the manifest file.

     The structure of the repository cache directory should look like:

     ```
     content_addressable/sha256/
       |- [sha256 of file]
       |  |- id-[sha256 of urls] # space separated list of urls
       |  |- file # The actual file
       |- ...
     ```

     You may use the information above in the repository cache directory, to either replicate the whole repository cache directory with the manifest file, or uses a old flag `--distdir` that is deprecated in bazel 7 in favor of --repository_cache. Though `--distdir` is deprecated, it is still supported by bazel 7 and the required directory structure for `--distdir` is simpler than repository cache directory, which only needs to contain the downloaded files.

     To organize all the files in distdir, add the following to your manifest file:

     ```yaml
     build-commands:
       - bazel build --distdir=$PWD/bazel-deps //path/to/target:name
     sources:
       - type: file
         url: [url to a dependency file, can be extracted from the content of id-* files from the repository cache directory]
         sha256: [sha256 of the file, can be extracted from the directory name under repository cache directory]
         dest: bazel-deps
       ...
     ```

     One may automate the process with some shell scripting.

Alternatively, you can add the following to your module if bazel wants to download some dependencies of its own:

```yaml
build-options:
  build-args:
    - --share=network
```

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
    bazel cquery --repository_cache="/path/to/cache" [--config config_name] --experimental_repository_resolved_file=resolved.bzl //path/to/target:name
    ```
   The reason that `bazel cquery` is a preferred way of triggering the dependency fetching over `bazel fetch` is that, `bazel cquery` may accept `--config` flag just like `bazel build`. While `bazel fetch` does not accept such flag. This is very handy if the project is cross-platform and only buildable under certain configuration for Linux.

1. As another approach, Run `bazel build --nobuild` to trigger the downloading process of external dependencies.
    ```bash
    bazel build --nobuild --repository_cache="/path/to/cache" [--config config_name] --experimental_repository_resolved_file=resolved.bzl //path/to/target1:name //path/to/target2:name .. 
    ```

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
       - bazel build --distdir=$PWD/bazel-deps --registry=$PWD/bcr //path/to/target:name
     sources:
       - type: file
         url: [url to a dependency file, can be extracted from the content of id-* files from the repository cache directory]
         sha256: [sha256 of the file, can be extracted from the directory name under repository cache directory]
         dest: bazel-deps
       - type: git
         url: https://github.com/bazelbuild/bazel-central-registry.git
         branch: main
         dest: bcr
       ...
     ```

     One may automate the process with some shell scripting.

     The above method was effective up to version 7.1.2. However, from version 7.2.0 onwards, the id-[sha256 of URL] file is empty and does not contain descriptions of URLs, hashes, etc. It seems a different approach may be necessary.

     The following is the approach as of bazel 7.3.1. It should be valid as long as the format of resolved.bzl doesn't change.
     ```sh
     bazel build --nobuild --repository_cache="/path/to/cache" [--config config_name] --experimental_repository_resolved_file=resolved.bzl //path/to/target1:name //path/to/target2:name .. 
     ```
     Process resolved.bzr with the sed command to make it easier to handle with the jq command.
     ```sh
     sed -e 's/^resolved = //' -e 's/ False/ "False"/g' -e 's/ None/ "None"/g' resolved.bzl > resolved.json
     ```
     Convert to JSON or YAML format that can be recognized by flatpak-builder.
     ```sh
     jq -r '[ .[] | select(.repositories != null) | .repositories[] | select(.attributes.name != null) | select(.rule_class != null) | { url: ( if .attributes.url == "" then ( .attributes.urls[0] // "") else (.attributes.url // (.attributes.urls[0] // "")) end ), sha256: ( .attributes.sha256 // "" ), downloaded_file_path: ( .attributes.downloaded_file_path // null ), } | select(.url != "") | { type: "file", url: .url, dest: "bazel-deps", } + ( if .downloaded_file_path != "" and .downloaded_file_path != null then { "dest-filename": .downloaded_file_path, } else {} end ) + { sha256: .sha256, } ]' resolved.json > projects-deps.json
     yq -y "." projects-deps.json > projects-deps.yaml
     ```
     If the checksum(sha256) is missing, please update it separately.
     Add either `- projects-deps.yaml` or `- projects-deps.json` to the sources.
     ```yaml
     build-commands:
       - bazel build --distdir=$PWD/bazel-deps --registry=$PWD/bcr //path/to/target:name
     sources:
       - type: git
         url: https://github.com/bazelbuild/bazel-central-registry.git
         branch: main
         dest: bcr
       - projects-deps.yaml
       ...
     ```

Alternatively, you can add the following to your module if bazel wants to download some dependencies of its own:

```yaml
build-options:
  build-args:
    - --share=network
```

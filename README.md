# Git Resource

Tracks the commits in a [git](http://git-scm.com/) repository.


## Source Configuration

* `uri`: *Required.* The location of the repository.

* `branch`: *Required.* The branch to track.

* `private_key`: *Optional.* Private key to use when pulling/pushing.
    Example:
    ```
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    ```

* `paths`: *Optional.* If specified (as a list of glob patterns), only changes
  to the specified files will yield new versions.

* `ignore_paths`: *Optional.* The inverse of `paths`; changes to the specified
  files are ignored.

* `skip_ssl_verification`: *Optional.* Skips git ssl verification by exporting
  `GIT_SSL_NO_VERIFY=true`.

* `branches`: *Optional.* Turns on multi-branch mode; takes a regular
  expression as argument -- branches matching the regular expression on origin
  will all be checked for changes.  Uses grep-style regular expression syntax

* `ignore_branches`: *Optional.* Used in conjunction with and applied after
  the `branches` pattern.  See example for common use case.

* `redis`: *Optional.*  Contains the information required to use a specified
  Redis server to store multibranch historic references so that they don't
  clutter up the ref lines.  It consists of the following subkeys:

  * `host`: *Required.* The name or ip of the Redis host.

  * `password`: *Optional.* The password to connect to the Redis server, if
    required.

  * `prefix`: *Optional.* If you have multiple multi-branch pipelines, you
    will want to prefix your keys to separate the histories of each pipeline.

  * `db_number`: *Optional.* The Redis database number, defaults to 0

### Example

Resource configuration for a private repo:

``` yaml
resource_types:
- name: git-multibranch
  type: docker-image
  source:
    repository: cfcommunity/git-multibranch-resource

resources:
- name: source-code
  type: git-multibranch
  source:
    uri: git@github.com:concourse/git-resource.git
    branch: master
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
```

Fetching a repo with only 100 commits of history:

``` yaml
- get: source-code
  params: {depth: 100}
```

Pushing local commits to the repo:

``` yaml
- get: some-other-repo
- put: source-code
  params: {repository: some-other-repo}
```

Detecting changes on all branches except master and deploy

``` yaml
resources:
- name: my-repo
  type: git-multibranch
  source:
    uri: git@github.com:my-org/my-repo.git
    branches: '.*'
    ignore_branches: '(master|deploy)'
```

## Behavior

### `check`: Check for new commits.

The repository is cloned (or pulled if already present), and any commits
made after the given version are returned. If no version is given, the ref
for `HEAD` is returned.

Any commits that contain the string `[ci skip]` will be ignored. This
allows you to commit to your repository without triggering a new version.

### `in`: Clone the repository, at the given ref.

Clones the repository to the destination, and locks it down to a given ref.
Returns the resulting ref as the version.

Submodules are initialized and updated recursively.


#### Parameters

* `depth`: *Optional.* If a positive integer is given, *shallow* clone the
  repository using the `--depth` option.

* `submodules`: *Optional.* If `none`, submodules will not be
  fetched. If specified as a list of paths, only the given paths will be
  fetched. If not specified, or if `all` is explicitly specified, all
  submodules are fetched.

* `short_ref_format`: *Optional.* When populating `.git/short_ref` use this `printf` format. Defaults to `%s`.

### `out`: Push to a repository.

Push the checked-out reference to the source's URI and branch. All tags are
also pushed to the source. If a fast-forward for the branch is not possible
and the `rebase` parameter is not provided, the push will fail.

#### Parameters

* `repository`: *Required.* The path of the repository to push to the source.

* `rebase`: *Optional.* If pushing fails with non-fast-forward, continuously
  attempt rebasing and pushing.

* `tag`: *Optional* If this is set then HEAD will be tagged. The value should be
  a path to a file containing the name of the tag.

* `only_tag`: *Optional* When set to 'true' push only the tags of a repo.

* `tag_prefix`: *Optional.* If specified, the tag read from the file will be
prepended with this string. This is useful for adding `v` in front of
version numbers.

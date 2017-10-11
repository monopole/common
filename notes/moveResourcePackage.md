# Proposal: Move `resource` and `validation` code to k8s.io/common

### Abstract

This doc describes a strategy to move some `kubectl`
code out of the kubernetes core and into a repo
intended for vendoring, in support the general goal of
[extracting kubectl from the core repo][598].  The
strategy allows the moved code to be used immediately,
even before the packages involved are fully extracted
from the core.

The moved code allows a wider range of people to work
on a [DAM] prototype called `kexpand`.  This program,
like `kubectl`, should not live in the core, but will
share code with the core.

### Background

`kubectl` currently lives in the _k8s.io/kubernetes_
repo, also known as the _core_ repo.  Packages in this
repo are not intended for vendoring.  On the other
hand, packages in [_k8s.io/client-go_] and (to some
extent) [_k8s.io/apimachinery_] are intended for
vendoring.

It's generally agreed that `kubectl`, a program that's
supposed to be a pure API client with no dependencies
on core code, should move out of _k8s.io/kubernetes_
and into _k8s.io/kubectl_, hence the name of the latter
repo.  Likewise, no new command line API clients should
appear in core.

Such a move requires additional repos like
_k8s.io/common_ and _k8s.io/utils_, to hold code shared
by `kubectl`, `kubernetes`, and other projects.

[DAM]: https://goo.gl/T66ZcD

Against this backdrop, we want to write a
[DAM]-enabling prototype called [`kexpand`] that wants
to use the shared code.  This should be done in a way
that improves, rather than worsens, relations between
these repositories:

* [_k8s.io/kubernetes_] -- core kubernetes components;
  kubelet, scheduler, etc.
* [_k8s.io/kubectl_] -- home of `kexpand`
  and eventual home of `kubectl`.
* [_k8s.io/common_] -- code shared by `kexpand`,
  `kubectl` and the core.
* [_k8s.io/utils_] -- code shared by everyone (a supplement to [golang]).

[`kexpand`]: https://github.com/kubernetes/kubectl/tree/master/cmd/kexpand

[_k8s.io/kubernetes_]: https://github.com/kubernetes/kubernetes
[_k8s.io/kubectl_]: https://github.com/kubernetes/kubectl
[_k8s.io/common_]: https://github.com/kubernetes/common
[_k8s.io/utils_]: https://github.com/kubernetes/utils
[_k8s.io/apimachinery_]: https://github.com/kubernetes/apimachinery
[_k8s.io/client-go_]: https://github.com/kubernetes/client-go
[golang]: https://golang.org/pkg/
[_k8s.io/kubernetes/pkg/kubectl_]: https://github.com/kubernetes/kubernetes/tree/master/pkg/kubectl

`kexpand` needs to re-use code currently used by
`kubectl` that lives in
[_k8s.io/kubernetes/pkg/kubectl_] - at least the
`resource` and `validation` packages (see
Procedures).

To allow reuse, these packages should move to
_k8s.io/common_.

[50475]: https://github.com/kubernetes/kubernetes/issues/50475
[598]: https://github.com/kubernetes/community/pull/598

Moving code requires

1. Copying code, retaining git history.
1. Arranging for the code to get the
   deps it needs in its new location.
1. Repairing things in the original location
   by vendoring from the new location.
1. Deleting the now unreferenced code
   from the original location.

This is part of a larger problem of factoring
_k8s.io/kubernetes_ into smaller reusable parts.  We
want to start working on `kexpand` before everything is
detangled, but do so in a way that helps improve the
overall situation rather than making it worse.

Rejected schemes include building `kexpand` in the core
repo (introducing yet another eventual extraction
problem), and building `kexpand` by vendoring in _all_
of _k8s.io/kubernetes_, solving problems that creates,
while doing nothing towards moving code out of core
into to repos meant for vendoring.

### Work

The Procedures section below has a
script that locally copies code from
_k8s.io/kubernetes_
<blockquote>
<pre>
{project}/      {repo}/   {path}
   k8s.io/  kubernetes/   pkg/api
   k8s.io/  kubernetes/   pkg/kubectl/resource
   k8s.io/  kubernetes/   pkg/kubectl/validation
</pre>
</blockquote>

to these respective directories in _k8s.io/common_:

<blockquote>
<pre>
   k8s.io/      common/   pkg/api
   k8s.io/      common/   resource
   k8s.io/      common/   validation
</pre>
</blockquote>

This informs and emulates the desired end state of
`resource` and `validation` living in
_k8s.io/common_.

Nobody wants `pkg/api` to live there too, since it
holds unversioned types that are not part of a public
API.  Unfortunately, `resource` and `validation` depend
on it at the moment.  Breaking this dependence is part
of work described below.

Two projects can now proceed independently:

#### Project 1) Permanently extract code from core to _k8s.io/common_

 * Add a warning to _k8s.io/common/README.md_
   explaining that the repo is merely a mirror and
   should not accept changes (as is done for
   [_k8s.io/apimachinery_]).

 * Within _k8s.io/kubernetes_, break `resource` and
   `validation`'s dependence on core packages like
   `pkg/api`.  This is the tricky part.

 * Per work in project 2, `resource` and `validation`
   are periodically copied to _k8s.io/common_
   preserving git history.

 * Modify all code in _k8s.io/kubernetes_ (in
   particular the `kubectl` program) to vendor
   `resource` and `validation` from _k8s.io/common_
   instead, and delete `resource` and `validation` from
   _k8s.io/kubernetes_.  Commit the change.

 * Remove the warning in _k8s.io/common/README.md_; the
   repo now a normal repository containing the
   canonical `resource` and `validation` source code.

#### Project 2) Establish development loop for `kexpand`

 * Come to work in the morning.

 * Optionally refresh `resource` and `validation`.

   * For a time (until project 1 completes) treat
     _k8s.io/common_ as a generated, non-canonical
     repo.  Its `README` should label the repo as a
     generated copy of certain directories from
     _k8s.io/kubernetes_.

   * Use the script below to copy `resource`,
     `validation`, etc. from an up-to-date local clone
     of _k8s.io/kubernetes_ to a local clone of
     _k8s.io/common_, preserving history and adapting
     the code as needed (e.g. package path changes).

   * Diff local _k8s.io/common_ against upstream
     github, and if it has sufficiently changed, push
     it upstream into github.

   * cd into _k8s.io/kubectl_ and run the [`dep`] tool
     to freshly vendor from (github) _k8s.io/common_
     into (local) _k8s.io/kubectl/vendor_.

   * Push _k8s.io/kubectl/vendor_ changes upstream to
     github.

 * Make improvements to `kexpand` in
   _k8s.io/kubectl_.

 * Push `kexpand` changes upstream.

[`dep`]: https://github.com/golang/dep

The result is that `kexpand` will be built to vendor
from _k8s.io/common_ which is the desired _final_
state, but without first requiring the work necessary
to turn _k8s.io/common_ into a "normal" repo

## Procedures

### Investigation - what core code is needed?

Define some query functions:

```
function whatTheHeck {
  pushd ~/gopath1/src/k8s.io/kubernetes >/dev/null
  find ./ -name "*.go" |\
    xargs grep "type SelfSubjectAccessReview "
  popd >/dev/null
}
```

```
function scanImports {
  find $1 -name "*.go" |\
    xargs grep "\"k8s.io/" |\
    grep -v "// import \"" |\
    sed 's|^\(.*\):.*\"\(.*\)\"|\2    \1|' |\
    grep -v k8s.io/api/ |\
    grep -v k8s.io/apimachinery/ |\
    grep -v k8s.io/apiserver/ |\
    grep -v k8s.io/client-go/ |\
    grep -v k8s.io/common/ |\
    grep -v k8s.io/kube-openapi/ |\
    grep -v k8s.io/utils/ |\
    sort |\
    uniq
}

# Show what $1 imports, excluding known good vendors
function whatDoesThisNeedGrep {
  pushd $GOPATH/src/k8s.io/kubernetes >/dev/null
  scanImports $1
  popd >/dev/null
}

# Show all dependencies of $1 using bazel
function whatDoesThisNeedBazel {
  local path=//$1
  local target=$path
  if [[ $target != *":"* ]]; then
    target="$target:go_default_library"
  fi
  pushd $GOPATH/src/k8s.io/kubernetes >/dev/null
  bazel query "buildfiles(deps($target))" |\
    grep -v @bazel_tools |\
    grep -v @io_bazel_rules |\
    grep -v @io_kubernetes_build |\
    grep -v @go1_8_3 |\
    grep -v @local_config |\
    grep -v @local_jdk |\
    grep -v build/visible_to: |\
    grep -v //external: |\
    grep -v $path |\
    grep -v //vendor/ |\
    sed 's/:BUILD//' |\
    grep -v /.bazel |\
    sort | uniq
  popd >/dev/null
}

# Show all dependencies of $1 using go list
function whatDoesThisNeedGo {
  pushd $GOPATH/src/k8s.io/kubernetes >/dev/null
  go list -f '{{ join .Deps "\n"}}' k8s.io/kubernetes/$1 |\
    grep k8s\.io |\
    grep -v /vendor/
  popd >/dev/null
}

function whatDoesThisNeed {
  echo "----- According to Bazel"
  whatDoesThisNeedBazel $1
  echo " "
  echo "----- According to Go"
  whatDoesThisNeedGo $1
  echo " "
  echo "----- According to Grep"
  whatDoesThisNeedGrep $1
  echo " "
}

# Find a string in Go files
function goFind {
  pushd $GOPATH/src/k8s.io/kubernetes >/dev/null
  find ./ -name "*.go" | xargs grep $1
  popd >/dev/null
}

# Show _any_ path from $1 to $2
function whyThisDep {
  local myGuy=$1
  local theConfusingDep=$2
  if [[ $myGuy != *":"* ]]; then
    myGuy="$myGuy:go_default_library"
  fi
  pushd $GOPATH/src/k8s.io/kubernetes >/dev/null
  bazel query "somepath($myGuy, $theConfusingDep:*)"
  popd >/dev/null
}

# Show _all_ paths from $1 to $2
function showAllPaths {
  local myGuy=$1
  local theConfusingDep=$2
  pushd $GOPATH/src/k8s.io/kubernetes >/dev/null
  local q="allpaths($myGuy/...,$theConfusingDep:*)"
  echo $q
  bazel query $q
  bazel query $q --output graph | dot -Tpng > /tmp/dep.png
  echo " "
  echo "Try:"
  echo " display /tmp/dep.png"
  popd >/dev/null
}
```

####  Investigate

What does `resources` need?
```
whatDoesThisNeed pkg/kubectl/resource
```

<blockquote>
<pre>
//hack/boilerplate
//pkg/api
//pkg/kubectl/validation
</pre>
</blockquote>


Why `//hack/boilerplate`?
```
whyThisDep pkg/kubectl/resource hack/boilerplate
```

apimachinery has a hack to copy `/hack` from the main repo into their own repo
to satisfy this dep; we might do the same, or just get rid of this entirely
until kubectl is extracted.

Why does `resource` depend on `validation`?

```
whyThisDep pkg/kubectl/resource pkg/kubectl/validation
```
Because it uses `schema.go`.

What does `validation` need?
```
whatDoesThisNeed pkg/kubectl/validation
```
Nothing.

Why does `resource` depend on `pkg/api`?

```
whyThisDep pkg/kubectl/resource pkg/api
```
The output is just
<blockquote>
<pre>
//pkg/kubectl/resource:go_default_library
//pkg/api:go_default_library
//pkg/api:taint.go
</pre>
</blockquote>

Ignoring tests, only
`pkg/kubectl/resource/result.go` imports `/pkg/api`,
and further it only needs the files
<blockquote>
<pre>
pkg/api/types.go
pkg/api/register.go
</pre>
</blockquote>

Hopefully all but these files can be removed from our
copy of `pkg/api` (and in parallel find a sane way to
break this dependency - Phil is working on this.).

It's interesting to look at all paths from build
targets below `resource` to targets below `pkg/api`,
some of which go through `pkg/apis`.  The tests in
`resource` have many more dependencies than just
`resource`.

```
showAllPaths pkg/kubectl/resource pkg/api
```
Hence it's important to delete the code
that `resource` doesn't directly use.

### The morning refresh script

Define the github user that owns the cloud based fork
of _kubernetes_:

```
# obviously replace your handle here.
GH_USER_NAME=monopole
```

Clone the target repo.

This is where copied from core must land and be
massaged into a working state before being pushed to
github's _kubernetes/common_.  `kexpand` will then
vendor it from github via the `dep` tool.

This disposable working directory is created by a script.
```
TMP_GOPATH=$(mktemp -d)
WORK_TARGET=$TMP_GOPATH/src/k8s.io/common
mkdir -p $WORK_TARGET
git clone https://github.com/$GH_USER_NAME/common $WORK_TARGET
```

Make a branch to work in:
```
cd $WORK_TARGET
BRANCH_NAME=contentMove
git checkout -b $BRANCH_NAME
```

[gbayer]: http://gbayer.com/development/moving-files-from-one-git-repository-to-another-preserving-history

Define a function to copy a specific directory from
a source repo to a target repo.  The technique used
to [retain git history][gbayer] means only one
directory can be moved at a time:

```
function copyDirectory {
  local DIR_SOURCE=$1
  local DIR_TARGET=$2
  local REMOTE_NAME=whatever

  # Place to clone it.
  local WORK_SOURCE=$(mktemp -d)

  git clone \
      https://github.com/$GH_USER_NAME/kubernetes \
      $WORK_SOURCE

  cd $WORK_SOURCE
  git checkout -b $BRANCH_NAME

  # Delete everything in the source repo
  # except the files to move:
  git filter-branch \
    --subdirectory-filter $DIR_SOURCE \
    -- --all

  # Move retained content to the target directory
  # in the target repo.
  mkdir -p $DIR_TARGET

  # The -k avoids the error from '*' picking
  # up the target directory itself.
  git mv -k * $DIR_TARGET

  # Commit the change locally.
  git commit -m "Isolated content of $DIR_SOURCE"

  # The repo now contains only the code to copy.
  # Move into the target and merge it in.
  cd $WORK_TARGET
  git status
  git remote add $REMOTE_NAME $WORK_SOURCE
  git fetch $REMOTE_NAME
  git merge --allow-unrelated-histories \
      -m "Copying $DIR_SOURCE" \
      $REMOTE_NAME/$BRANCH_NAME
  git remote rm $REMOTE_NAME

  # Delete the traumatized `$WORK_SOURCE` directory.
  rm -rf $WORK_SOURCE
}
```

Do the actual copy:

```
# copyDirectory pkg/api              pkg/api
copyDirectory pkg/kubectl/validation validation
copyDirectory pkg/kubectl/resource   resource
copyDirectory hack/boilerplate       hack/boilerplate
```

This leaves `$WORK_TARGET`, which started as a clone of
_k8s.io/common_, with fresh copies of the given packages.

#### Apply Phil's change


```
cd $WORK_TARGET
patch resource/result.go \
  -i ~/gopath1/src/k8s.io/common/notes/patch_result_go
```

#### Delete stuff we don't need

No need to test copied code, so delete all the tests
(so their dependencies don't need to be addressed):

```
cd $WORK_TARGET
find ./ -name "*_test.go" | xargs git rm
git rm -rf pkg/api/testing
git rm -rf pkg/api/testapi
```

Recall from above that only a few files are needed from
`pgk/api`, so delete everything else to avoid having to
transitively copy in or vendor dependencies:

```
cd $WORK_TARGET
git rm -rf pkg/api/*/*/*/
git rm -rf pkg/api/*/*/
git rm -rf pkg/api/*/
```

```
cd $WORK_TARGET/pkg/api
find ./ -name "*.go" |\
  grep -v \./taint.go |\
  grep -v \./types.go |\
  grep -v \./register.go |\
  grep -v \./zz_generated.deepcopy.go |\
  xargs git rm
```

```
cd $WORK_TARGET/pkg/api
git rm OWNERS
```

Overwrite the `BUILD` file:

```
cd $WORK_TARGET/pkg/api
git edit BUILD
cat <<EOF >BUILD
load("@io_bazel_rules_go//go:def.bzl", "go_library")

go_library(
    name = "go_default_library",
    srcs = [
        "resource.go",
        "taint.go",
        "types.go",
    ],
    importpath = "k8s.io/kubernetes/pkg/api",
    visibility = ["//visibility:public"],
    deps = [
        "//vendor/k8s.io/apimachinery/pkg/api/resource:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/apimachinery/announced:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/apimachinery/registered:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/apis/meta/internalversion:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/apis/meta/v1:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/conversion:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/fields:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/labels:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/runtime:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/runtime/schema:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/runtime/serializer:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/types:go_default_library",
        "//vendor/k8s.io/apimachinery/pkg/util/intstr:go_default_library",
    ],
)

filegroup(
    name = "package-srcs",
    srcs = glob(["**"]),
    tags = ["automanaged"],
    visibility = ["//visibility:private"],
)

filegroup(
    name = "all-srcs",
    srcs = [
        ":package-srcs",
    ],
    tags = ["automanaged"],
    visibility = ["//visibility:public"],
)
EOF
```

### Adjust imports

Edit imports so that the moved code is
imported from _k8s.io/common_ instead of _k8s.io/kubernetes_:

```
function adjustImport {
  echo adjusting $1
  local file=$1
  local old=$2
  local new=$3
  local c="s|\\\"k8s.io/kubernetes/$old/|\\\"k8s.io/common/$new/|"
  sed -i $c $file
  local c="s|\\\"k8s.io/kubernetes/$old\\\"|\\\"k8s.io/common/$new\\\"|"
  sed -i $c $file
}
function adjustAllImports {
  for i in $(find . -name '*.go' );
  do
    adjustImport $i $1 $2
  done
}
```

```
cd $WORK_TARGET
# adjustAllImports pkg/api                pkg/api
adjustAllImports pkg/kubectl/validation validation
adjustAllImports pkg/kubectl/resource   resource
```

Run an import scan:

```
cd $WORK_TARGET
scanImports .
```

There should be no output, since everything should now
be coming from packages intended for vendoring.

At this point one could try a build, e.g.
<blockquote>
<pre>
bazel build validation:go_default_library
</pre>
</blockquote>
or
<blockquote>
<pre>
GOPATH=$TMP_GOPATH go build k8s.io/common/validation
</pre>
</blockquote>

But it won't work because we've not yet vendored in
dependencies.


#### Reset workspace

```
cd $WORK_TARGET
bazel clean --expunge

export WORKSPACE=$WORK_TARGET
rm $WORKSPACE/WORKSPACE
touch $WORKSPACE/WORKSPACE
cp $GOPATH/src/k8s.io/kubernetes/WORKSPACE $WORKSPACE
# Just in case
bazel clean --expunge

rm -rf $WORK_TARGET/__my_bazel
alias mybazel="bazel --output_user_root=$WORK_TARGET/__my_bazel "
```

Reset `dep` vendoring:
```
cd $WORK_TARGET
rm -rf vendor/ Gopkg.lock  Gopkg.toml
```

#### Building the copied code

<!--

Deprecated?

Try this:
```
cd $WORK_TARGET
mybazel build hack/boilerplate:boilerplate.go.txt
```

If it fails, try this, and try again:
```
# see https://github.com/kubernetes/kubernetes/issues/52677
function fixBug52677 {
  local hash=$(ls -C1 /tmp/bazel | grep -v install)
  local fixMe=/tmp/bazel/$hash/external/io_bazel_rules_go/go/private/binary.bzl
  grep "\[lib\]" $fixMe
  sed -i 's/\[lib\]/\[libs\]/' $fixMe
  grep "\[libs\]" $fixMe
}
fixBug52677
```

and continue...
```
cd $WORK_TARGET
mybazel build validation:go_default_library
```

If that  doesn't work - try raw go:
```
GOPATH=$TMP_GOPATH go build k8s.io/common/validation
```
Depending on where you are in the timeline, you may get complaints about
missing packages that need to be vendored.

-->

### Set up vendoring

```
GOPATH=$TMP_GOPATH
cd $WORK_TARGET
dep init
dep status
```

### Build

This works
```
GOPATH=$TMP_GOPATH go build k8s.io/common/validation
```
but this fails:
```
GOPATH=$TMP_GOPATH go build k8s.io/common/resource
```

__UNDER CONSTRUCTION__
__UNDER CONSTRUCTION__



### Checkin

__UNDER CONSTRUCTION__

If these local repos now differ from _k8s.io/common_,
they can be pushed up to origin:

```
cd $WORK_TARGET
# assure all tests pass
git diff
git push -f origin $BRANCH_NAME
```

where a PR can be made against _k8s.io/common_.

Hopefully any code changes needed can be done
mechanically, e.g. package name changes.

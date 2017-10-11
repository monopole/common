# common

Common Kubernetes-specific libraries that are not part of
API machinery and not so generic they could be used
outside Kubernetes.

__WARNING__:

The code in this repo under the following directories
<blockquote>
<pre>
   k8s.io/      common/   pkg/api
   k8s.io/      common/   resource
   k8s.io/      common/   validation
</pre>
</blockquote>

is periodically manually copied from

<blockquote>
<pre>
   k8s.io/  kubernetes/   pkg/api
   k8s.io/  kubernetes/   pkg/kubectl/resource
   k8s.io/  kubernetes/   pkg/kubectl/validation
</pre>
</blockquote>

and should not be modified here.

Instead modify it in its canonical location under
[k8s.io/kubernetes/pkg/kubectl].

Goal is to permanently move `resource` and `validation`
(but not `pkg/api`) in q4 2017.

See discussion in this [proposal].

[k8s.io/kubernetes/pkg/kubectl]: https://github.com/kubernetes/kubernetes/tree/master/pkg/kubectl
[proposal]: notes/moveResourcePackage.md

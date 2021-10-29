# End to end verifiability of software, a call to action

The overall problem and vision for solving can be seen in @LukeHinds's talk at
SupplyCHainSecurityCon 2021 (a link to which will be added once the recording is
available).

It essentially boils down to the fact that while much of the world has come to
rely on software in the early 21st century, and this software is overwhelmingly
free and open source (or built to some extent on free and open source software)
the vast majority of that software is built in bespoke ways from relatively
unkown sources. By this, we mean the building blocks from which the software is
built, which we term dependencies, have no _reliable_ provenance information
attached, and the build systems don't necessarily follow standardised (and safe)
approaches nor are they themselves built from parts of reliable provenance.

This is akin to an entire automobile industry building cars out of various
parts, none of which can be reliable traced back to a manufacturer, and using
unknown processes that have not been vetted for. As a process, it is acceptable 
for building backyard soapbox racers, but a disaster in the making when building ambulances.

This document aims to describe the design of a system that follows the vision
outlined above.

# Characteristics

## SBOM based

In a nutshell, the system uses SBOMs (Software Bill of Materials)
generated at every level of the software supply chain to verify and attest
provenance, as a proxy for (or really, component of) trust. This makes it
possible to make policy decisions, such as admitting into production a piece of
software based on reliable information: identity, provenance, and by extension
security vulnerabilities.


## Cloud native

The system is based on a modern software stack, typically a Kubernetes cluster.

It makes use of automation and declarative solutions, so as to emphasises
reproducibility and limit human error, and common cryptograhic building blocks
such as secure hash functions and signatures.

In practice, the technologies used are the following:
- [Kubernetes](https://kubernetes.io/), or a distribution thereof such as 
[OpenShift](https://openshift.com/), as the underlying platform
- [Tekton Pipelines](https://github.com/tektoncd/pipeline/), a Kubernetes-native
 CI/CD system
- [Tekton Chains](https://github.com/tektoncd/chains/), an extention to Tekton 
Pipelines that makes it possible to cryptographically signthe result Tekton 
actions
- a container image registry, such as [Quay](https://quay.io/) 
- [sigstore](https://sigstore.dev) tools (such as
  [cosign](https://github.com/sigstore/cosign)) to sign SBOMs
- [Keylime](https://keylime.dev), to remotely attest servers


## Reproducible builds

One of the goals (and means) of this design is reproducible builds, that is for
two different actors to be able to build the same software (bit for bit exact
copies) from the same source code. This property is desirable as it makes
possible wide-scale comparison and emergence of trust in given software
artefacts: if multiple actors can take the same source and, following a given
process, reach the exact same result, the likelihood of all of these actors
building a compromised version of the software is dramatically reduced.
On the other hand, if each actor can build the software but produce a unique
version, comparison becomes near impossible and the value of the group effort
disappears.

# Design proposal

A developer authors a piece of software. They commit it to a Git repository and
generate a SBOM which contains a digest of the piece of software (ie: a hash of
it) and which they sign.

A Tekton Pipeline watches the Git repository, sees the commit and verifies the
SBOM. Upon validation, it runs the defined build actions and generates (and
validates) a new SBOM documenting the build results of its run, e.g.: an
OCI-compliant container image for instace.

The Tekton Pipeline pushes the newly built container image and associated SBOM
to an OCI-compatible image registry.

A Kubernetes admission controller watches the registry for new versions of
images. Upon detecting the new container image, it replays the chain of
provenance by checking the multiple associated SBOMs (ie: the Tekton-generated
one as well as the developer-generated one). If the image and SBOM are
validated, it allows it into the production cluster: the admission controller is
the arbiter of deployment.

Further aspects:
- the Tekton Pipeline definition is itself signed and verified by the admission
  controller of the cluster on which it runs. This definition has its own SBOM.
- the production nodes are remotely attested using Keylime, based on their
  hardware TPM
- anyone going through the same steps should build bit-for-bit identical
  container images (ie: reproducible buils), allowing for widescale comparison
and verification of builds

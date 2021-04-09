---
title: Packaging
---

Available in [v0.17.0-alpha.1+](https://github.com/vmware-tanzu/carvel-kapp-controller/tree/dev-packaging/alpha-releases)

**Disclaimer:** These APIs are still very much in an alpha stage, so changes
will almost certainly be made and no backwards compatibility is guaranteed
between alpha versions.

The new alpha release of kapp-controller adds new APIs to
bring common package management workflows to a Kubernetes cluster.
This is done using three new CRs: PackageRepository, Package, and
InstalledPackage, which are described further in their respective sections.
As this is still an alpha feature, we would love any and all feedback regarding these
APIs or any documentation relating to them! (Ping us on Slack)

## Install

See the documentation on [installing the alpha release of kapp-controller](install-alpha.md).

## Terminology

To ensure understanding of the newly introduced CRDs and their uses,
establishing a shared vocabulary of some terms commonly used
in package management may be necessary.

### Package

A single package is a combination of configuration metadata and OCI images that ultimately inform the package manager what software it holds and how to install it into a Kubernetes cluster. For example, an nginx-ingress package would instruct the package manager where to download the nginx container image, how to configure associated Deployment, and install it into a cluster.

### Package Repositories

A package repository is a collection of packages that are grouped together.
By informing the package manager of the location of a package repository, the
user gives the package manager the ability to install any of the packages the
repository contains.

---
## CRDs

### Package CR

In kapp-controller, a package is represented by the Package CR. The Package CR
contains versioned metadata which tells kapp-controller where to find the
kubernetes manifests which make up the package's underlying workload and how
to template and install those manifests.

**Note:** for this alpha release, dependency management is not handled by kapp-controller

```yaml
apiVersion: package.carvel.dev/v1alpha1
kind: Package
metadata:
  # Resource name. Should not be referenced by InstalledPackage.
  # Should only be populated to comply with Kubernetes resource schema.
  # spec.publicName/spec.version fields are primary identifiers
  # used in references from InstalledPackage
  name: fluent-bit.vmware.com.1.5.3
  # Package is a cluster scoped resource, so no namespace
spec:
  # Name of the package; Referenced by InstalledPackage (required)
  publicName: fluent-bit.vmware.com
  # Package version; Referenced by InstalledPackage;
  # Must be valid semver (required)
  version: 1.5.3
  # App template used to create the underlying App CR.
  # See 'App CR Spec' docs for more info
  template:
    spec:
      fetch:
      - imgpkgBundle:
          image: registry.tkg.vmware.run/tkg-fluent-bit@sha256:...
      template:
      - ytt:
          paths:
          - config/
      - kbld:
          paths:
          # - must be quoted when included with paths
          - "-"
          - .imgpkg/images.yml
      deploy:
      - kapp: {}
```

### PackageRepository CR

This CR is used to point kapp-controller to a package repository (which contains Package CRs). Once a PackageRepository has been added to the cluster, kapp-controller will automatically make all packages within the store available for installation on the cluster.

```yaml
apiVersion: install.package.carvel.dev/v1alpha1
kind: PackageRepository
metadata:
  # Any user-chosen name that describes package repository
  name: basic.vmware.com
  # PackageRepository is a cluster scoped resource, so no namespace
spec:
  # Must have only one directive.
  fetch:
    # pulls imgpkg bundle from Docker/OCI registry (v0.17.0+)
    imgpkgBundle:
      # Docker image url; unqualified, tagged, or
      # digest references supported (required)
      image: host.com/username/image:v0.1.0
    # pulls image containing packages from Docker/OCI registry
    image:
      # Image url; unqualified, tagged, or
      # digest references supported (required)
      url: host.com/username/image:v0.1.0
      # grab only portion of image (optional)
      subPath: inside-dir/dir2
    # uses http library to fetch file containing packages
    http:
      # http and https url are supported;
      # plain file, tgz and tar types are supported (required)
      url: https://host.com/archive.tgz
      # checksum to verify after download (optional)
      sha256: 0a12cdef83...
      # grab only portion of download (optional)
      subPath: inside-dir/dir2
    # uses git to clone repository containing package list
    git:
      # http or ssh urls are supported (required)
      url: https://github.com/k14s/k8s-simple-app-example
      # branch, tag, commit; origin is the name of the remote (required)
      ref: origin/develop
      # grab only portion of repository (optional)
      subPath: config-step-2-template
      # skip lfs download (optional)
      lfsSkipSmudge: true
```

Example usage:

```yaml
apiVersion: install.package.carvel.dev/v1alpha1
kind: PackageRepository
metadata:
  name: my-pkg-repo.corp.com
spec:
  fetch:
    imgpkgBundle:
      image: registry.corp.com/packages/my-pkg-repo:1.0.0
```

### InstalledPackage CR

This CR is used to install a particular package which ultimately results in installation of package resources onto a cluster. It must reference one of the Package CRs.

```yaml
apiVersion: install.package.carvel.dev/v1alpha1
kind: InstalledPackage
metadata:
  name: fluent-bit
  namespace: my-ns
spec:
  # specifies service account that will be used to install underlying package contents
  serviceAccountName: fluent-bit-sa
  packageRef:
    # Public name of the package to install. (required)
    publicName: fluent-bit
    # Specifies a specific version of a package to install (optional)
    # Either version or versionSelection is required.
    version: 1.5.3
    # Selects version of a package based on constraints provided (optional)
    # Either version or versionSelection is required.
    versionSelection:
      # Constraint to limit acceptable versions of a package;
      # Latest version satisying the contraint is chosen;
      # Newly available, acceptable later versions are picked up and installed automatically. (optional)
      constraint: ">v1.5.3"
      # Include prereleases when selecting version. (optional)
	    prereleases: {}
  # Values to be included in package's templating step
  # (currently only included in the first templating step) (optional)
  values:
  - secretRef:
      name: fluent-bit-values
# Populated by the controller
status:
  packageRef:
    # Kubernetes resource name of the package chosen against the constraints
    name: fluent-bit.tkg.vmware.com.v1.5.3
  # Derived from the underlying App's Status
  conditions:
  - type: ValuesSchemaCheckFailed
  - type: ReconcileSucceeded
  - type: ReconcileFailed
  - type: Reconciling
```

**Note:** In this alpha release, values will only be included in the first
templating step of the package, though we intend to improve this experience in
later alpha releases.

---
## Artifact formats

### Package bundle format

Package bundle is an [imgpkg bundle](/imgpkg/docs/latest/resources/#bundle) that holds package contents such as Kubernetes YAML configuration, ytt templates, Helm templates, etc.

Filesystem structure used for package bundle creation:

```bash
my-pkg/
└── .imgpkg/
    └── images.yml
└── config/
    └── deployment.yml
    └── service.yml
    └── ingress.yml
```

- `.imgpkg/` directory (required) is a standard directory for any imgpkg bundle
  - `images.yml` file (required) contains container image refs used by configuration (typically generated with `kbld`)
- `config/` directory (optional) should contain arbitrary package contents such as Kubernetes YAML configuration, ytt templates, Helm templates, etc.
  - Recommendations:
    - Group Kubernetes configuration into a single directory (`config/` is our recommendation for the name) so that it could be easily referenced in Package CR (e.g. using `ytt` template step against single directory)

See [Creating a package](package-authoring.md#creating-a-package) for example creation steps.

### Package Repository bundle format

Package repository bundle is an [imgpkg bundle](/imgpkg/docs/latest/resources/#bundle) that holds Package CRs.

Filesystem structure used for package repository bundle creation:

```bash
my-pkg-repo/
└── .imgpkg/
    └── images.yml
└── packages/
    └── simple-app.corp.com.1.0.0.yml
    └── simple-app.corp.com.1.2.0.yml
```

- `.imgpkg/` directory (required) is a standard directory for any imgpkg bundle
  - `images.yml` file (required) contains package bundle refs used by Package CRs (typically generated with `kbld`)
- `packages/` directory (required) should contain zero or more YAML files describing available packages
  - Each file may contain one or more Package CR (using standard YAML document separator)
  - Files may be grouped in directories or kept flat
  - File names do not have any special meaning
  - Recommendations:
    - Keep each Package CR in its own file
    - Name each file same as contained Package CR's name (e.g. `simple-app.corp.com.1.0.0.yml`)

See [Creating a Package Repository](package-authoring.md#creating-a-package-repository) for example creation steps.

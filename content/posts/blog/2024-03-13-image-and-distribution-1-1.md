---
title: OCI Image and Distribution Specs v1.1 Releases
author:
  name: Open Container Initiative
  tag: oci
tags: blog
date: 2024-03-13
---

The OCI Image Specification and Distribution Specification each had a 1.1.0 release on February 15, 2024.
These are the first minor releases since the 1.0.0 releases in July 2017 and May 2021 respectively, and include a number of notable changes.

# Artifacts

The biggest highlight of these releases is support for associating metadata artifacts with existing OCI images.
New `subject` and `artifactType` fields have been added to the image specification and retrieval of these associated artifacts can be done via a new referrers API in the distribution specification.
However, this is far from the only thing that was changed.
And many of these changes may affect existing OCI conformant projects.

## Guidance for Artifacts

OCI has been used to package objects other than container images for some time, including projects like Helm, OPA, and Wasm.
The guidance for how projects should implement artifacts has now been [codified into the image specification][image-artifact-guidance].

The recommendation to use a `config.mediaType` value different from the recognized container image values remains.
That guidance is now extended to include a new `artifactType` field that enables use cases without a dedicated `config.mediaType`.
In the scenarios were an artifact does not have content for the config or any layers, an [empty JSON descriptor is recommended][image-empty-descriptor] as a placeholder to satisfy the requirement that these fields exist, while serving as an indication to consuming tools that there is no useful content in that descriptor.

## Describing Associations

A `subject` field has been added to the image specification manifests to define an association to another manifest.
This is used by the [Referrers API](#referrers-api) described below.
It allows tools to associate signatures, attestations, and other metadata with images with a common specification.

Putting together the advice for packaging artifacts with the `subject` field to associate the artifact with an image, results in a manifest that may look like:

```jsonc
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/example",
  "config": {
    "mediaType": "application/vnd.oci.empty.v1+json",
    "digest": "sha256:44136fa355b3678a1146a...",
    "size": 2
  },
  "layers": [
    {
      "mediaType": "application/example",
      "digest": "sha256:e3b0c44298fc1c149af...",
      "size": 0
    }
  ],
  "subject": {
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "digest": "sha256:21edd7d11800e94bae9f4...",
    "size": 1769
  },
  "annotations": {
    "org.opencontainers.image.created": "2024-03-02T15:16:17Z",
    "org.opencontainers.image.description": "Example artifact for image sha256:21ed..."
  }
}
```

## Referrers API

In the distribution-spec, the counterpart to the `subject` field is the the new [referrers API][distribution-referrers]: `GET /v2/<name>/referrers/<digest>`.
This API returns an OCI Index listing the manifests where the `subject` digest matches the requested `<digest>` value.
Each of the descriptors in the response also include the `artifactType` and `annotation` values from the referenced manifest.
Including those values allows tooling to select their desired artifact from the OCI Index the same way runtimes use the platform to select an image to run from a multi-platform Index.

The referrers response to `GET /v2/<name>/referrers/sha256:21edd7d11800e94bae9f4...` may look like:

```jsonc
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:9e6569b15e5ed981334003eb8...",
      "size": 724,
      "annotations": {
        "org.opencontainers.image.created": "2024-03-02T15:16:17Z",
        "org.opencontainers.image.description": "Example artifact for image sha256:21ed..."
      },
      "artifactType": "application/example"
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:e40ce69ad428e47eb72ea76d9...",
      "size": 701,
      "annotations": {
        "org.opencontainers.image.created": "2024-03-03T15:15:15Z",
        "org.opencontainers.image.description": "Another example artifact for same image"
      },
      "artifactType": "application/example2"
    }
  ]
}
```

## Support For Existing Registries

The design of the changes to the image and distribution specifications assume that it will take time for registries to upgrade.
To facilitate this transition period, changes to the image spec added new fields which should be ignored by registries without generating errors or modifying the content of the manifest.
Clients pushing a manifest with a `subject` field and querying for referrers have an additional responsibility to manage and use a [fallback tagged Index when the registry does not support the referrers API][distribution-referrers-fallback].
When a registry supports the referrers API, it will return the header `OCI-Subject` set to the digest in the `subject` field in response to a client pushing a manifest with the `subject` field.
Similarly, when querying for referrers, a registry that supports the API will always return a `200` response, even if the response is an empty Index, to distinguish between registries that have not implemented the API and may respond with a `404` or similar error.

The fallback tag is the digest of the requested subject, with the `:` replaced by a `-`.
Clients would fetch that tag similar to pulling any other tagged image.
For example if `GET /v2/<name>/referrers/sha256:21edd7d11800e94bae9f4...` returned a 404, clients should fallback to requesting `GET /v2/<name>/manifests/sha256-21edd7d11800e94bae9f4...`.
The content of the fallback tag on older registries is maintained by clients to be identical to the referrers response on upgraded registries.

The client requirement to manage the fallback tag is subject to race conditions since the existing content of the fallback tag must first be queried to add the new entry.
The fallback tags also clutter the tag listing with entries that are not the images end users are searching for.
Because of this, having registries upgrade to support the referrers API is preferred.

# Data Field

A `data` field was added to the definition of a [descriptor][image-descriptor] in the image specification.
This optional field contains the base64 encoded content of the blob.
When a client supports the field and encounters it, the additional requests to get the blob may be skipped.
Because many clients do not support this field, and base64 encoding adds overhead, it is recommended to only use this feature for small blobs.

# Manifest Maximum Size

When [pushing manifests][distribution-manifest-push], many clients and registries have size limits to prevent various resource exhaustion attacks.
The distribution specification now clarifies that a registry seeing its limit exceeded should return a 413 (payload too large) response to clients.
And the specification also recommends that registries should support up to a 4MiB image manifest (containing JSON descriptors, not the image layer content).
For portability, clients should avoid exceeding this limit by limiting the usage of the `data` field and annotations.

# Deprecation of Non-Distributable Layers

OCI has deprecated the creation of [non-distributable layers][image-non-distributable-layers].
These layers were used to indicate content that should not be uploaded to registries but rather downloaded from an original distributor.
Because the primary use case for these layers no longer required the feature, and the use case breaks many scenarios, including air-gapped network support, OCI recommends against their usage going forward.

# Support for zstd Compression

Media types have been added for [zstd image layers][image-layer-zstd] in the image specification.
Compared to the traditional gzip compression, zstd typically uses less CPU and compresses content to a smaller size.

# Anonymous Blob Mounts

When pushing blobs, a client may first attempt to [mount the blob from another repository][distribution-blob-mount], avoiding the need to upload the blob content.
A blob mount traditionally required a source repository.
This `from` parameter is now optional, and the registry may support the blob mount without a defined source.
This is useful for scenarios where the base image location on a registry is not known to the client, perhaps because the base image was originally pulled from another registry but has already been pushed to the target registry by other image builds.

If the mount fails for any reason, the registry should still default to initiating the blob upload session, making a blob push with or without the attempted blob mount take the same number of API requests.

# Resumable Chunked Blob Uploads

[Chunked uploads][distribution-blob-chunked] allow a blob to be pushed across multiple requests to the registry, which may be useful for pushing large blobs over an unreliable network.
In order to recover an interrupted blob push, it is necessary to know the current progress of the upload on the registry, which the client would not have received.
To update the client state, a `GET` request to one of the preceding `POST` or `PATCH` request locations now returns the current upload location and the received range.

# Registry API Extensions

[Extensions to the registry API][distribution-extensions] may be added by implementations without requiring a change to the OCI distribution specification.
This allows development and testing of new features, in addition to custom features in a registry.
Extensions are denoted by a leading `_` in their API path, which is invalid for a repository name.
The syntax for an extension API is either `/v2/_<extension>/<component>/<module>[?params]` for registry level extensions, or `/v2/<name>/_<extension>/<component>/<module>[?params]` for repository level extensions.
For example, the following extension is used to list installed extensions: `GET /v2/<name>/_oci/ext/discover`.
To prevent conflicts, extensions should register their name with the distribution specification.

# Multiple Matching Platforms

When multiple entries in an [OCI Index][image-index] appear to be the same, runtime tooling should prefer the first match.
This clarification allows future extensibility for tooling that may wish to add additional images for the same platform that would be detected and used by their tooling.

To get involved with efforts to increase the platform extensibility, see the [OCI Working Group on Image Compatibility][wg-image-compatibility].

# Warning Header

Registries may now include a [warning header][distribution-warning] in responses.
This can be used to notify users of issues that are not yet error conditions, including deprecation announcements.

# No Artifact Manifest

Notably absent from this release is a new type of manifest dedicated to artifacts.
This was removed from the release candidates because of a combination of portability concerns and a lack of added value.

A new manifest type is inherently non-portable to older registries since registries parse manifests and reject unknown manifest media types.
This would typically be handled by defining a new type, upgrading the various consumer tools and registries to be ready, and later upgrade the tooling that produces artifacts after time had passed to allow a controlled upgrade.
In practice, OCI saw a notable uptake of the new manifest by content producers in the very early release candidates, long before registries and other consumers had the opportunity to upgrade.

While debating whether and how OCI could phase a rollout of a new manifest type, the alternative of reusing the existing image manifest was revisited.
The image manifest was already being used by artifact producers, including projects like Helm and Sigstore.
By adding the `artifactType` field to the image manifest, and defining the empty JSON descriptor, all of the use cases that previously required a new artifact manifest could be retrofitted into the existing image manifest.

Without any requirements that necessitated the new manifest type, and no easy solution to the portability concerns, the difficult decision was made to remove the new manifest type from the release candidates.
Future work in OCI may revisit a new manifest type by focusing on use cases and capabilities that cannot be provided with the existing manifests.
To participate in the development of the OCI specifications, please [join the open community][oci-community].

[distribution-blob-chunked]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/spec.md#pushing-a-blob-in-chunks
[distribution-blob-mount]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/spec.md#mounting-a-blob-from-another-repository
[distribution-extensions]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/extensions/README.md
[distribution-manifest-push]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/spec.md#pushing-manifests
[distribution-referrers]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/spec.md#listing-referrers
[distribution-referrers-fallback]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/spec.md#unavailable-referrers-api
[distribution-warning]: https://github.com/opencontainers/distribution-spec/blob/v1.1.0/spec.md#warnings
[image-artifact-guidance]: https://github.com/opencontainers/image-spec/blob/v1.1.0/artifacts-guidance.md
[image-descriptor]: https://github.com/opencontainers/image-spec/blob/v1.1.0/descriptor.md
[image-empty-descriptor]: https://github.com/opencontainers/image-spec/blob/v1.1.0/manifest.md#guidance-for-an-empty-descriptor
[image-index]: https://github.com/opencontainers/image-spec/blob/v1.1.0/image-index.md
[image-layer-zstd]: https://github.com/opencontainers/image-spec/blob/v1.1.0/layer.md#zstd-media-types
[image-non-distributable-layers]: https://github.com/opencontainers/image-spec/blob/v1.1.0/layer.md#non-distributable-layers
[oci-community]: https://opencontainers.org/community/overview/
[wg-image-compatibility]: https://github.com/opencontainers/wg-image-compatibility

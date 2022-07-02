# Cloud resources

## IAM

As you can see in the [global documentation](/README.md), some IAM roles and policies are needed to be created on the
selected cloud provider for this stack to work properly. We decided to document them all separately in order to make
it easier to understand them.

The documentation for each policy can be read on their specific documentation:
- AWS
  - [External DNS](./aws/iam/external-dns.md)
  - [CSI Driver](./aws/iam/csi-driver.md)
- GCP
  - [External DNS](./gcp/iam/external-dns.md)
  - [CSI Driver](./gcp/iam/csi-driver.md)
## External DNS
In the following lines, you can see the IAM Role required by External DNS to manage DNS registries on AWS Route53.

> DISCLAIMER:
> Don't change the name of the resources created on AWS. They need to match a pattern, as you can see in the 
> [global documentation](/README.md)

```terraform
# IAM rules for External DNS
# Needed to allow ExternalDNS manage DNS resources on AWS
resource "aws_iam_role" "external_dns" {
  assume_role_policy = data.aws_iam_policy_document.oidc_controller_document_policy.json
  name               = "${var.cluster_name}-external-dns"
}

resource "aws_iam_role_policy" "external_dns" {
  name = "${var.cluster_name}-external-dns"
  role = aws_iam_role.external_dns.id

  policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Action" : [
          "route53:ChangeResourceRecordSets"
        ],
        "Resource" : [
          "arn:aws:route53:::hostedzone/*"
        ]
      },
      {
        "Effect" : "Allow",
        "Action" : [
          "route53:ListHostedZones",
          "route53:ListResourceRecordSets"
        ],
        "Resource" : [
          "*"
        ]
      }
    ]
  })
}
```

The policy used on the previous Terraform code piece has been extracted from the original repository. 
Even when it is not expected to be changed, because the support for AWS is on a stable stage, if needed, go to see the 
[updated policy](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md) 

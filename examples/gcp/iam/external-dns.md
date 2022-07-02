## External DNS
In the following lines, you can see the IAM Role required by External DNS to manage DNS registries on GCP.

> DISCLAIMER:
> Don't change the name of the resources created on GCP. They need to match a pattern, as you can see in the 
> [global documentation](/README.md)

```terraform
# Service account resource.
resource "google_service_account" "external_dns" {
  account_id   = "${var.cluster_name}-external-dns"
  display_name = "external dns"
  project      = var.project
}

# Crossplane needs dns.admin role in order to create DNS entries
resource "google_project_iam_member" "external_dns" {
  project = var.project
  role    = "roles/dns.admin"
  member  = "serviceAccount:${google_service_account.external_dns.email}"
}

# SA needs workloadIdentityUser role in order to be used by external-dns.
# member = serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]. KSA=kubernetes Service account
resource "google_service_account_iam_member" "external_dns" {
  service_account_id = google_service_account.external_dns.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project}.svc.id.goog[external-dns/external-dns]"
}
```

The policy used on the previous Terraform code piece has been extracted from the original repository. 
Even when it is not expected to be changed, because the support for AWS is on a stable stage, if needed, go to see the 
[updated policy](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/gke.md) 

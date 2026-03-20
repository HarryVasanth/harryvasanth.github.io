---
layout: post
title: "DevOps - Testing: Infrastructure-as-Code Verification with Terratest and Go"
date: 2024-06-15 21:22:04 +01
categories: devops automation
tags: terratest terraform devops testing automation golang infrastructure-as-code
---

## The 'Happy Path' Delusion in Terraform

In Infrastructure as Code (IaC), a successful `terraform apply` does not mean your infrastructure is actually working. It only means that the cloud provider's API accepted your request. Your Terraform might say "Created Security Group," but did you get the CIDR range right? Is the port actually open? Can the application actually connect to the database?

A smart DevOps engineer treats infrastructure like application code. This means it needs **Unit and Integration tests**. **Terratest** is a Go library that allows you to write actual tests that spin up real infrastructure, verify it works, and then tear it down.

## Why Go for Testing Infrastructure?

Terratest is written in Go because Go has a powerful, built-in testing framework and first-class support for cloud SDKs (AWS, Azure, GCP). By using a real programming language instead of a DSL, you can perform complex logic:

- "Wait until the HTTP endpoint returns a 200 OK."
- "SSH into the instance and check if the service is running."
- "Try to write to an S3 bucket and verify it's blocked (Negative testing)."

## Implementation: A Simple Terratest for a Web Server

Imagine you have a Terraform module that creates an EC2 instance running Nginx.

```go
// test/webserver_test.go
package test

import (
    "testing"
    "time"
    "github.com/gruntwork-io/terratest/modules/http-helper"
    "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestWebServer(t *testing.T) {
    // 1. Define the Terraform options
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/webserver",
    })

    // 2. Clean up at the end of the test
    defer terraform.Destroy(t, terraformOptions)

    // 3. Deploy the infrastructure
    terraform.InitAndApply(t, terraformOptions)

    // 4. Get the output (the public IP)
    publicIp := terraform.Output(t, terraformOptions, "public_ip")

    // 5. Verify the service is working
    url := "http://" + publicIp + ":80"
    http_helper.HttpGetWithRetry(t, url, nil, 200, "Hello World", 30, 5*time.Second)
}
```

## Running the Test

```bash
cd test
go mod init my-infra-tests
go mod tidy
go test -v -timeout 30m
```

## Suitable Strategy: The 'Module Example' Pattern

Terratest encourages a professional repository structure:

- `/modules`: Your reusable Terraform code.
- `/examples`: Small, working examples of how to use those modules.
- `/test`: Terratest code that runs against the `/examples`.

This provides two benefits:

1.  **Documentation**: Users can see the `/examples` to understand how to use your module.
2.  **Continuous Integration**: Your CI pipeline (GitLab/GitHub) runs the Terratests on every Pull Request. If a change to the module breaks the Nginx connection, the test fails, and the code is never merged.

## Cost Management with 'defer'

The biggest fear with automated infra testing is "Leaked Resources" (leaving a NAT Gateway running for a month). Terratest's use of Go's `defer` keyword ensures that `terraform destroy` is called even if the test panics or fails halfway through. This is much more reliable than manual cleanup scripts.

## Summary

Infrastructure as Code without testing is just "Infrastructure as Luck." By adopting Terratest, you move your infrastructure management to a mature, engineering-first approach. You gain the confidence to refactor complex modules, knowing that your automated tests will catch any regressions before they reach your production environment. It is the definitive standard for high-scale, professional DevOps teams in 2024. It turns the "Apply" button into a "Verify" pipeline.

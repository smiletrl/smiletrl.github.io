+++
author = "Rulin Tang"
title = "Terraform import aws s3 bucket"
date = "2021-03-16"
description = "terraform import aws s3 bucket"
featured = true
tags = [
    "terraform",
    "featured"
]
categories = [
    "terraform",
]
series = ["Terraform"]
aliases = ["terraform-import-aws-s3-bucket"]
thumbnail = "images/building.png"
+++

This post shows two possible methods to import aws s3 buckets into terraform state.
<!--more-->

## Terraform Apply Error

It's common to get terraform s3 bucket error when we start using terraform to work with existing aws account, saying something like:


> `Error:` Error creating S3 bucket: BucketAlreadyOwnedByYou: Your previous request to create the named bucket succeeded and you already own it.

It means this s3 bucket is existing in aws already, and what we can do is to import the S3 bucket back to our terraform state. Then `terraform apply` will not try to create it again.

Before we start run import command, it might be a good idea to run `aws s3 ls` to get a list of existing s3 buckets at aws. Result is like:

{{< highlight text >}}
% aws s3 ls
2021-03-15 12:03:25 s3-bucket-name1
2021-03-15 13:06:25 s3-bucket-name2
2021-03-15 13:06:05 s3-bucket-name3
{{< /highlight >}}

## Terraform Import - method one

According to the [S3 official Doc](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket), S3 bucket can be imported using

{{< highlight text >}}
$ terraform import aws_s3_bucket.mybucket s3-bucket-name
{{< /highlight >}}

This command will work for s3 resource declaration like:

{{< highlight terraform >}}
resource "aws_s3_bucket" "mybucket" {
  bucket = "s3-bucket-name"

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        kms_master_key_id = aws_kms_key.mykey.arn
        sse_algorithm     = "aws:kms"
      }
    }
  }
}
{{< /highlight >}}

There's a [great article](https://medium.com/faun/importing-aws-resources-into-terraform-52139c1630a5) with more details you may check.

## Terraform Import - method two

Method one works fine for one bucket, but in case there're different modules reusing the same S3 bucket resource, then there might be problem to make it work.

For example, we have infrastructure directory structure

{{< highlight text >}}
|-- modules
|   |-- s3
|   |   |--main.tf
|-- prod
|   |-- main.tf
|-- staging
|   |-- main.tf
{{< /highlight >}}

File `/modules/s3/main.tf` is having content:

{{< highlight terraform >}}
resource "aws_s3_bucket" "mybucket" {
  bucket = "${var.name}"
}

variable "name" {
    type = string
}
{{< /highlight >}}

File `/prod/main.tf` and `/staging/main.tf` may have content:

{{< highlight terraform >}}
module "s3-bucket-1" {
    source = "../modules/s3"
    name = "s3-bucket-name1"
}

module "s3-bucket-2" {
    source = "../modules/s3"
    name = "s3-bucket-name2"
}
{{< /highlight >}}

In this case, we will use [module import](https://www.terraform.io/docs/cli/commands/import.html#example-import-into-module) to import the S3 bucket.

We may cd into directory `/prod`, and run command like below:

{{< highlight text >}}
% terraform import module.s3-bucket-1.aws_s3_bucket.mybucket s3-bucket-name1
% terraform import module.s3-bucket-2.aws_s3_bucket.mybucket s3-bucket-name2
{{< /highlight >}}

Now, when we run `terraform plan` again, it will not try to create the two buckets any more.

## Conclusion

It's common to have other types of resources existing in aws already, we may use a similar module import method to get it working with terraform :)



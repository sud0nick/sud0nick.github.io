---
author: nick
title: Iterative Resources in Terraform
date: 2024-04-19 16:30:00 -0500
categories: [IaC, terraform]
tags: [terraform, iac, cloud, aws]
comments: true
---

<img src="{{ '/assets/iterative-resources-terraform/header.png' | relative_url }}" alt="Header"/>

## Introduction
`count` and `for_each` work in similar ways but have a subtle difference between them that could spell disaster for your infrastructure. The main difference between them is how the reference to the resource is stored in the state file. When you use `count`, each reference is stored at an index in an array like this:

```config
aws_instance.this[0]
aws_instance.this[1]
aws_instance.this[2]
```
{: file="terraform.tfstate" }

When using `for_each`, each reference is stored in a map where the key comes from the key => value pair at each iteration.

```config
aws_instance.this[“bastion”]
aws_instance.this[“prod_service_a”]
aws_instance.this[“prod_service_b”]
```
{: file="terraform.tfstate" }

## The Problem
Let’s assume both examples above represent the same resources: an EC2 instance for Service A, one for Service B, and a bastion host for administrative purposes. Both examples will deploy the infrastructure and allow you to update them without any issue. So, why do we even have the option to choose?

A problem arises when we want to change our infrastructure. Consider the bastion host which, for security purposes, we don’t want running all of the time next to our production servers. We want to deploy it just for the time needed to perform some administrative tasks and then destroy it to reduce our [attack surface](https://en.wikipedia.org/wiki/Attack_surface). How does terraform react differently between the two examples above? Let’s look at some code and real world output.

## Countdown to Detonation
First, we'll establish an instance list which we'll iterate over in our resource block. You likely wouldn't use such a list unless you wanted all of your instances to be identical but we'll keep things simple for the purposes of this article.

```config
locals {
  instances = [
    "bastion",
    "prod_service_a",
    "prod_service_b"
  ]
}
```
{: file="locals.tf" }

Next, we establish an `aws_instance` resource and use the `count` parameter to tell terraform how many we want. A `Name` tag is set using `count.index` to assign the string from the current iteration of the `local.instances` list to the instance.

```config
resource "aws_instance" "this" {
  count         = length(local.instance_names)
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnet.private.id
  tags = {
    Name = local.instance_names[count.index]
  }
}
```
{: file="ec2.tf" }

We run `terraform apply` and it creates three EC2 instances as expected. Now let's remove our bastion instance by commenting it out in `local.instances` and run `terraform plan`.

```config
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  ~ update in-place
  - destroy

Terraform will perform the following actions:

  # aws_instance.this[0] will be updated in-place
  ~ resource "aws_instance" "this" {
        id                                   = "i-0578762313b033b3e"
      ~ tags                                 = {
          ~ "Name" = "bastion" -> "prod_service_a"
        }
      ~ tags_all                             = {
          ~ "Name" = "bastion" -> "prod_service_a"
        }
        # (28 unchanged attributes hidden)

        # (8 unchanged blocks hidden)
    }

  # aws_instance.this[1] will be updated in-place
  ~ resource "aws_instance" "this" {
        id                                   = "i-0a8032179039f04f5"
      ~ tags                                 = {
          ~ "Name" = "prod_service_a" -> "prod_service_b"
        }
      ~ tags_all                             = {
          ~ "Name" = "prod_service_a" -> "prod_service_b"
        }
        # (28 unchanged attributes hidden)

        # (8 unchanged blocks hidden)
    }

  # aws_instance.this[2] will be destroyed
  # (because index [2] is out of range for count)
  - resource "aws_instance" "this" {
      - ami                                  = "ami-0e001c9271cf7f3b9" -> null
      - tags                                 = {
          - "Name" = "prod_service_b"
        } -> null
      - tags_all                             = {
          - "Name" = "prod_service_b"
        } -> null
    }

Plan: 0 to add, 2 to change, 1 to destroy.
```
{: file="terraform.tfplan" }

Can you see what happened? We expect one instance to be destroyed but why are the other two being changed? If you look closely at the plan output you'll see that the two prod instances shifted left in the array. Where the bastion used to be at `aws_instance.this[0]` we now see `prod_service_a`. Likewise, we see `prod_service_b` at `aws_instance.this[1]` where `prod_service_a` used to be. Since they've changed indexes, terraform sees them as needing to be updated.

Also, notice how terraform is just updating the tags. We're not even getting the desired result by removing `bastion` from the list. The bastion host is being renamed to `prod_service_a`, `prod_service_a` is being renamed to `prod_service_b`, and `prod_service_b` is being destroyed! It's going to be a long night for the poor engineer who applies this plan.

## A Surgical Approach
Fortunately, there is a better way using `for_each`. Let's convert our list of instances into a map. This will also give us the ability to define custom attributes for each if we so desire.

```config
locals {
  instances = {
    bastion        = {}
    prod_service_a = {}
    prod_service_b = {}
  }
}
```
{: file="locals.tf" }

Next, we update our resource block replacing `count` with `for_each` and changing the `Name` tag to use `each.key` which pulls the key from each iteration of the map. The code looks cleaner now without the calls to `length()` and `[count.index]`.

```config
resource "aws_instance" "this" {
  for_each      = local.instances
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnet.private.id
  tags = {
    Name = each.key
  }
}
```
{: file="ec2.tf" }

When we run a plan, the output looks slightly different than before. We see the key from each map entry has been used in the resource map. Other than that, this is the same code as before and we get the same result - three EC2 instances.

```config
Terraform will perform the following actions:

  # aws_instance.this["bastion"] will be created
  ...
  # aws_instance.this["prod_service_a"] will be created
  ...
  # aws_instance.this["prod_service_b"] will be created
  ...

Plan: 3 to add, 0 to change, 0 to destroy.
```
{: file="terraform.tfplan" }

Now, let's comment out the bastion host once again and run a new plan. This time our instances don't get shifted around. Instead, we get exactly the desired result - just the bastion host is targeted for destroy.

```config
Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_instance.this["bastion"] will be destroyed
  # (because key ["bastion"] is not in for_each map)
  - resource "aws_instance" "this" {
      - tags                                 = {
          - "Name" = "bastion"
        } -> null
      - tags_all                             = {
          - "Name" = "bastion"
        } -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```
{: file="terraform.tfplan" }

## When Should I Count?
Does this mean we should abandon all use of `count`? Not quite, we just need to know when to use it. One such time is when we want resources to be optional. Let's reuse our bastion host example and this time we'll add a variable to enable/disable the resource.


```config
variable "enable_bastion" {
    description = "Whether to deploy a bastion instance."
    type = bool
    default = false
}
```
{: file="variables.tf" }

```config
resource "aws_instance" "bastion" {
  count         = var.enable_bastion ? 1 : 0
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnet.private.id
  tags = {
    Name = "bastion"
  }
}
```
{: file="ec2.tf" }

Notice how this time we separated the bastion instance into its own resource and used `count` with a [ternary operator](https://en.wikipedia.org/wiki/Ternary_conditional_operator) to determine whether we would deploy 1 or 0 instances. Now when we set the `enable_bastion` var to `true` the instance will be deployed; when it is set to `false` the instance will be destroyed.

We could have also done this with `for_each` and a single resource block by conditionally adding the bastion element to the `local.instances` map.

```config
locals {
  instances = merge({
    prod_service_a = {}
    prod_service_b = {}
    },
    var.enable_bastion ? { bastion = {} } : {}
  )
}
```
{: file="locals.tf" }

Whichever option is easier for your team to read and maintain is usually the better choice.

## Avoiding Pitfalls
There is one major pitfall I must warn you about when using `for_each`. If you're fairly new to terraform, I worry that you'll run into this issue and revert back to using count for "simplicity". The error you might encounter is:

> Error: Invalid for_each argument
...
The "for_each" map includes keys derived from resource attributes that cannot be determined until apply, and so Terraform cannot determine the full set of keys
that will identify the instances of this resource.

You'll encounter this error if you attempt to build a map's keys from values that won't be known until you apply, e.g. an EC2 instance ID. The good news is this only applies to the `keys` of a map and not the `values`. The solution is to restructure your map so the keys are static strings.

Here's some more info from the [docs](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each#using-expressions-in-for_each)

> unlike most arguments, the for_each value must be known before Terraform performs any remote resource actions. This means for_each can't refer to any resource attributes that aren't known until after a configuration is applied (such as a unique ID generated by the remote API when an object is created).

## Conclusion
Iterating over data structures can be a powerful way to deploy your resources and keep your code tidy. However, you must be aware of the different use cases for terraform's iterative meta-arguments and know when to apply each. When used correctly they have the ability to transform your modules from a kludgy mess to an elegant solution without compromising operational stability.

{% if page.comments %}
<div id="disqus_thread"></div>
<script>
    /**
    *  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
    *  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables    */
    var disqus_config = function () {
    this.page.url = "{{ page.url }}";
    this.page.identifier = "{{ page.id }}";
    };
    (function() { // DON'T EDIT BELOW THIS LINE
    var d = document, s = d.createElement('script');
    s.src = 'https://infiniteinterval.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
{% endif %}

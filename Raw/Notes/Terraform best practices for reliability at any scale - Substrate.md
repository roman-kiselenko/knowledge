---
title: Terraform best practices for reliability at any scale - Substrate
source: https://substrate.tools/blog/terraform-best-practices-for-reliability-at-any-scale
clipped: 2023-09-04
published: 
category: tools
tags:
  - terraform
read: false
---

## State file proliferation for fun and profit

When first adopting Terraform there’s a natural attraction, to having a single state file that covers your entire infrastructure. It’s simple. It’s expressive. It’s fast—at first. And it’s a terrible idea.

At scale, many Terraform state files are better than one. But how do you draw the boundaries and decide which resources belong in which state files? What are the best practices for organizing Terraform state files to maximize reliability, minimize the blast-radius of changes, and align with the design of cloud providers?

Our recommendations are based on everything we learned using Terraform at Segment and Slack. Our experience is with AWS, but we think the following strategy holds for any major cloud provider.

Certainly the primary goal for anyone using Terraform is to describe their infrastructure as code so that it may be versioned, branched, reviewed, merged, etc. At scale though, a number of other desired capabilities tend to emerge:

-   Promote changes from development to staging to production. Pre-production environments are important for building confidence in the correctness of our changes.
    
-   Make incremental changes to production infrastructure. A partial outage is preferable to a full outage, and no pre-production environment can completely mitigate the risk of making a change.
    
-   Isolate services to reduce the blast-radius of changes.
    
-   Stamp out copies of bits of infrastructure across regions or availability zones.
    
-   Accommodate the cloud provider’s global services, even while a team is stamping out copies of regional infrastructure.
    
-   Refer to resources in other state files to wire services together into one cohesive whole.
    

It’s simply not possible to achieve all of this with one big Terraform state file or even one state file for each environment. Using the `[-target](https://developer.hashicorp.com/terraform/cli/commands/plan#target-address)` option to `[terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)` is discouraged (the Terraform documentation says, “this is for exceptional use only”). Anyway, it’s likely to lead to confusing infrastructure states if changes are applied incrementally with ad hoc, situational boundaries.

No matter how many state files you have, though, they make refactoring Terraform code very tedious. Even since the introduction of `[moved](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring)` blocks, renaming resources is much more involved than renaming a variable in a programming language, to say nothing of the ceremony required to refactor resources into or out of modules. Thus, it’s important to start with a scalable Terraform architecture.

*If you want a head start on your Terraform setup, with all the scaffolding described in this article done for you, and a great set of tools for managing AWS accounts and roles, along with IdP integration. Take a look at* [*Substrate*](https://substrate.tools/)*, or contact us for a* [*demo*](https://substrate.tools/contact)*.*

## The end result

After following the recommendations detailed here you’ll have a Terraform architecture that is easy enough for a small team and will scale to very large systems. You’ll have skipped all the pain, outages, and difficult refactors that we had to experience to get here.

You will have:

-   At least two environments with separate root modules and state files, such as *development* and *production*.
    
-   Global and regional module separation, even if you only start with one region.
    
-   The ability to test changes in *development*, without impacting *production*.
    

So let’s speed-run engineering our way from a single Terraform state file through to a Terraform architecture that meets all the capability goals set above without having to step slowly through all of these migrations and refactors over the course of months.

## Testing changes in development and staging before reaching production

*💡 How and why to setup separate state files to isolate development and staging changes from production.*

With a single-state file, every attempt to change your *development* or *staging* environment has the potential to change *production*, too.

Using separate Terraform state files is the best way to ensure changes start in *development* and progress to *staging* and then *production*. Have your tooling apply Terraform changes to development first, then staging, and finally production.

To achieve this, you’ll need one directory for each of your environments to serve as the root Terraform module. Each of these directories will have `[provider](https://developer.hashicorp.com/terraform/language/providers/configuration)` and `[terraform](https://developer.hashicorp.com/terraform/language/settings)` blocks somewhat like these:

*💡 For the very best isolation between environments, arrange for each one to be hosted in its own AWS account. Check out “*[*You should have lots of AWS accounts*](https://substrate.tools/blog/you-should-have-lots-of-aws-accounts)*” for more on that design pattern and* [*Substrate*](https://substrate.tools/) *for tooling that makes it all easy.*

Now that each environment has its own root module and state file, you can arrange for changes to progress from one environment to the next. Regardless of whether you apply Terraform changes from engineers’ laptops or from job execution services like AWS CodeBuild or GitHub Actions, you’ll end up running approximately these commands:

You may choose to adorn this process with explicit `[terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)` invocations, human review, blocking for acceptance or approval, or any other step to detect defects. And, discomforting though it may be at first, if pre-production acceptance testing is good, consider adding the `[-auto-approve](https://developer.hashicorp.com/terraform/cli/commands/apply#auto-approve)` option to `[terraform apply](https://developer.hashicorp.com/terraform/cli/commands/apply)` to make the process more hands-off.

## Making changes to one service without affecting another

*💡 How to separate Terraform state at the service level.*

The introduction of separate state files for each environment has bolstered everyone’s confidence in making changes and detecting most defects in *development* and *staging* before they reach *production*. Still, there’s a lingering fear of the unknown, especially if a team’s growing and no one knows what every single person is merging.

Reliable systems are built by controlling changes. With separate Terraform state, a team is never beholden to another in deciding when to apply Terraform changes.

Organizing Terraform code and state by service and environment requires an extra level in a directory tree. Now each service in each environment will have its own root module.

For example:

In each of these leaf directories, you’ll need to reconfigure where their Terraform state is stored, too:

If Terraform were already implemented with a state file for each environment, migrating a service into its own root module with its own state file would require moving the relevant code and then using the `[terraform state mv](https://developer.hashicorp.com/terraform/cli/commands/state/mv)` command (repeatedly) to extract all the relevant resources. This is tedious! (But we’re on a speed run, here, and skipping over this migration.)

Once services have their own root modules and state files, their progression through the environments becomes as independent as the semantics of the services themselves allows:

What’s more: Per-service state files end up being faster because each one’s checking on the state of fewer resources at the beginning of each run.

## Incrementally implementing changes to critical services

*💡 How to safely change services that require the most reliability.*

At most companies, there are typically one or a handful of services that have far higher reliability requirements than everything else. They benefit the most from per-environment state files *and* from their own private state files. Still, they want for ever higher reliability and ever more robust change control. This is the inflection point where doing many small deploys breaks down. It’s not that small deploys are bad; it’s that small deploys from a big team means frequent deploys and frequent deploys demands fast deploys.

Fast deploys don’t give monitors, whether human or machine, any time to detect a problem in production before the problem is affecting 100% of traffic. It’s far better for customers if you can detect a problem and roll back or fix forward based on 1% or 10% of traffic experiencing errors. In that happier case, most folks don’t experience problems at all, and those who do aren’t sure whether your service had a tiny outage or their WiFi had a hiccup.

The easiest way to incrementally change a service is to deploy application servers one at a time. (This might mean updating files and restarting a process on an existing server, or it could mean launching a new server and terminating an old one; the effect is the same.) In case of a defect, stop. But what about changes to all the supporting cloud infrastructure? That’s the realm of Terraform and where even further separation of state files can come in handy.

Consider creating, say, three arbitrary fractions of the whole infrastructure—call them alpha, beta, and gamma for the purposes of this example. Perhaps use Route 53 weighted DNS records or Application Load Balancer target groups to send 1% of our traffic to alpha, 9% to beta, and 90% to gamma.

Likewise, you could use the cloud provider’s regions or availability zones to create differently arbitrary fractions of the whole infrastructure and use the same technique to balance traffic between them. In the case of regions, the geographic distribution of customers will probably dictate the distribution of traffic to each region and possibly inform the order in which changes are applied there. In the case of availability zones, the most natural thing is to have them each take about the same amount of traffic.

To demonstrate both arbitrary and regional separation for `my-cool-service` in production, the directory tree of root modules now looks like this. (Zonal separation would look very much like regional separation and is left as an exercise.)

With the introduction of a second region, be sure to configure the AWS `[provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)` block in those root modules to use the the correct region:

The `[terraform](https://developer.hashicorp.com/terraform/language/settings/backends/s3)` blocks also need a significant upgrade. Qualify the name of the S3 bucket used to store state files because S3 bucket names are global, though S3 buckets are regional. Storing the state files for a given region in that same region gives the best possible chance of surviving outages in other regions. Moving state files around in S3 is perilous so, again, it’s good to be on a speed-run.

This technique can be incredibly powerful when applied to serverless and serverless-adjacent architectures that push a lot of their complexity into cloud provider resources like Lambda functions and API Gateways. It does not apply very well, however, to stateful services, whether you’re running MySQL yourself in EC2 or you’re using the latest and greatest Aurora or DynamoDB products. In most cases, it’s best to manage stateful services once per environment, with the possible extension to once per environment per region. Trying to operate alpha, beta, and gamma copies of a database sounds just crazy enough to cause data loss.

## Stamping out copies of services in many regions

*💡 How to use Terraform with multiple copies of your services in multiple regions.*

Instead of looking at AWS regions as steps in an incremental change process, it may be more natural to look at them as places to stamp out copies of a service over and over again.. Consider, for example, a caching proxy service meant to improve performance for end users and reduce load on an upstream service. It’s an advantage to run this service in every region that’s near any end users. The more, the merrier, in fact, because it’s harder to tolerate the failure of a region serving a huge amount of traffic than it is a region serving only a modest amount.

To implement this, build on the regional directory structure introduced as an option in the previous section—but with an extra layer of indirection to really drive home the idea that you’re stamping out copies. The `root-modules/` directory tree and all the `[provider](https://developer.hashicorp.com/terraform/language/providers/configuration)` and `[terraform](https://developer.hashicorp.com/terraform/language/settings)` blocks will remain unchanged.

Inside each leaf directory in the `root-modules/` directory tree, the ones with the names of AWS regions, have one Terraform resource. Here’s `root-modules/my-cool-service/production/gamma/us-west-2/main.tf`, for example:

This `[module](https://developer.hashicorp.com/terraform/language/modules)` block references a new directory that will host all the Terraform resources to stamp out in every region. (To clarify that big relative pathname, there’s now a `modules/` tree next to the existing `root-modules/` tree.) `modules/my-cool-service/regional` needs a different sort of `[terraform](https://developer.hashicorp.com/terraform/language/settings)` block than what’s seen in root modules:

If you want to change how (each copy of) the service works, make a change in `modules/my-cool-service/regional`. Apply Terraform changes region by region, just like in the regions section to build confidence in the correctness of changes slowly, while minimizing the risk of total outages. Furthermore, add regions easily by creating an additional root module and copying that same three-line `[module](https://developer.hashicorp.com/terraform/language/modules/syntax)` block into it.

## Dealing with AWS’ truly global services

*💡 How to use Terraform with AWS global services that don’t really have a region isolation.*

You may be wondering why, in the last section, there’s a directory called `regional`. It’s because you’re now about to create some directories called `global` to complement it.

If you try to manage the same resource in multiple Terraform state files, Terraform will be very cross and may decide to destroy and recreate some resources, which may be enough to cause an outage. In this Terraform architecture, each root module is configured to use one AWS region.

To complicate matters, a few very important AWS services—IAM, Organizations, Route 53, and CloudFront among them—are global services. These services can be managed from any AWS region, and changes are (eventually) visible in every other region.

Rather than asking everyone who writes Terraform code to remember to only manage their IAM roles from one region, it makes better sense to introduce an explicit place called `global`for these resources to be managed:

Arrange for global root modules to be applied before specific regions in this automation, like so:

Applying changes in global root modules first ensures global resources are available to be queried by `[data](https://developer.hashicorp.com/terraform/language/data-sources)` sources in regional modules that come after. For example, create an IAM role in `modules/my-cool-service/global`:

And when you want to reference this IAM role in `modules/my-cool-service/regional` in order to associate it with regional resources like the EC2 instances in an autoscaling group, use a `[data](https://developer.hashicorp.com/terraform/language/data-sources)` source:

This extra step ensures a smooth experience adding regions.

Now, we’re calling these modules global, but the `[provider](https://developer.hashicorp.com/terraform/language/providers/configuration)` block in global root modules does have to declare a region. Which one should you use? It pains us to recommend this, but AWS us-east-1 is the best choice here. Why? Because some AWS services are, for lack of a better term, pseudo-global. Lambda@Edge, ACM when used with CloudFront, and Service Quotas that concern truly global services are three examples of AWS services that must be managed in us-east-1 and only in us-east-1. Semantically, they’re global, but their API leaks the fact that their control plane is in us-east-1. But since these are pretty common services, the best default is to use us-east-1 to manage all the AWS services with global semantics.

## How much should you do from the start?

*💡 How to use our recommendations when you’re just getting started with Terraform with a relatively small infrastructure.*

The final form of this Terraform architecture offers isolation between environments, services, and regions. There are a couple of techniques available for applying changes incrementally. AWS’ hodgepodge of global and regional services are no bother. It’s relatively easy to add services and add regions.

But not every service in every environment needs such pomp and circumstance. Not now and possibly not ever. How much of this should you do on day one?

We recommend creating enough isolation and structure in Terraform to ensure that future changes are simple additions instead of complicated refactors. In particular, adopt all of these patterns on day one:

-   At least two environments with separate root modules and state files.
    
-   Global and regional module separation, even if you only start with one region.
    
-   Tooling to apply changes in order from less-important to more-important environments, starting at each level with global and then progressing region by region.
    

Terraform is a powerful tool. The worst experiences with it are with massive state files making it very, very slow and with stressful, high-stakes refactoring. Hopefully you can put the lessons we learned the hard way to work early to save yourself from this pain and enjoy the full power of Terraform now and in the future.

——

*About the authors: Richard Crowley and Travis Cole are the authors of* [*Substrate*](https://substrate.tools/)*, a tool that delivers state-of-the-art multi-account AWS architectures, complete with Terraform infrastructure that follows all the recommendations from this article.*
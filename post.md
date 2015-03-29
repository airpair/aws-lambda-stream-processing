[AWS Lambda][lambda] is an AWS (developer preview) product that consumes events
from Kinesis (stream processing), S3 events, DynamoDB changes, and more. You
can use it to make advanced materialized views out of DynamoDB tables, react to
uploaded images, or archive old content. In short, you write a function
(currently only in node.js) and it is presented with JSON containing
information about the event's source and content.

This post is a tour of the powerful ways you can use (or misuse) Lambda to
handle events. First, we'll tour a sample application I built that generates a
static site from markdown files in S3, then we'll examine how that
application's use of Lambda is actually *wrong* in many ways, and why that's
so.

## Caveat Emptors

Before we get started, let's agree on a couple of things. Lambda as a service
name is a bit annoying because it stomps over several other useful contexts for
the word, but we'll suspend those for the moment.

## Enter Lambda

When Amazon introduced [AWS Lambda][lambda] I saw tons of interesting
possibilities. Being able to react to events without needing to constantly run
(and pay for) EC2 instances. I built [hugo-lambda][hugolambda] to take
advantage of Lambda to rebuild my static site whenever I made a change. I
thought Lambda sounded like a great way to build the cheapest hosted [CMS][cms]
around. With S3 [website hosting][s3web] for generated content, [Route53][r53]
for DNS, and Lambda-generated content you can host your site within the AWS
free tier. Even if you don't qualify for the free tier, the total cost for a
site updated once daily would be less than $1 per month.

Of course, if you're like me you don't get around to updating your blog daily,
but that's ok. As with all of AWS, you only pay for what you use.

Every time new content is uploaded, hugo-lambda downloads your site templates,
themes, and content to run `hugo` and uploads the generated site (with the
correct storage ACLs) to the public bucket for your site.

## Something's Not Right

This use case is actually totally wrong for Lambda because it violates two
pretty critical assumptions made by the service. Lambda is built for event
processing, and makes the assumption that every event is independent and that
the event stream can be processed incrementally. For a full static site (in my
case a blog), this isn't true because edits to the content build on one
another and it isn't easy to tell what parts of the site are affected by, say,
a new post.

A new post can cause changes all over the site. The sidebar of every page, the
tag listing page (if the post has a new tag), the archives page, and more.
Without having these changes expressed when a new file is added to S3 it's
impossible to regenerate the site without downloading all the content and
templates first.

## Remedies

The only way to *really* fix this would be to build a dependency tree between
inputs (templates, content, etc) to allow each hugo-lambda run to only download
content that depends on a piece of content. This would further reduce costs and
make each run that much faster.

[cms]: TODO
[hugolambda]: https://github.com/ryansb/hugo-lambda
[lambda]: https://aws.amazon.com/lambda/
[r53]: https://aws.amazon.com/route53/
[s3web]: TODO

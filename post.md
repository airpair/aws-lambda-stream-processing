[AWS Lambda][lambda] is a service (in developer preview) that consumes events
from Kinesis, S3, DynamoDB, and more. You can use it to make advanced
materialized views out of DynamoDB tables, react to uploaded images, or archive
old content. In short, you write a function (currently only in node.js) and it
is presented with JSON containing information about the event's source and
content.

This post is a tour of the powerful ways you can use (or misuse) Lambda to
react to events. First, we'll tour a sample application I built that generates
a static site from markdown files in S3, then we'll examine how that
application actually uses Lambda *incorrectly* in many ways, and how to use it
more effectively.

## Caveat Emptors

Before we get started, let's agree on a couple of things. Lambda as a service
name is a bit annoying because it stomps over several other useful contexts for
the word, but we'll suspend those for the moment.

Lambda also has several limitations at the time of this writing.

* Function runtime is limited to 60 seconds
* Node.js is the only supported language
* Maximum of 500MB storage and 1GB memory
* Debugging involves a lot waiting for CloudWatch logs to show up

## Enter Lambda

When Amazon introduced [AWS Lambda][lambda] I saw tons of interesting
possibilities. Being able to react to events without needing to constantly run
(and pay for) EC2 instances. I built [hugo-lambda][hugolambda] to take
advantage of Lambda to rebuild my static site whenever I made a change.

It's likely the cheapest hosted [CMS][cms] around. With S3 [website
hosting][s3web] for generated content, [Route53][r53] for DNS, and
Lambda-generated content you can host your site within the AWS free tier. Even
if you don't qualify for the free tier, the total cost for a site updated daily
would be less than $1 per month.

Every time new content is uploaded, hugo-lambda downloads your site templates,
themes, and content to run `hugo` and uploads the generated site (with the
correct storage ACLs) to the public bucket for your site.

Of course, if you're like me you don't get around to updating your blog daily,
but that's ok. As with all of AWS, you only pay for what you use. You're
charged only for time hugo-lambda actually spends generating your site instead
of paying to run WordPress, Drupal, or another CMS 24/7.

## Running Unsupported Languages

Over the last several years there has been a huge crop of excellent
static site generators, led by [Jekyll][jekyll]. I prefer [hugo][hugo] for its
flexibility, but every project has its strengths. Conveniently, Lambda
functions are run on 64-bit Amazon Linux and [Go][golang]
compiles its binaries statically.

After some experimentation, I included the `hugo` binary with the Node.js
libraries and was able to `spawn` it as a subprocess.

```
var async = require('async');
var spawn = require('child_process').spawn;

exports.handler = function(event, context) {
    async.waterfall([
    // function to download content skipped for brevity
    function runHugo(next) {
        var child = spawn("./hugo", ["-v", "--source=/tmp", "--destination=/tmp/public"], {});
        child.on('close', function(code) {
            console.log("hugo exited with code: " + code);
            next(null);
        });
    },
    // function to upload finished site skipped for brevity
    ], function(err) {
        if (err) console.error("Failure because of: " + err)
        else console.log("Site generated successfully!");

        context.done();
    }}
}
```

The above code is an abbreviated version of [RunHugo.js][runhugo] from the
hugo-lambda project, but it can (almost) stand on its own.

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

[cms]: http://en.wikipedia.org/wiki/Content_management_system
[golang]: http://golang.org/
[hugo]: http://gohugo.io/
[hugolambda]: https://github.com/ryansb/hugo-lambda
[jekyll]: http://jekyllrb.com/
[lambda]: https://aws.amazon.com/lambda/
[r53]: https://aws.amazon.com/route53/
[runhugo]: https://github.com/ryansb/hugo-lambda/blob/master/generate/lib/RunHugo.js
[s3web]: http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html

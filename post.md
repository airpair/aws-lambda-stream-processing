[AWS Lambda][lambda] is a service (in developer preview) that consumes events
from Kinesis, S3, DynamoDB, and more. You can use it to make advanced
materialized views out of DynamoDB tables, react to uploaded images, or archive
old content. In short, you write a function (currently only in node.js) and it
is presented with JSON containing information about the event's source and
content.

> Another way to run Node.js? Why bother?
>
> -- Everyone

In a way, Lambda is a unique take on the Platform as a Service concept. A
typical PaaS offering might offer to serve your Ruby (or whatever) web app, but
Lambda takes the "serve" part out and replaces it with "reactively run". The
instance your Lambda function runs on isn't running all the time, and you can
have as many functions as you can trigger running at once. You could use it as
a replacement for resque or another background job processor with a managed
solution.

This post is a tour of the powerful ways you can use Lambda to react to events.
First, we'll tour a sample application I built that generates a static site
from markdown files in S3, then we'll examine more effective ways to use
Lambda.

## Caveat Emptors

Before we get started, let's get a few things out of the way. Lambda as a
service name is a bit annoying because it stomps over several other useful
contexts for the word, but we'll suspend those for the moment.

Lambda also has several limitations at the time of this writing.

* Function runtime is limited to 60 seconds
* Node.js is the only supported language
* Maximum of 500MB storage and 1GB memory
* Debugging involves a lot waiting for CloudWatch logs to show up
* Only one Lambda trigger can exist per S3 bucket

## Hugo-Lambda: Demo App

Being able to react to events without needing to constantly run (and pay for)
EC2 instances opens up new ways to use existing tools.
[Hugo-lambda][hugolambda] rebuilds a static site from source whenever a change
is uploaded to S3.

It's likely the cheapest hosted [CMS][cms] around. Using S3 [website
hosting][s3web] for generated content, [Route53][r53] for DNS, and Lambda to
generate the site from source can host your entire site within the AWS free
tier. Even if you don't qualify for the free tier, the total cost for a site
updated daily would be less than $1 per month.

Every time new content is uploaded, hugo-lambda downloads your site templates,
themes, and content to run `hugo` and uploads the generated site (with the
correct storage ACLs) to the public bucket for your site.

Of course, if you're like me you don't get around to updating your blog daily,
but that's ok. As with all of AWS, you only pay for what you use. You're
charged only for time hugo-lambda actually spends generating your site instead
of paying to run WordPress, Drupal, or another CMS 24/7.

### Running Unsupported Languages

Over the last several years there has been a huge crop of excellent
static site generators, led by [Jekyll][jekyll]. I prefer [hugo][hugo], and
since it's written in [Go][golang] it's distributed as a single static
binary.

When included with the Node.js dependencies for the function, `hugo` can be
invoked as a subprocess using `spawn`.

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

### Handling Events

Lambda can take events from a variety of sources, but hugo-lambda only needs to
listen to S3 events. S3 is sort of the odd duck of Lambda notifications because
it doesn't show up in the `list-event-sources` API, instead it's attached to
the bucket and is a part of S3's `get-bucket-notification` API.

```
{
  "CloudFunctionConfiguration": {
    "InvocationRole": "arn:...InvokeRole-SQU198TLCHES",
    "CloudFunction": "arn:...:function:HugoLambdaGenerate",
    "Events": [
      "s3:ObjectCreated:*"
    ],
    "Id": "HugoLambdaGenerate-notification",
    "Event": "s3:ObjectCreated:*"
  },
}
```

Event sources for other DynamoDB and Kinesis follow a similar format, requiring
an invocation role, function ARN, and source ARN.

## Is Lambda a Microservice Platform?

> As an aside: if you haven't, I really recommend reading Martin Fowler's
> definitive piece on [Microservices][microservices].

Now, you may be thinking "small programs with limited state and transparent
scaling? That's just microservices right?" There are certainly overlapping
advantages, let's see what matches up.

### Componentization

Each Lambda function is an independent component, and they can be chained
together by having the output of one trigger the next function (or group of
functions). Because of this, they are easy to experiment with and play well
with other data systems.

### Smart endpoints and dumb pipes

In a lot of definitions of microservices, people take this to mean "uses
RESTful HTTP interfaces between components". Lambda events follow a strict JSON
format. Here's an abbreviated example of an S3 event for a new object.

```
{
  "Records": [
    {
      "s3": {
        "object": {
          "eTag": "50ed8c18234b65e3baf1417eac1bb03f",
          "size": 307,
          "key": "content/posts/test.md"
        },
        "bucket": {
          "arn": "arn:aws:s3:::some-bucket-name
          "ownerIdentity": {
            "principalId": "..."
          },
          "name": "some-bucket-name
        },
        "configurationId": "Kappa-HugoLambdaGenerate-notification",
        "s3SchemaVersion": "1.0"
      },
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2015-04-02T22:50:04.028Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "..."
      }
    }
  ]
}
```

That seems pretty simple, it even includes extra metadata about the object like
it's size and etag (md5sum). The message format is one part of the pipe, the
other part is how messages are received. The event notification system is very
straightforward because it only needs the ARN of the sender (source) and
receiver (Lambda function) to successfully route messages. All the delivery
semantics are hidden completely.

### Decentralized Data Management

This is up to you. Of course, hugo-lambda is a case of highly *centralized* data
management as each function run needs all the site sources to do its job. The
best use cases for Lambda have events that contain all (or most) of the
information needed to process it. An example might be the event generated by an
image upload to be resized in Lambda, or a new document to be indexed.

### Design for Failure

Lambda functions abstract away most failure modes, since instance- and
availability-zone-level failures can be routed around by triggering functions
to run elsewhere.

## Hugo-Lambda Usage Patterns

Hugo-lambda is a great demo application, but not a great use of Lambda. In
fact, it violates two pretty critical assumptions made by the service. Lambda
is on the idea that every event is independent can be processed incrementally.
Unfortunately, for a full static site (in my case a blog), this isn't true.
Edits can be interdependent, and it isn't easy to tell what parts of the site
are affected by a new post or partial template.

A new post can cause changes all over the site. The sidebar of every page, the
tag listing page (if the post has a new tag), the archives page, and more.
Without having these changes expressed when a new file is added to S3 it's
impossible to regenerate the site without downloading all the content and
templates first.

## Improved Usage Patterns

The only way to *really* fix this would be to express the site dependency tree
between inputs (templates, content, etc) to allow each hugo-lambda run to only
download content that depends on a piece of content. This would further reduce
costs and make each run that much faster.

A better use case for Lambda would be to have it roll up events into summary
events, or into other indices. Let's walk through what an example that makes
better use of Lambda.

## DynamoDB Event Roll-Ups

Our better example is a Lambda function that rolls up the stream of incoming
scores into a "recent best" record that has the best scores in the past hour.
This fits Lambda much better because each event (game play-through) is
independent and the high score list doesn't need to be updated by the client,
and is high-traffic so it can't be computed on every read.

### Problem Outline

Writes and reads *both* need to be quick for this case, because you don't want
users to wait after they finish a game to start the next one or wait to see the
high score list when the app opens. At the same time, you can afford to have
some latency between a game completing and the score being posted to the high
score list. 

To solve this with Lambda, we can build a flow like:

1. Game completes and writes information to DynamoDB
1. Lambda function is invoked with the score event
1. Lambda views the new scores and if it beats the old scores, updates the
   list.
1. If changed, the score list is stored in a well-known DynamoDB key in the
   same table to be read by everyone

### Event Format

at the end of each game, this record is stored to the DynamoDB
table. First, let's see what an item looks like.

```
{
    "event_key": "{uid}",
    "length_seconds": 55,
    "start_timestamp": 1428071547,
    "completed": true,
    "score": 924,
    "handle": "EdScissorHands",
    "achievements": [
        {"bonus_score": 200, "id": "{uid}", "shortname": "30 Kill Streak"},
        ...
    ]
}
```

The KeySchema is also important here. That looks like:

```
[
  {
    AttributeName: "event_key",
    KeyType: "HASH"
  },
  {
    AttributeName: "start_timestamp",
    KeyType: "RANGE"
  },
]
```

The event key is composed of the UID of the player and a range key of the event
timestamp. This isn't a great key design, and you can learn more about shard
key design in this [AWS Advent DynamoDB post][awsadvent] or in the [MongoDB docs][mongo-shard], but that's way beyond this article's scope.

### Function Roll-Up

```
// ProcessScores.js
var AWS = require('aws-sdk');
var async = require('async');
exports.handler = function(event, context) {
  var ddb = new AWS.DynamoDB();
  console.log("Event: %j", event);
  async.waterfall([
    function getScores(next) {
      ddb.getItem({
        // ... scores record info ...
      }, function(err, data) {
        // pass the scores to the next step
        next(null, data.Item);
      });
    },
    function readNew(scores, next) {
      var newScores = false;
      for(i = 0; i < event.Records.length; ++i) {
        // for all the new scores, see if any of them beat the old scores
      }
      // if they do, update the "scores" item
      if (newScores) next(null, scores)
      else context.done(); // bail out if there is no change
    },
    function writeNew(scores, next) {
      ddb.putItem({
        Item: scores,
        TableName: "scores-table"
      }, function(err, data){
        next(err);
      })
    }
  ], function(err) {
    if (err) console.error("Failure! " + err);
    context.done();
  });
}
```

The steps we outlined earlier translated easily for this example, and we can
even handle batches of writes (say, 100 completed games at a time) to reduce
the number of Lambda function calls that are made. It's just as simple to
trigger this every time there's a new score, but it's optional since a high
score list doesn't always need to be up-to-date.

## Wrapping Up

Here we've seen two applications of Lambda to different problems, and learned
why some workloads make more sense for this nascent service.


[cms]: http://en.wikipedia.org/wiki/Content_management_system
[golang]: http://golang.org/
[hugo]: http://gohugo.io/
[hugolambda]: https://github.com/ryansb/hugo-lambda
[jekyll]: http://jekyllrb.com/
[lambda]: https://aws.amazon.com/lambda/
[r53]: https://aws.amazon.com/route53/
[runhugo]: https://github.com/ryansb/hugo-lambda/blob/master/generate/lib/RunHugo.js
[s3web]: http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html
[microservices]: http://martinfowler.com/articles/microservices.html
[awsadvent]: http://awsadvent.tumblr.com/post/104615780454/aws-advent-2014-an-introduction-to-dynamodb
[mongo-shard]: http://docs.mongodb.org/manual/tutorial/choose-a-shard-key/

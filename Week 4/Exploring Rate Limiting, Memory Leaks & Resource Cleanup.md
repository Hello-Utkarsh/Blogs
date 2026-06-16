So in our previous few weeks we have been trying to learn to optimize api responses and database queries so that our users are able to get their data instantly and without overwhelming our servers and databases. 

But sometimes there are users/attackers/bots who keep spamming our apis which results in crashing of our servers, no matters how much you optimize your queries or cache your data, there is a limit to which servers and databases can handle requests and after that they goes down/crashes

That when **Rate Limiting** comes into picture, it allows us to control at what rate user requests can be accepted/processed by our servers. For example we can limit the number of requests a user can make to our api to 10req/per hour. If the use exceed the amount, we'd ignore their request and throw an error indicating that he/she has crossed the hourly limit
# Rate Limiting

There are many different algorithms which help apply rate limiting based **traffic predictability**, **memory resources**, **precision requirements**, and **infrastructure architecture**. Let's discuss each one of them one by one

## Token Bucket

As the name suggests, its kind of like a bucket full or token(maybe 10/100/1000 or more) and each request has to firstly get a token, and then only they are allowed to be processed, and the bucket is constantly refilled at a particular rate(maybe 10/sec, 100/sec or more)

If the bucket is empty, the requests is denied with  429(too many requests) header. 
It is a great algorithm if your apis get **Burst Traffic** or with periodic **usage spike** 
### Implementation

```js
import { NextFunction, Request, Response } from "express";
const bucket = new Map([
  ["lastrefill", Date.now()]
  ["token", 10],
]);

const maxTokens = 10;
const refilRate = 1;

export default function TokenBucket(
  req: Request,
  res: Response,
  next: NextFunction,
) {

  const date = Date.now();
  const lastRefill = bucket.get("lastrefill")
  const currentToken = bucket.get("token");
  const tokensGen = Math.floor(
    ((date - bucket.get("lastrefill")) / 1000) * refilRate,
  );
  const bucketToken = Math.min(maxTokens, currentToken + tokensGen);

  if (tokensGen > 0) {
    bucket.set("lastrefill", lastRefill + (tokensGen * 1000 )/refilRate);
  }

  bucket.set("token", bucketToken);

  if (bucketToken >= 1) {
    bucket.set("token", bucketToken - 1);
    console.log(bucket);
    return res.send(200);
  } else {
    console.log(bucket, "overuse");
    return res.send(429);
  }
}
```

## Leaky Bucket

So you must have observed a leaky bucket or pipe right, how water keeps dripping at a constant rate right, this algorithm is exactly like that. It firstly pushes all the incoming requests into a bucket(array) and then process requests from that bucket one by one at a constant rate. If the bucket is full, the requests are dropped.

It smoothens out the incoming traffic and does not allow burst requests unlike Token Bucket

### Implementation

```js
function LeakyBucket(req: Request, res: Response, next: NextFunction) {
  if (bucket.length < 10) {
    bucket.push(() => next());
  } else {
    res.send("Too many requests").status(429);
  }
}

app.get("/", LeakyBucket, (req, res)=>{
  const now = new Date();
  console.log(now.toLocaleTimeString());
  res.send("")
});

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`);
  setInterval(() => {
    if (bucket.length > 0) {
      let a = bucket.shift();
      a();
    } else {
      console.log("empty");
    }
  }, 2000);
});
```

![[Leaky Bucket Test.png]]

As you can see at first it was printing empty because there was no requests in the bucket, and then i sent 6 request, 1/per second using autocannon, but you can se we are getting the log at the interval of 2 seconds, that's because the request is firstly sent in the bucket and then executed at a constant rate of 1 request/ per 2 second. 




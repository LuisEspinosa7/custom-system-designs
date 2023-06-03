# Facebook News Feed Core System

### Explanation
This system was built with the following requirements in mind:
Build a system that:
- Is scalable.
- Fast and efficient
- Has low latency through regional replication (Important)
- Loads a user's news feed
- Posts status updates and updates the poster's friends news feeds in real time.
- Allows 1 billion users
- Allows 500 friends on average for each user
- Uses an out of the box ad server as well as a post ranking subsystem (Integration)

</br>
This system consist of two main parts: </br>

- CreatePost Request </br>
 For the purpose of this design, the CreatePost API call will be very simple and look something like this:
<pre>
CreatePost(
        user_id: string,
        post: data
    )
</pre>

There will be one main relational database to store most of our system's data, including posts and users. This database will have very large tables (be aware that sharding HERE is not useful because of long and time-consuming queries to build the news feed). Since our databases tables are going to be so large, with billions of millions of users and tens of millions of posts every week, fetching news feeds from our main database every time a GetNewsFeed call is made isn't going to be ideal. We can't expect low latencies when building news feeds from scratch because querying our huge tables takes time, and sharding the main database holding the posts wouldn't be particularly helpful since news feeds would likely need to aggregate posts across shards, which would require us to perform cross-shard joins when generating news feeds; we want to avoid this. Instead, we can store news feeds separately from our main database across an array of shards. We can have a separate cluster of machines that can act as a proxy to the relational database and be in charge of aggregating posts, ranking them via the ranking algorithm that we're given, generating news feeds, and sending them to our shards every so often (every 5, 10, 60 minutes, depending on how often we want news feeds to be updated).
If we average each post at 10kB, and a newsfeed comprises of the top 1000 posts that are relevant to a user, that's 10MB per user, or 10 000TB of data total. We assume that it's loaded 10 times per day per user, which averages at 10k QPS for the newsfeed fetching. </br>

Assuming 1 billion news feeds (for 1 billion users) containing 1000 posts of up to 10 KB each, we can estimate that we'll need 10 PB (petabytes) of storage to store all of our users' news feeds. We can use 1000 machines of 10 TB each as our news-feed shards.

<pre>
  ~10 KB per post
  ~1000 posts per news feed
  ~1 billion news feeds
  ~10 KB * 1000 * 1000^3 = 10 PB = 1000 * 10 TB
</pre>

To distribute the newsfeeds roughly evenly, we can shard based on the user id. When a GetNewsFeed request comes in, it gets load balanced to the right news feed machine, which returns it by reading on local disk. If the newsfeed doesn't exist locally, we then go to the source of truth (the main database, but going through the proxy ranking service) to gather the relevant posts. This will lead to increased latency but shouldn't happen frequently. </br>

* Feed Updates (Events for feed update when a post has been created)
 We now need to have a notification mechanism that lets the feed shards know that a new relevant post was just created and that they should incorporate it into the feeds of impacted users. We can once again use a Pub/Sub service for this. Each one of the shards will subscribe to its own topic--we'll call these topics the Feed Notification Topics (FNT)--and the original subscribers S1 will be the publishers for the FNT. When S1 gets a new message about a post creation, it searches the main database for all of the users for whom this post is relevant (i.e., it searches for all of the friends of the user who created the post), it filters out users from other regions who will be taken care of asynchronously, and it maps the remaining users to the FNT using the same hashing function that our GetNewsFeed load balancers rely on. For posts that impact too many people, we can cap the number of FNT topics that get messaged to reduce the amount of internal traffic that gets generated from a single post. For those big users we can rely on the asynchronous feed creation to eventually kick in and let the post appear in feeds of users whom we've skipped when the feeds get refreshed manually. </br>

When CreatePost gets called and reaches our Pub/Sub subscribers, they'll send a message to another Pub/Sub topic that some forwarder service in between regions will subscribe to. The forwarder's job will be, as its name implies, to forward messages to other regions so as to replicate all of the CreatePost logic in other regions. Once the forwarder receives the message, it'll essentially mimic what would happen if that same CreatePost were called in another region, which will start the entire feed-update logic in those other regions. We can have some additional logic passed to the forwarder to prevent other regions being replicated to from notifying other regions about the CreatePost call in question, which would lead to an infinite chain of replications; in other words, we can make it such that only the region where the post originated from is in charge of notifying other regions. </br>


- GetNewsFeed Request </br>
The GetNewsFeed API call will most likely look like this:

<pre>
    GetNewsFeed(
        user_id: string,
        pageSize: integer,
        nextPageToken: integer,
    ) => (
        posts: []{
            user_id: string,
            post_id: string,
            post: data,
        },
        nextPageToken: string,
    )
</pre>

The pageSize and nextPageToken fields are used to paginate the newsfeed; pagination is necessary when dealing with large amounts of listed data, and since we'll likely want each news feed to have up to 1000 posts, pagination is very appropriate here. Finally, the load balancer will have the same hashing mechanism to get the userId hashed value, which is required to know where the load balancer should point to get the news feed for that user directly, this is absolutely important.

### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/bef1a634-4a5b-4fcc-9bf5-5e5444f2329b">
    </td>
  </tr>
</table>

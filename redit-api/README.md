# Redit API

### Explanation
This system was built with the following requirements in mind:
Build a system that:
- Is scalable.
- Fast and efficient
- Allow users write posts on subreddits, comment on posts, and upvote / downvote posts and comments.
- Allow to manage three entities (Posts, Comments, and Votes)
- Allow users to buy and give awards on Reddit.

</br>

1. Coming Up With A Plan </br>
The first major point of contention is whether to store votes only on Comments and Posts, and to cast votes by calling the EditComment and EditPost methods, or whether to store them as entirely separate entities--siblings of Posts and Comments, so to speak. Storing them as separate entities makes it much more straightforward to edit or remove a particular user's votes (by just calling EditVote, for instance), so we'll go with this approach.

We can then plan to tackle Posts, Comments, and Votes in this order, since they'll likely share some common structure. </br> 

2. Posts
Posts will have an id, the id of their creator (i.e., the user who writes them), the id of the subreddit that they're on, a description and a title, and a timestamp of when they're created.

Posts will also have a count of their votes, comments, and awards, to be displayed on the UI. We can imagine that some backend service will be calculating or updating these numbers when some of the Comment, Vote, and Award CRUD operations are performed.

Lastly, Posts will have optional deletedAt and currentVote fields. Subreddits display posts that have been removed with a special message; we can use the deletedAt field to accomplish this. The currentVote field will be used to display to a user whether or not they've cast a vote on a post. This field will likely be populated by the backend upon fetching Posts or when casting Votes.</br>

<pre>
    postId: string
    creatorId: string
    subredditId: string
    title: string
    description: string
    createdAt: timestamp
    votesCount: int
    commentsCount: int
    awardsCount: int
    deletedAt?: timestamp
    currentVote?: enum UP/DOWN
</pre>

Our CreatePost, EditPost, GetPost, and DeletePost methods will be very straightforward. One thing to note, however, is that all of these operations will take in the userId of the user performing them; this id, which will likely contain authentication information, will be used for ACL checks to see if the user performing the operations has the necessary permission(s) to do so.

<pre>
CreatePost(userId: string, subredditId: string, title: string, description: string)
  => Post

EditPost(userId: string, postId: string, title: string, description: string)
  => Post

GetPost(userId: string, postId: string)
  => Post

DeletePost(userId: string, postId: string)
  => Post
</pre>

Since we can expect to have hundreds, if not thousands, of posts on a given subreddit, our ListPosts method will have to be paginated. The method will take in optional pageSize and pageToken parameters and will return a list of posts of at most length pageSize as well as a nextPageToken--the token to be fed to the method to retrieve the next page of posts.

<pre>
ListPosts(userId: string, subredditId: string, pageSize?: int, pageToken?: string)
  => (Post[], nextPageToken?)
</pre>
</br>

3. Comments
Comments will be similar to Posts. They'll have an id, the id of their creator (i.e., the user who writes them), the id of the post that they're on, a content string, and the same other fields as Posts have. The only difference is that Comments will also have an optional parentId pointing to the parent post or parent comment of the comment. This id will allow the Reddit UI to reconstruct Comment trees to properly display (indent) replies. The UI can also sort comments within a reply thread by their createdAt timestamps or by their votesCount.

<pre>
    commentId: string
    creatorId: string
    postId: string
    createdAt: timestamp
    content: string
    votesCount: int
    awardsCount: int
    parentId?: string
    deletedAt?: timestamp
    currentVote?: enum UP/DOWN
</pre>

Our CRUD operations for Comments will be very similar to those for Posts, except that the CreateComment method will also take in an optional parentId pointing to the comment that it's replying to, if relevant.

<pre>
CreateComment(userId: string, postId: string, content: string, parentId?: string)
  => Comment

EditComment(userId: string, commentId: string, content: string)
  => Comment

GetComment(userId: string, commentId: string)
  => Comment

DeleteComment(userId: string, commentId: string)
  => Comment

ListComments(userId: string, postId: string, pageSize?: int, pageToken?: string)
  => (Comment[], nextPageToken?)
</pre>

4. Votes
Votes will have an id, the id of their creator (i.e., user who casts them), the id of their target (i.e., the post or comment that they're on), and a type, which will be a simple UP/DOWN enum. They could also have a createdAt timestamp for good measure.</br>

<pre>
    voteId: string
    creatorId: string
    targetId: string
    type: enum UP/DOWN
    createdAt: timestamp
</pre>

Since it doesn't seem like getting a single vote or listing votes would be very useful for our feature, we'll skip designing those endpoints (though they would be straightforward).

Our CreateVote, EditVote, and DeleteVote methods will be simple and useful. The CreateVote method will be used when a user casts a new vote on a post or comment; the EditVote method will be used when a user has already cast a vote on a post or comment and casts the opposite vote on that same post or comment; and the DeleteVote method will be used when a user has already cast a vote on a post or comment and just removes that same vote.</br>

<pre>
CreateVote(userId: string, targetId: string, type: enum UP/DOWN)
  => Vote

EditVote(userId: string, voteId: string, type: enum UP/DOWN)
  => Vote

DeleteVote(userId: string, voteId: string)
  => Vote
</pre>

5. Awards
We can define two simple endpoints to handle buying and giving awards. The endpoint to buy awards will take in a paymentToken, which will be a string that contains all of the necessary information to process a payment. The endpoint to give an award will take in a targetId, which will be the id of the post or comment that the award is being given to. </br>

<pre>
BuyAwards(userId: string, paymentToken: string, quantity: int)

GiveAward(userId: string, targetId: string)
</pre>


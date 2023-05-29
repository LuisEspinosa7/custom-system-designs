# Slack Core System

### Explanation
This system was built with the following requirements in mind:
Build a system that:
- Is scalable.
- Fast and efficient
- Has low latency
- Loads the most recent messages in a Slack channel when a user clicks on the channel.
- Immediately sees which channels have unread messages for a particular user when that user loads Slack.
- Immediately sees which channels have unread mentions of a particular user, when that user loads Slack, the number of these unread 
mentions will be on each relevant channel besides the channel name.
- Maintains registry of last message seen and last channel interaction.
- Real time chating
- Multi device synchronization
- Allows 20 million users
- Allows organizations with up to 50.000 users
- Allows channels with up to 50.000 users

This system consist of two main parts:

- Slack app loads </br>
Whenever the application loads, all clients will send requests through a load balancer which will have communication with cluster of API Servers that interacts with the database. Since our application will handle such an amount of organizations, channels and users, it becomes a good practice to take a smart sharding approach for persisting the application data, in this case the shards will be created depending on organizations size (messages stored for each organization), we can have the biggest organizations (with the biggest channels) in their individual shards, and we can have smaller organizations grouped together in other shards. Over time, organization sizes and Slack activity within organizations will change. Some organizations might double in size overnight, others might experience seemingly random surges of activity, etc.. This means that, despite our relatively sound sharding strategy, we might still run into hot spots, which is very bad considering the fact that we care about latency so much.
</br>
To handle this, we can add a "smart" sharding solution: a subsystem of our system that'll asynchronously measure organization activity and "rebalance" shards accordingly. This service can be a strongly consistent key-value store like Etcd or ZooKeeper, mapping orgIds to shards. Our API servers will communicate with this service to know which shard to route requests to.
</br>
Since the application data is relation, therefore there will structure data to represent the business data, we'll need an storage solution in this case and SQL database. We have the following tables: </br>
- Channels: To store the the channels </br>
- Channel members: Holds users who is in a particular channel. We'll use this table, along with the one above, to fetch a user's relevant when the app loads. </br>
- Messages: To store all historical messages sent on Slack. This will be our largest table, and it'll be queried every time a user fetches messages in a particular channel. The API endpoint that'll interact with this table will return a paginated response, since we'll typically only want the 50 or 100 most recent messages per channel. Also, this table will only be queried when a user clicks on a channel; we don't want to fetch messages for all of a user's channels on app load, since users will likely never look at most of their channels. </br>
- Latest Channel Timestamps: Stores the latest activity in each channel (this table will be updated whenever a user sends a message in a channel) </br>
- Channel Read Receipts: Stores the last time a particular user has read a channel (this table will be updated whenever a user opens a channel). </br>
NOTE: Last two tables are meant not to fetch recent messages for every channel on app load, while supporting the feature of showing which channels have unread messages. </br>
- Unread Channel-User-Mention Counts: For the number of unread user mentions that we want to display next to channel names, we'll have another table similar to the read-receipts one, except this one will have a count of unread user mentions instead of a timestamp. This count will be updated (incremented) whenever a user tags another user in a channel message, and it'll also be updated (reset to 0) whenever a user opens a channel with unread mentions of themself. 

</br>

Databases </br>
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/b80833e7-27f9-4680-ae00-bd90d8eb3a4a">
    </td>
  </tr>
</table>
</br>

- Real-time messaging as well as cross-device synchronization. </br>
The real time communication strongly rely on pub/sub messaging supported by a smart sharding strategy. There are kafka topics per organization
shared by size, this is to say, if an organization is really big, it will have its own shard, conversely if an organization is small it will for sure
be sharded together along with others. Therefore, every Slack organization or group of organizations will be assigned to a Kafka topic, and whenever a user sends a message in a channel or marks a channel as read, the API servers in the cluster will send a Pub/Sub message to the appropriate Kafka topic after persisting on the database. </br>

Example of new message kafka event  </br>
{
  "type": "chat",
  "orgId": "AAA",
  "channelId": "BBB",
  "userId": "CCC",
  "messageId": "DDD",
  "timestamp": "2020-08-31T01:17:02",
  "body": "this is a message",
  "mentions": ["CCC", "EEE"]
}
</br>
</br>
Example of read channel kafka event  </br>
{
  "type": "read-receipt",
  "orgId": "AAA",
  "channelId": "BBB",
  "userId": "CCC",
  "timestamp": "2020-08-31T01:17:02"
}
</br>

All clients will be using web sockets for long TCP communication, their connections will go through a load balancer which request the smart sharding tool like ETCD to know which API SERVER to point to, right after the api server sharding id is obtained the request will go ahead a hit that server
which of course will be listening to a specific kafka topic. When clients receive Pub/Sub messages, they'll handle them accordingly (mark a channel as unread, for example), and if the clients refresh their browser or their mobile app, they'll go through the entire "on app load" system that we described earlier.


### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/7708f71b-9bd9-4e63-a1aa-01d8fc7bb0e0">
    </td>
  </tr>
</table>

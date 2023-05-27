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


    Seeing all of the channels that a user is a part of.
    Seeing messages in a particular channel.
    Seeing which channels have unread messages.
    Seeing which channels have unread mentions and how many they have.



Databases
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="CCC">
    </td>
  </tr>
</table>


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
Example of read channel kafka event  </br>
{
  "type": "read-receipt",
  "orgId": "AAA",
  "channelId": "BBB",
  "userId": "CCC",
  "timestamp": "2020-08-31T01:17:02"
}

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

# AirBnB Core System

### Explanation
This system was built with the following requirements in mind:
Build a system that:
- Is scalable.
- Fast and efficient
- Allows Hosts create and delete listings (properties)
- Allows Renters list all available listings (neither reserved nor booked)
- Allows Renters open a specific listing
- Allows Renters reserve a listing which available (neither reserved nor booked)
- Is efficient but memory responsible 
- Allows 1 million listings
- Allows 50 million users (listing and reserving)

This system consist of two main parts:

- Host </br>
Hosts are able to create and delete listings, for the first they have to provide the title, description and thumbnail url, all the thousands 
requests will go through a load balancer and a cluster of EC2 machines running a Java or Python application responsible for creating the listing
on the LISTINGS DATABASE, right after they are required to inform the IN-MEMORY QUADTREE CLUSTER, so it can keep up with the database by pulling 
the data again (Synchronous synchronization). On the other hand, the delete listing follows the same approach but instead of creating, it disable
the record (soft deleting), indicating is not valid anymore, of course the QUADTREE must be informed. 
Since the IN-MEMORY QUADTREE CLUSTER is
fundamental for this system and represents the core of it, I consider fundamental to show its behaviour and a respective example. The QUADTREE is
a datastructure to to store data in by dividing in different zones depending on some data characteristics, then whenever we want to access that
data, we'll be able to come up with it way faster than from a database, and much more if it is store in MEMORY. So please imagine that OUR CLUSTER 
of QUADTREE implementations will be composed of several FOLLOWERS and one and only one LEADER (selected by leader election), the latter will be 
assigned the responsibility of upgrading its data from the LISTINGS DATABASE whenever we have a new listing or one of them deleted 
(Synchronous synchronization). The followers will upgrade themselves from the LEADER (Asynchronous synchronization). The following example will 
make it clearer:

We can call this a geo index, and let's see if we can fit the listings there. Assuming a single listing takes up roughly 10 KB of space (as an upper bound), some simple math confirms that we can store everything we need about listings in memory.

~10 KB per listing
~1 million listings
~10 KB * 1000^2 = 10 GB

<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/7ae6e2ce-cbfd-4a4e-a1a9-36f4611bb417">
    </td>
  </tr>
</table>

To finish, every QUADTREE element will have the LISTINGS table record info (listing basic information) and additionally will have the list of 
UNAVAILABLE date ranges (explain later on).


- Renters </br>
Renter can either list all listings filtered by the location and date ranges, load an individual listing (selected from the listings) and finally 
reserve that listing. For listing all listings all requests go through a SINGLE PATH BASED LOAD BALANCER SHARED BY all the three endpoints for 
RENTERS, then we have the specific cluster of machines running applications responsible of requesting the LEADER INSTANCE on the QUADTREE, 
this will use A BINARY SEARCH ON THE QUADTREE to find the listings in that location BUT which data ranges are not present or come accross the 
QUADTREE's listing list of NOT AVAILABLE date ranges, this is to say, to look for only the listing in that area available. So for instance if one
listing is inside the area of interest but its NOT AVAILABLE DATE RANGES (already occupied date ranges) coincide with the date ranges searched 
by the user, that listing will be ignored or skipped. Additionally the listings will be paginated just to avoid overloading the clients with huge 
amounts of listings, it will be controlled by an offset. Finding relevant locations should be fairly straightforward and very fast, especially since we can estimate that our quadtree will have a depth of approximately 10, since 4^10 is greater than 1 million (required listings in memory). 
Then we have the individual loading of a listing, this is the simplest of all because it will only go through the load balancer and the cluster of 
applications to finally look for that listing directly in the database.
Finally we have RESERVING action from the user, this one is special, as the others it will go through the same LOAD BALANCER, then the cluster of 
applications, right after those applications will create a record on the RESERVATIONS DATABASE with the LISTING ID, INIT_DATE_RANGE, 
FINAL_DATE_RANGE and expiration (usually 15 minutes, if inside this time, the listing is only reserved but not booked), as long as this records 
has the expiration column value in a VALID STATE (not surpassed yet the time,  therefore inside the 15 minutes) the listing WON'T APPEAR to the
rest of the RENTERS, why? because after this record's been created, the applications have to inform the LEADER in the QUADTREE to add those 
NOT AVAILABLE DATE RANGES for that particular listing in the MEMORY QUADTREE (Synchronous synchronization). If the client do not pay during the 
expiration time, the date ranges on that particular listing in the quadtree will be deleted,  therefore available again to everyone else, but 
if the client payed successfuly this column value will be assigned null on the database and the not available date ranges will be kept.


### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/ac0114a5-9506-4276-8fbc-95a4d1447eab">
    </td>
  </tr>
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/253343b0-bba9-4e6d-a661-80fb16c2db8c">
    </td>
  </tr>
</table>

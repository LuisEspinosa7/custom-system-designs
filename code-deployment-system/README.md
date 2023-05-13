# Highly Available Code Deployment System

### Explanation
This system was built with the following requirements in mind:
Build a system that:
- Is global and fast
- Takes code, builds it into a binary, and deploys the result globally in an efficient and scalable way.
- After the code has been merged into the master branch of a central code repository, engineers should be able to trigger a build and deploy that build through an UI. (No code reviews supported)
- Builds the binary and deploys it around the world.
- Scales massively to hundreds of thousands of machines spread across 5-10 regions throughout the world.
- Is internal, has a decent availability, is fault tolerant.
- Eventually after a build reach a SUCCESS or FAILURE state. (clear end-state for builds)
- Once a binary has been successfully built, it should be shippable to all machines globally within 30 minutes.
- Builds code within no more than 15 minutes, and that can replicate binaries of sizes up to 10 GB
- Can take commits with SHA identifiers from a code repo.

This system consist of two main parts:

- Build </br>
Engineers have an user interface to create the jobs, this requests are taken by the api gateway and lambdas to finally create the jobs on the 
database. Then naturally there are an autoscaling group of workers responsible for building the binary, they will be using the FIRST DATABASE
TRANSACTION to get the jobs in QUEUE status (available jobs), once a worker detect an available job, immediately uses the SECOND DATABASE 
TRANSACTION to change the job status to RUNNING. Then with job details (name, repository, version, status) the code is gathered from the repository
and the build starts. One important worker responsibility is to use the THIRD DATABASE TRANSACTION to update the heartbeat column in the job row.
Provided the worker has build the binary successfully, the work  must use the FOURTH DATABASE TRANSACTION to update the job status to completed. 
With the purpose of being consistent there has to be a monitor application running all the time which duty would be to get the heartbeat of all the jobs and look for times greater than 5 or 10 minutes, those cases may be taken as failures on dead workers, maybe issues with network, so on and so forth, in that specific case the monitor might use the FIFTH DATABASE TRANSACTION to update the job's status to QUEUE (so they can be taken again by oter working workers). Once a binary has been built by any worker successfully, of course after updating the job status to COMPLETED, the worker will 
upload the binary (up to 10 Gb) to the MAIN BLOB STORAGE (s3), which has a predefined multi-region replication feature, that has to take no more than 10 minutes (using a P2P network). 

- Deploy </br>
There is an application called the synchronizer borned to accomplish three tasks, the first one is use the DATABASE TRANSACTION NUMBER 6 to get the completed jobs, the second is to check on regional blob storages to see if the binary has been already replicated and the third would be to update a NEW DATABASE called BUILDS DATABASE, specifically update the replication status of that binary. This checking should be executed at least every minute. On the other hand, engineers will have another interface through which the job's status can be visualized (lambdas read the build database) and the deployment process can be started either, only if a job was successfully completed, a deploy button may be activated, with this option the use is able to start the work of deploy the binary throughtout world.
The user clic the deploy button, the request is attended by the api gateway and lambdas which communicates with the build database and also with the ETCD key-value database responsible of holding the build name and corresponding version, this is CRUTIAL, because the version that the user confirms to be deloyed will be saved on ETCD and immediately replicated to the ETCD instance on each regional cluster of machines.
Finally every regional cluster will have a replication manager application that will be reading the regional instance of ETCD and comparing the current version of the binary runnning on that cluster of machines, if the version do not coincide, the machine will try to get the new one from their regional S3 blob storage and send distribute the binary throught all the machines per cluster using a P2P network as fast as possible. Finally the binary should be deployed successfully on each cluster machine. 


### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/c1cdb2f4-2f73-4004-bd8c-7def251ca81c">
    </td>
  </tr>
</table>

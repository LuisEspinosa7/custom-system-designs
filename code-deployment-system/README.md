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

- Build
Engineers have an user interface to create the jobs, this requests are taken by the api gateway and lambdas to finally create the jobs on the 
database. Then naturally there are an autoscaling group of workers responsible for building the binary, they will be using the FIRST DATABASE
TRANSACTION to get the jobs in QUEUE status (available jobs), once a worker detect an available job, immediately uses the SECOND DATABASE 
TRANSACTION to change the job status to RUNNING. Then with job details (name, repository, version, status) the code is gathered from the repository
and the build starts. One important worker responsibility is to use the THIRD DATABASE TRANSACTION to update the heartbeat column in the job row.
Provided the worker has build the binary successfully, the work  must use the FOURTH DATABASE TRANSACTION to update the job status to completed. 
With the purpose of being consistent there has to be a monitor application running all the time which duty would be to get the heartbeat of all the jobs and look for times greater than 5 or 10 minutes, those cases may be taken as failures on dead workers, maybe issues with network, so on and so forth, in that specific case the monitor might use the FIFTH DATABASE TRANSACTION to update the job's status to QUEUE (so they can be taken again by oter working workers). 


- Deploy

### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/c1cdb2f4-2f73-4004-bd8c-7def251ca81c">
    </td>
  </tr>
</table>

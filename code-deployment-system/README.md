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



- Deploy

### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/c1cdb2f4-2f73-4004-bd8c-7def251ca81c">
    </td>
  </tr>
</table>

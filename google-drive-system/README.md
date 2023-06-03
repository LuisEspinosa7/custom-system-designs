# Google Drive Core System

### Explanation
This system was built with the following requirements in mind:
Build a system that:
- Is scalable.
- Fast and efficient
- Simulates Google Drive web application
- Allow users to create folders, upload and download files, and rename and move entities once they're stored.
- Allows 1 billion users
- Each user with 15GB of data stored in Google Drive on average (15,000 PB of data in total)
- Allows metadata for each entity (folders and files), like its name or its type. 
- Is Highly Available and also very redundant. No data that's successfully stored in Google Drive can ever be lost

</br>

1. Coming Up With A Plan </br>
we'll need to support the following operations: </br>
<pre>
    For Files
        UploadFile
        DownloadFile
        DeleteFile
        RenameFile
        MoveFile
    For Folders
        CreateFolder
        GetFolder
        DeleteFolder
        RenameFolder
        MoveFolder
</pre>
</br>
Secondly, we'll have to come up with a proper storage solution for two types of data:</br>

<pre>
    Fissle Contents: The contents of the files uploaded to Google Drive. These are opaque bytes with no particular structure or format.
    Entity Info: The metadata for each entity. This might include fields like entityID, ownerID, lastModified, entityName, entityType. This list is non-exhaustive, and we'll most likely add to it later on.
</pre>

Let's start by going over the storage solutions that we want to use, and then we'll go through what happens when each of the operations outlined above is performed. </br>


2. Storing Entity Info </br>
To store entity information, we can use key-value stores. Since we need high availability and data replication, we need to use something like Etcd, Zookeeper, or Google Cloud Spanner (as a K-V store) that gives us both of those guarantees as well as consistency (as opposed to DynamoDB, for instance, which would give us only eventual consistency). </br>

Since we're going to be dealing with many gigabytes of entity information (given that we're serving a billion users), we'll need to shard this data across multiple clusters of these K-V stores. Sharding on entityID means that we'll lose the ability to perform batch operations, which these key-value stores give us out of the box and which we'll need when we move entities around (for instance, moving a file from one folder to another would involve editing the metadata of 3 entities; if they were located in 3 different shards that wouldn't be great). Instead, we can shard based on the ownerID of the entity, which means that we can edit the metadata of multiple entities atomically with a transaction, so long as the entities belong to the same user. </br>

Given the traffic that this website needs to serve, we can have a layer of proxies for entity information, load balanced on a hash of the ownerID. The proxies could have some caching, as well as perform ACL checks when we eventually decide to support them. The proxies would live at the regional level, whereas the source-of-truth key-value stores would be accessed globally. </br>

3. Storing File Data </br>
When dealing with potentially very large uploads and data storage, it's often advantageous to split up data into blobs that can be pieced back together to form the original data. When uploading a file, the request will be load balanced across multiple servers that we'll call "blob splitters", and these blob splitters will have the job of splitting files into blobs and storing these blobs in some global blob-storage solution like GCS or S3. </br>

One thing to keep in mind is that we need a lot of redundancy for the data that we're uploading in order to prevent data loss. So we'll probably want to adopt a strategy like: try pushing to 3 different GCS buckets and consider a write successful only if it went through in at least 2 buckets. This way we always have redundancy without necessarily sacrificing availability. In the background, we can have an extra service in charge of further replicating the data to other buckets in an async manner. For our main 3 buckets, we'll want to pick buckets in 3 different availability zones to avoid having all of our redundant storage get wiped out by potential catastrophic failures in the event of a natural disaster or huge power outage. </br>

In order to avoid having multiple identical blobs stored in our blob stores, we'll name the blobs after a hash of their content. This technique is called Content-Addressable Storage, and by using it, we essentially make all blobs immutable in storage. When a file changes, we simply upload the entire new resulting blobs under their new names computed by hashing their new contents. </br>

This immutability is very powerful, in part because it means that we can very easily introduce a caching layer between the blob splitters and the buckets, without worrying about keeping caches in sync with the main source of truth when edits are made--an edit just means that we're dealing with a completely different blob. </br>


4. Entity Info Structure </br>
Since folders and files will both have common bits of metadata, we can have them share the same structure. The difference will be that folders will have an is_folder flag set to true and a list of children_ids, which will point to the entity information for the folders and files within the folder in question. Files will have an is_folder flag set to false and a blobs field, which will have the IDs of all of the blobs that make up the data within the relevant file. Both entities can also have a parent_id field, which will point to the entity information of the entity's parent folder. This will help us quickly find parents when moving files and folders. </br>

<pre>
    File Info

    {
      blobs: ['blob_content_hash_0', 'blob_content_hash_1'],
      id: 'some_unique_entity_id'
      is_folder: false,
      name: 'some_file_name',
      owner_id: 'id_of_owner',
      parent_id: 'id_of_parent',
    }
</pre>

<pre>
    Folder Info

    {
      children_ids: ['id_of_child_0', 'id_of_child_1'],
      id: 'some_unique_entity_id'
      is_folder: true,
      name: 'some_folder_name',
      owner_id: 'id_of_owner',
      parent_id: 'id_of_parent',
    }
</pre>

5. Garbage Collection </br>
 Any change to an existing file will create a whole new blob and de-reference the old one. Furthermore, any deleted file will also de-reference the file's blobs. This means that we'll eventually end up with a lot of orphaned blobs that are basically unused and taking up storage for no reason. We'll need a way to get rid of these blobs to free some space.

We can have a Garbage Collection service that watches the entity-info K-V stores and keeps counts of the number of times every blob is referenced by files; these counts can be stored in a SQL table.

Reference counts will get updated whenever files are uploaded and deleted. When the reference count for a particular blob reaches 0, the Garbage Collector can mark the blob in question as orphaned in the relevant blob stores, and the blob will be safely deleted after some time if it hasn't been accessed. </br>


6. End To End API Flow [WORKING FLOW] </br>
 Now that we've designed the entire system, we can walk through what happens when a user performs any of the operations we listed above.

CreateFolder is simple; since folders don't have a blob-storage component, creating a folder just involves storing some metadata in our key-value stores.

UploadFile works in two steps. The first is to store the blobs that make up the file in the blob storage. Once the blobs are persisted, we can create the file-info object, store the blob-content hashes inside its blobs field, and write this metadata to our key-value stores.

DownloadFile fetches the file's metadata from our key-value stores given the file's ID. The metadata contains the hashes of all of the blobs that make up the content of the file, which we can use to fetch all of the blobs from blob storage. We can then assemble them into the file and save it onto local disk.

All of the Get, Rename, Move, and Delete operations atomically change the metadata of one or several entities within our key-value stores using the transaction guarantees that they give us.  </br>

### Pictures
<table style="width:100%">
  <tr>
    <td>
  	<img width="950" alt="Image" src="https://github.com/LuisEspinosa7/custom-system-designs/assets/56041525/e24f8c52-4a80-413c-9183-fde81523545f">
    </td>
  </tr>
</table>

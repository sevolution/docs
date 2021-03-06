#Synchronization types

The synchronization is triggered by a file modification or executed as a periodic task. In order to minimize the amount of data necessary to synchronize a file, 
RavenFS determines the kind of file differences to figure out what type of work needs to be done. It is implemented by comparing file metadata from both file systems, 
which means that the source needs to retrieve metadata from the destination before it starts to push any bytes. There are four types of synchronizations:

* [Content update](#content-update)
* [Metadata update](#metadata-update)
* [Rename](#rename)
* [Delete](#delete)


##Content update

There are two types of actions that are considered to be content updates:

* a new file upload,
* an upload of an existing file with changed content.

The first case is simple, as we have to transfer the entire file. The second scenario is much more interesting - RavenFS uses Remote Differential Compression to handle that.

###How to detect that content has changed?

Behind the scenes RavenFS calculates a file hash based on its content every time you upload it. The result is stored under the `Content-MD5` key in the file metadata.
The source file system is able to determine whether the contents of the local file and a the remote one are different just by comparing these metadata records.

###Remote Differential Compression

In order to synchronize large files in an efficient way and minimize the amount of data transferred between the source and the destination files systems, RavenFS uses Remote Differential Compression (RDC).
It is the built-in Windows feature. Below there are presented basics of RDC algorithms that will allow you to understand how the file content synchronization works.
To explore RDC in details click [here](https://msdn.microsoft.com/en-us/library/dd357428%28v=prot.20%29.aspx).

####Overview

Remote Differential Compression detects which parts of the two files are the same, and which are different. The result of RDC detection is the list
called *a needs list*, which contains items that describe how to construct the target file on a destination system by using chunks sent by the source and parts
of the file already existing there. Each element of the needs list consists of the accurate byte range and the information concerning the file version (*source* or *seed*).
Based on the needs list the source server pushes chunks of the file marked as *source* to the destination. The transferred data and the existing file chunks
(marked as *seed* in the list) are combined according to the order in which they appear on the needs list to create the synchronized file.

RDC is able to calculate the mentioned byte ranges very precisely, which allows for the send data reduction, especially when a small part of the file is changed
(it doesn't matter if it happens at the beginning / end or in the middle of a file).

####Signatures

RDC divides a file into chunks. Each chunk has a hash value assigned, which, together with the chunk size, creates a signature. A collection of file signatures 
contains all the information about a file's content. The source retrieves information about the file content existing on the destination by downloading its signatures.
By comparing its own signatures with the downloaded ones, the source is able to generate the needs list.

{INFO: Signatures synchronization}
A really large file can have very big signatures. Therefore, they aren't directly downloaded. RavenFS internally synchronizes the signatures to speed up the entire operation and reduce the amount of the exchanged data (the signature is always smaller than its file).
{INFO/}

###HTTP request format

An upload of the missing file chunks between the source and the destination file systems is performed by HTTP multipart POST message. 
The destination server exposes the `/synchronization/MultipartProceed` endpoint which accepts only RavenFS specific formatted MIME multipart content. 
Below there is a sample synchronization request sent by the server (the `Content-Type` request header has to be set to `multipart/form-data; boundary=syncing`).

	--syncing   
	Content-Disposition: form-data; Syncing-need-type=seed; Syncing-range-from=0; Syncing-range-to=407029   
	Content-Type: plain/text   

	--syncing   
	Content-Disposition: file; Syncing-need-type=source; Syncing-range-from=407030; Syncing-range-to=412242   
	Content-Type: application/octet-stream   
   
	[... data from byte 407030 to 412242 goes here...]   
   
	--syncing--

Note that the first seed part is empty because it involves the destination file chunk. It contains only the information about byte range that needs to be copied from the existing file.
The second one is *source* part and it contains a range of file content bytes.

If the file does not exist on the destination file system, then the synchronization request consists of one source part that contains entire file data.

###Temporary sync file

RavenFS assumes that an error during the synchronization operation may happen at any time. To avoid the scenario in which the failed synchronization damages the synced file on the destination system, target file is built under the `[FILENAME].downloading` name. If the synchronization finishes without any error, then the `[FILENAME]` file will be deleted and the `[FILENAME].downloading` will be renamed to `[FILENAME]`.


##Metadata update

If the difference between the source file and the destination file is only in the metadata, the source system creates a POST request just with file metadata placed in the request headers.
The destination server has the dedicated endpoint for that: `/synchronization/updateMetadata`. The sent metadata will override the existing one while `ETag` and `Last-Modified` 
will get new values (as usual after every successful synchronization).

##Rename

RavenFS is able to recognize that the file was renamed and in order to reflect that change on the destinations it doesn't need to transfer the file content at all. 
Synchronizing the rename operation is forcing destination file systems to rename the file according to a given name. The appropriate HTTP endpoint on a destination server is `/synchronization/rename`.

##Delete

To deal with synchronization of deletions, the file system keeps tombstones with a delete marker for each deleted file. Then it is able to determine that the destination node has files that were already deleted on the source.
The delete synchronization sends a POST request to the destination server to the `/synchronization/delete` endpoint.
 
# SubRocks PHP Uploader

An outline for a PHP-headed video upload utility for the video-sharing website, SubRocks.

## **Goal:**

Create a video upload utility via HTTP using and PHP, FFmpeg, and SQL. Preferably exceed the quality of FulpQueue. 

This will be pulled off using a great deal of communication between the four interfaces. Due to limitations currently in place, a two-ended connection such as WebSocket is not possible. In order to compensate, the client will only likely accidentally denial of service attack the server with the vast amount of requests, coupled with the negative upload of the server. Additionally, this does not interruptions in the process due to a false trigger of Cloudflare security measures, due to said vast number of requests. For some reason, the subrock.rocks domain name does not even have an associated mail server, so an assertion that using a WebSocket server cannot be used is not out of the question. Theoretically, if all goes well, then this will provide a greater uploading experience over the predecessent FulpQueue server.

## **SQL set-up:**

In order for this to work, the "fulptube" database must be extended to contain an upload temporary storage table (hereby, "uploader"). This will store metadata for the video during the upload process, including a unique uploader key used to identify the video on both the client and server ends. Each time this table is updated, the ping cell is updated to be equivalent to the modification date, in order to aid with automated cleanup. The table is quite simple, and only needs to store six columns per instance.

| ***Column***      |          init          |         ping        |       key       |  publicid |      chunks      |    metadata   |     error    |
|-------------------|:----------------------:|:-------------------:|:---------------:|:---------:|:----------------:|:-------------:|:------------:|
| ***Description*** | Time of initialisation | Time of last update | Interaction key | Public ID | Number of chunks | Metadata JSON | Error status |

The interaction key will be a long, random string of 32 or more characters, such as that of a one way hash; this is harder to intercept unlike the public ID. The interaction key will never be displayed by the client, unlike the public ID, which is expected to be. In contrast, the public ID will be a shorter random string of 11 characters; this is the ID used in public video web addresses, and as such, this is generated as early as possible to create a permanent URL that the end-user can share.

The number of chunks is stored in the database as early as possible to aid with the first phase of processing: to fuse all chunks back into the original binary.

The metadata JSON contains all user-selected metadata, and is updated frequently during the upload process to reflect changes by the client. Here is an example of the metadata JSON:
```js
{
    "title": "Example Video",
    "description": "This is an example video",
    "thumbnail": "default.png",
    "tags": ["example", "test"],
    "category": "None",
    "allowCommenting": true,
    "allowRatings": true
}
```

## **Client set-up:**

When an upload is initialised by the user, the client sends a request to the server to generate two important bits of information and return them: an interaction key and a public video identifier. The interaction key will be used to more securely interact with the upload process, whilst the video ID will be used to display a permanent link to the video to the end-user. The client is also responsible for chunking the quite possibly large video binary into smaller, approximately 50 MB chunks which can be sent individually to the server. Once an upload interaction key is retrieved, the client can begin uploading each chunk to the server. Progress will be reported at this stage solely by the client. Once all chunks are uploaded, the client switches communication ends with the server, where it will ping the server at a few seconds long interval asking for progress reports, which will be reported back in the client interface. Additionally, metadata changes will be actively reported to the server at regular intervals during this stage, which will adjust the metadata cell in the database table.

## **PHP set-up:**

The entire process thus far described must be held together by a collection of PHP scripts that will bear responsibility for taking the video file from end-user to publicly accessible on the website. Video processing, including optimisation for the web, will be performed using FFmpeg. URLs for the upload API must provide the interaction key of the video in order to work; these will, then, make adjustments to the SQL table, alongside other tasks. Every API interaction will update the ping cell in the database table to equate to the date of modification, or for practical purposes, the current date. The PHP and client ends will both closely communicate during the processing phase of uploading, in which the client will consistently ping the server at an interval of a few seconds requesting for a progress report to be returned by the server. After the processing process is done, the video will be moved into the public database; its metadata JSON will be parsed alongside its public ID and final file location, and the process should be done. In the case of a fatal error, the uploader will report this to the client on the next request, and proceed to clean up all temporary elements used.

Using task scheduling on the server, another script can be created for automated cleanup in the case of an uncaught error. It will iterate through the entire uploader database and compare the latest ping times, removing everything related to the ones that haven't been touched in a while.

## **Closing statements:**

This task is much more complicated to execute than theorised. Emulating two-way communication with consistent pinging over HTTP, first off, is bound to go poorly with protective applications, such as Cloudflare. Establishing a real two-way connection via a different protocol, such as WebSocket, is a much better way to go about this. This may also eliminate the need for splitting file sizes to get around upload size limits imposed by other factors. However, this is a possible option as a last resort, and it is quite possibly the easiest method to pull this task off, given the limitations of SubRocks at the moment. Of course, the greatest issue with uploading, or steaming video for that moment, is the immensely slow upload speed imposed on the SubRocks servers by the owner's ISP.

# Servless Infrastructure-media-convert-and-delievery in Cloud Formation
This Serverless Soultion is about providing encoded videos for different formats from a single source file
Scenario:
We need to build a solution for customer where they can display their videos with the ability of downloading that videos into different formats. So, users are able to watch these videos on their compatible devices in terms of video format (resolution, source media, internet speed etc.). Users also should be able to express their thoughts about video by liking or disliking the video.
We need to build this solution serverless and auto scalable as much as possible and automate the deploying process. So, customer can use this by minimum efforts.

![image](https://user-images.githubusercontent.com/95842706/151727588-2699edf6-5bfa-4c7f-a6fc-5c8acfcf3cd6.png)
![image](https://user-images.githubusercontent.com/95842706/151727599-0e4d5404-ee5f-4fcc-983f-2e2055a1c86f.png)

This solution is based on a workflow so all the process from start to end is divided into three distinctive pieces. I’ll explain the above architecture in more detail.
1.	Source Bucket: First thing, we uploading video file in source bucket and for that, we can use pre-signed URL for a person who don’t have access to AWS account otherwise we can upload through console. Uploading the file in bucket will trigger the work flow. At this point we have two option to trigger the work flow, either video file or metadata file.

a)	Source Video Option: If deployed with the workflow trigger parameter set to Video File, the CloudFormation template will configure S3 event notifications on the source S3 bucket to trigger the workflow whenever a video file (mpg, mp4, m4v, mov, or m2ts) is uploaded.

b)	Source Metadata Option: If deployed with the workflow trigger parameter set to Metadata File, the S3 notification is configured to trigger the workflow whenever a JSON file is uploaded. This allows different workflow configuration to be defined for each source video processed by the workflow.
Full list of options in Metadata File: (only required field is srcVideo)


{
    "srcVideo": "string",
    "archiveSource": string,
    "frameCapture": boolean,
    "srcBucket": "string",
    "destBucket": "string",
    "cloudFront": "string",
    "jobTemplate_2160p": "string",
    "jobTemplate_1080p": "string",
    "jobTemplate_720p": "string",
    "jobTemplate": "custom-job-template",
    "inputRotate": "DEGREE_0|DEGREES_90|DEGREES_180|DEGREES_270|AUTO",
    "captions": {
        "srcFile": "string",
        "fontSize": integer,
        "fontColor": "WHITE|BLACK|YELLOW|RED|GREEN|BLUE"
    }
}


It also supports adding additional metadata, such as title, genre, or any other information, we want to store in Amazon DynamoDB.


2. Ingest (AWS Step Function):  After we upload video in S3, this will kick off our ingest process. That process will validate the input and check we can get access to it and workflow is able to get to that content, then we run an open source software “Media info” which will extract metadata from video file. Once we have that information, we will store it into our DynamoDB table. At the end of ingest process we will send a SNS notification to uploader that file has been uploaded successfully.

3. Process (Media Covert):  Same again, we using step function. In this step, we are doing two simple things. First thing is “Profiler” step, which going to take that metadata which we stored in DynamoDB in last step and its going to look at the resolution of the video and based on that resolution, its going to set an encoding profile.

4. Lambda Functions:  There are few lambda functions here which providing the outputs for work flow by step functions in Step-1 and Step-2 in architecture.
	archive-source: Lambda function to tag the source video in s3 to enable the Glacier lifecycle policy.
	
    custom-resource: Lambda backed CloudFormation custom resource to deploy MediaConvert templates configure S3 event notifications.
	
    dynamo: Lambda function to Update DynamoDB.
	
    encode: Lambda function to submit an encoding job to Elemental MediaConvert.
	
    error-handler: Lambda function to handler any errors created by the workflow or MediaConvert.
	
    input-validate: Lambda function to parse S3 event notifications and define the workflow parameters.
	
    
    media-package-assets: Lambda function to ingest an asset into Media Package-VOD.
	
    output-validate: Lambda function to parse MediaConvert CloudWatch Events.
	
    profiler: Lambda function used to send publish and/or error notifications.
	
    step-functions: Lambda function to trigger AWS Step Functions.
    
    Lambda-function for likes/dislikes: will update dynamoDB with like/dislikes number of the videos and return the updated number
    
    Pre-signed URL: will generate pre-signed URL for uploading the source file
    
    Query: will search dynamoDB for searching videos by Title, catagory or description of the video

5. DynamoDB:  will store the meta data of video file which will be provided by lambda after ingesting in step-2 of architecture and will update the table after processing the media convert job in step-3 of architecture.

6. Cloud Watch:  After encoding job finished, it will send logs in cloud watch

7. SNS:  This step is associated with publishing step function in this work flow. When encoding job is finished, its gonna store that in S3. In cloud watch when it sees “Media job is completed” then its gonna kick off publishing step function which will go through and validate all the content that we were expecting is there in S3. Then its going to generate cloud front links for deliver the contents.

8. AWS Elemental Media Package:  This is an alternative way to delivering the content in this solution. By default, we doing single file in and we are creating different encoding i.e. hls, dash, mp4 and all the bit rate version of the different formats. However, with media package, we can create just one set of content. Foer example, we can create a single set of HLS content and then use that “Media Package” to ingest that and it will do just in time packaging to then convert that content into all the different formats we might need. It will save the cost of storage, rather than storing five different formats of same video in S3, we can store on format and only convert that and deliver it when we need it. “However, there is trade off between the cost of storage and Media Packaging cost. We have to choose wisely as per our requirements. “

9. S3:  This is our destination bucket which is created by cloud formation template and it will hold all the encoded files.

10. Cloud Front:  We will deliver all the contents from destination S3 bucket through cloud front so users wouldn’t have any access to S3 bucket.


Conclusion:

This is a reference architecture which could be used as baseline to build any other capabilities as per need. For instance, we can use “AWS Interactive Video Service” at front end for live streaming and it could record live streaming and send it to S3 source bucket where we can present that streaming on demand with this solution. In other words, this is very scalable and automated solution which can be modified as per our media related needs.

For more details https://docs.aws.amazon.com/solutions/latest/video-on-demand/welcome.html 

# Script to Upload files to AWS S3 Using Golang

In this article, we will create a simple script to upload files to AWS Simple Storage Service (S3).

Amazon S3 is a storage service that provides object storage through a web service interface. Amazon S3 provides secure, durable, and highly-scalable cloud storage. It is easy to use and allows quick storage and retrieval of any amount of data, from anywhere in the world.

Amazon S3 boasts in its website to provide industry-leading scalability, data availability, security, and performance.

In this tutorial you will learn how to upload files to AWS S3 using the Go AWS SDK .

##Prerequisites

- Access to the AWS Account where the bucket resides
- Text editor that allows you to type the code.
- Go installed locally. I am using Golang version 1.17 in my machine.

```bash
➜ go version
go version go1.17.1 darwin/amd64
```

## Table of contents

- Setting up AWS access and creating the bucket to use
- Creating the directory structure and initializing Golang module
- Adding the code to the script
- Building and testing the script

## 1. Setting up AWS access and creating the bucket to use

Before proceeding, we need to have access to AWS. We need credentials that have

Head over to AWS IaM console and create a programmatic user with access to S3 – Create buckets, put content in it and list bucket contents.

Once you have the AWS Access Key and Secret Key, Use this command to update the default profile with the credentials.

```bash
aws configure
```

The alternative is to create a separate profile in the file `~/.aws/config`. This will create a profile `citizix` setting the default region to `eu-west-1` while supplying the credentials:

```
[default]
region=us-west-2
output=json

[profile citizix]
aws_region=eu-west-1
aws_default_region=eu-west-1
aws_access_key_id=xxxxxxx
aws_secret_access_key=`xxxxxx
```

In terminal use this command to activate the profile:

```
export AWS_PROFILE=citizix
```

Confirm that access is working as expected by using this command:

```
aws s3 ls
```

### Creating a bucket

Now that our credentials are working as expected, let’s create a bucket. This command creates a bucket citizix-data-files in the region eu-west-1:

```
aws s3api create-bucket \
    --bucket citizix-data-files \
    --region eu-west-1 \
    --create-bucket-configuration LocationConstraint=eu-west-1
```

I got this response after running that command:

```
{
    "Location": "http://citizix-data-files.s3.amazonaws.com/"
}
```

## 2. Creating the directory structure and initializing golang module

With the bucket created, let us set up our local directory. Create a directory with this command:

```
mkdir upload-to-s3
```

Switch into the directory:

```
cd upload-to-s3
```
Now initialize a Golang module called `upload-to-s3` in the directory:

```
➜ go mod init upload-to-s3
go: creating new go.mod: module upload-to-s3
```

Once this is done, a file called go.mod will be created with this content defining the module and specifying the go version to use:

```
module upload-to-s3

go 1.17
```

Next create a file called main.go in the same place. This is where our code resides. This is my directory structure:

```
➜ tree upload-to-s3
upload-to-s3
├── go.mod
└── main.go

0 directories, 2 files
```

## 3. Adding the code to the script

With the S3 bucket in place and the directory structure created, let’s now add the code to execute the logic we want.

We want to have the script accept command line arguments for the AWS region, Destination bucket and the file to upload. This will make it flexible enough to be re-used.

Let’s add the necessary imports:

```go
package main

import (
    "bytes"
    "flag"
    "fmt"
    "log"
    "net/http"
    "os"
    "path/filepath"

    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
)
```

Next, let’s create the main function where we will initialize the command line arguments, initialize the AWS Session and S3 Session then call the uploadFileToS3 function to handle the logic of uploading to s3:

```go
func main() {
    // Create variables to be used by flag assignments
    var awsRegion, bucketName, filePath string

    // Initialize flags to be passed to the script
    flag.StringVar(&awsRegion, "r", "", "AWS Region")
    flag.StringVar(&bucketName, "b", "", "AWS S3 Bucket to upload to")
    flag.StringVar(&filePath, "f", "", "Path to the file to upload")
    flag.Parse()

    // Basic validation to ensure the flags values are not empty
    if awsRegion == "" || bucketName == "" || filePath == "" {
        log.Printf("Required arguments have not been provided.")
        flag.PrintDefaults()
        os.Exit(1)
    }

    // Create an AWS Session. This will use credentials defined in the environment
    session, err := session.NewSession(&aws.Config{Region: aws.String(awsRegion)})
    if err != nil {
        log.Fatalf("could not initialize new aws session: %v", err)
    }

    // Initialize an s3 client from the session created
    s3Client := s3.New(session)

    // Call the upload to s3 file
    err = uploadFileToS3(s3Client, bucketName, filePath)
    if err != nil {
        log.Fatalf("could not upload file: %v", err)
    }
}
```

Let us now create the uploadFileToS3 functionL:

```go
// Uploads a file to AWS S3 given an S3 session client, a bucket name and a file path
func uploadFileToS3(
    s3Client *s3.S3,
    bucketName string,
    filePath string,
) error {

    // Get the fileName from Path
    fileName := filepath.Base(filePath)

    // Open the file from the file path
    upFile, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("could not open local filepath [%v]: %+v", filePath, err)
    }
    defer upFile.Close()

    // Get the file info
    upFileInfo, _ := upFile.Stat()
    var fileSize int64 = upFileInfo.Size()
    fileBuffer := make([]byte, fileSize)
    upFile.Read(fileBuffer)

    // Put the file object to s3 with the file name
    _, err = s3Client.PutObject(&s3.PutObjectInput{
        Bucket:               aws.String(bucketName),
        Key:                  aws.String(fileName),
        ACL:                  aws.String("private"),
        Body:                 bytes.NewReader(fileBuffer),
        ContentLength:        aws.Int64(fileSize),
        ContentType:          aws.String(http.DetectContentType(fileBuffer)),
        ContentDisposition:   aws.String("attachment"),
        ServerSideEncryption: aws.String("AES256"),
    })
    return err
}
```

### Full code

Here is the full code for the script:

```go
package main

import (
    "bytes"
    "flag"
    "fmt"
    "log"
    "net/http"
    "os"
    "path/filepath"

    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
)

func main() {
    // Create variables to be used by flag assignments
    var awsRegion, bucketName, filePath string

    // Initialize flags to be passed to the script
    flag.StringVar(&awsRegion, "r", "", "AWS Region")
    flag.StringVar(&bucketName, "b", "", "AWS S3 Bucket to upload to")
    flag.StringVar(&filePath, "f", "", "Path to the file to upload")
    flag.Parse()

    // Basic validation to ensure the flags values are not empty
    if awsRegion == "" || bucketName == "" || filePath == "" {
        log.Printf("Required arguments have not been provided.")
        flag.PrintDefaults()
        os.Exit(1)
    }

    // Create an AWS Session. This will use credentials defined in the environment
    session, err := session.NewSession(&aws.Config{Region: aws.String(awsRegion)})
    if err != nil {
        log.Fatalf("could not initialize new aws session: %v", err)
    }

    // Initialize an s3 client from the session created
    s3Client := s3.New(session)

    // Call the upload to s3 file
    err = uploadFileToS3(s3Client, bucketName, filePath)
    if err != nil {
        log.Fatalf("could not upload file: %v", err)
    }
}

// Uploads a file to AWS S3 given an S3 session client, a bucket name and a file path
func uploadFileToS3(
    s3Client *s3.S3,
    bucketName string,
    filePath string,
) error {

    // Get the fileName from Path
    fileName := filepath.Base(filePath)

    // Open the file from the file path
    upFile, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("could not open local filepath [%v]: %+v", filePath, err)
    }
    defer upFile.Close()

    // Get the file info
    upFileInfo, _ := upFile.Stat()
    var fileSize int64 = upFileInfo.Size()
    fileBuffer := make([]byte, fileSize)
    upFile.Read(fileBuffer)

    // Put the file object to s3 with the file name
    _, err = s3Client.PutObject(&s3.PutObjectInput{
        Bucket:               aws.String(bucketName),
        Key:                  aws.String(fileName),
        ACL:                  aws.String("private"),
        Body:                 bytes.NewReader(fileBuffer),
        ContentLength:        aws.Int64(fileSize),
        ContentType:          aws.String(http.DetectContentType(fileBuffer)),
        ContentDisposition:   aws.String("attachment"),
        ServerSideEncryption: aws.String("AES256"),
    })
    return err
}
```

## 4. Building and testing the script

After the script has been added to `main.go`, we need to build it to an executable that we can use.

First ensure that the Golang module dependencies have been downloaded. Use this command:

```
➜ go mod tidy
go: finding module for package github.com/aws/aws-sdk-go/service/s3
go: finding module for package github.com/aws/aws-sdk-go/aws
go: finding module for package github.com/aws/aws-sdk-go/aws/session
go: found github.com/aws/aws-sdk-go/aws in github.com/aws/aws-sdk-go v1.40.55
go: found github.com/aws/aws-sdk-go/aws/session in github.com/aws/aws-sdk-go v1.40.55
go: found github.com/aws/aws-sdk-go/service/s3 in github.com/aws/aws-sdk-go v1.40.55
```

Let’s build the script to binary `upload-to-s3`:

```
go build -o upload-to-s3
```

Then test the file with this command. This will upload the file in `~/Desktop/data.csv` to the created bucket. If nothing is printed out then the file was uploaded successfully.

```
./upload-to-s3 -r eu-west-1 -b citizix-data-files -f ~/Desktop/data.csv
```

Confirm that the file was added using the `aws s3api list-objects` command:

```
aws s3api list-objects --bucket citizix-data-files --query 'Contents[].{Key: Key, Size: Size}'
```

Output:

```
[
    {
        "Key": "data.csv",
        "Size": 24
    }
]
```

From the above output, the file was successfully uploaded to s3.

## Conclusion

Congratulations!

Up to this point, we have managed to create a simple Golang script that we can use to upload any file to any s3 bucket as long as we are authenticated.


# Access Logs for Your Application Load Balancer<a name="load-balancer-access-logs"></a>

Elastic Load Balancing provides access logs that capture detailed information about requests sent to your load balancer\. Each log contains information such as the time the request was received, the client's IP address, latencies, request paths, and server responses\. You can use these access logs to analyze traffic patterns and troubleshoot issues\.

Access logging is an optional feature of Elastic Load Balancing that is disabled by default\. After you enable access logging for your load balancer, Elastic Load Balancing captures the logs and stores them in the Amazon S3 bucket that you specify as compressed files\. You can disable access logging at any time\.

Each access log file is automatically encrypted before it is stored in your S3 bucket and decrypted when you access it\. You do not need to take any action; the encryption and decryption is performed transparently\. Each log file is encrypted with a unique key, which is itself encrypted with a master key that is regularly rotated\. For more information, see [Protecting Data Using Server\-Side Encryption with Amazon S3\-Managed Encryption Keys \(SSE\-S3\)](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html) in the *Amazon Simple Storage Service Developer Guide*\.

There is no additional charge for access logs\. You are charged storage costs for Amazon S3, but not charged for the bandwidth used by Elastic Load Balancing to send log files to Amazon S3\. For more information about storage costs, see [Amazon S3 Pricing](https://aws.amazon.com/s3/pricing/)\.

**Topics**
+ [Access Log Files](#access-log-file-format)
+ [Access Log Entries](#access-log-entry-format)
+ [Bucket Permissions](#access-logging-bucket-permissions)
+ [Enable Access Logging](#enable-access-logging)
+ [Disable Access Logging](#disable-access-logging)
+ [Processing Access Log Files](#log-processing-tools)

## Access Log Files<a name="access-log-file-format"></a>

Elastic Load Balancing publishes a log file for each load balancer node every 5 minutes\. Log delivery is eventually consistent\. The load balancer can deliver multiple logs for the same period\. This usually happens if the site has high traffic\.

The file names of the access logs use the following format:

```
bucket[/prefix]/AWSLogs/aws-account-id/elasticloadbalancing/region/yyyy/mm/dd/aws-account-id_elasticloadbalancing_region_load-balancer-id_end-time_ip-address_random-string.log.gz
```

*bucket*  
The name of the S3 bucket\.

*prefix*  
The prefix \(logical hierarchy\) in the bucket\. If you don't specify a prefix, the logs are placed at the root level of the bucket\.

*aws\-account\-id*  
The AWS account ID of the owner\.

*region*  
The region for your load balancer and S3 bucket\.

*yyyy*/*mm*/*dd*  
The date that the log was delivered\.

*load\-balancer\-id*  
The resource ID of the load balancer\. If the resource ID contains any forward slashes \(/\), they are replaced with periods \(\.\)\.

*end\-time*  
The date and time that the logging interval ended\. For example, an end time of 20140215T2340Z contains entries for requests made between 23:35 and 23:40\.

*ip\-address*  
The IP address of the load balancer node that handled the request\. For an internal load balancer, this is a private IP address\.

*random\-string*  
A system\-generated random string\.

The following is an example log file name:

```
s3://my-bucket/prefix/AWSLogs/123456789012/elasticloadbalancing/us-east-2/2016/05/01/123456789012_elasticloadbalancing_us-east-2_my-loadbalancer_20140215T2340Z_172.160.001.192_20sg8hgm.log.gz
```

You can store your log files in your bucket for as long as you want, but you can also define Amazon S3 lifecycle rules to archive or delete log files automatically\. For more information, see [Object Lifecycle Management](https://docs.aws.amazon.com/AmazonS3/latest/dev/object-lifecycle-mgmt.html) in the *Amazon Simple Storage Service Developer Guide*\.

## Access Log Entries<a name="access-log-entry-format"></a>

Elastic Load Balancing logs requests sent to the load balancer, including requests that never made it to the targets\. For example, if a client sends a malformed request, or there are no healthy targets to respond to the request, the request is still logged\. Note that Elastic Load Balancing does not log health check requests\.

Each log entry contains the details of a single request \(or connection in the case of WebSockets\) made to the load balancer\. For WebSockets, an entry is written only after the connection is closed\. If the upgraded connection can't be established, the entry is the same as for an HTTP or HTTPS request\.

**Important**  
Elastic Load Balancing logs requests on a best\-effort basis\. We recommend that you use access logs to understand the nature of the requests, not as a complete accounting of all requests\.

**Topics**
+ [Syntax](#access-log-entry-syntax)
+ [Actions Taken](#actions-taken)
+ [Error Reason Codes](#error-reason-codes)
+ [Examples](#access-log-entry-examples)

### Syntax<a name="access-log-entry-syntax"></a>

The following table describes the fields of an access log entry, in order\. All fields are delimited by spaces\. When new fields are introduced, they are added to the end of the log entry\. You should ignore any fields at the end of the log entry that you were not expecting\.


| Field | Description | 
| --- | --- | 
| type | The type of request or connection\. The possible values are as follows \(ignore any other values\): [\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html)  | 
| timestamp | The time when the load balancer generated a response to the client, in ISO 8601 format\. For WebSockets, this is the time when the connection is closed\. | 
| elb | The resource ID of the load balancer\. If you are parsing access log entries, note that resources IDs can contain forward slashes \(/\)\. | 
| client:port | The IP address and port of the requesting client\. | 
| target:port |  The IP address and port of the target that processed this request\. If the client didn't send a full request, the load balancer can't dispatch the request to a target, and this value is set to \-\. If the target is a Lambda function, this value is set to \-\. If the request is blocked by AWS WAF, this value is set to \- and the value of elb\_status\_code is set to 403\.  | 
| request\_processing\_time |  The total time elapsed \(in seconds, with millisecond precision\) from the time the load balancer received the request until the time it sent it to a target\. This value is set to \-1 if the load balancer can't dispatch the request to a target\. This can happen if the target closes the connection before the idle timeout or if the client sends a malformed request\. This value can also be set to \-1 if the registered target does not respond before the idle timeout\.  | 
| target\_processing\_time |  The total time elapsed \(in seconds, with millisecond precision\) from the time the load balancer sent the request to a target until the target started to send the response headers\. This value is set to \-1 if the load balancer can't dispatch the request to a target\. This can happen if the target closes the connection before the idle timeout or if the client sends a malformed request\. This value can also be set to \-1 if the registered target does not respond before the idle timeout\.  | 
| response\_processing\_time |  The total time elapsed \(in seconds, with millisecond precision\) from the time the load balancer received the response header from the target until it started to send the response to the client\. This includes both the queuing time at the load balancer and the connection acquisition time from the load balancer to the client\. This value is set to \-1 if the load balancer can't send the request to a target\. This can happen if the target closes the connection before the idle timeout or if the client sends a malformed request\.  | 
| elb\_status\_code | The status code of the response from the load balancer\. | 
| target\_status\_code | The status code of the response from the target\. This value is recorded only if a connection was established to the target and the target sent a response\. Otherwise, it is set to \-\. | 
| received\_bytes |  The size of the request, in bytes, received from the client \(requester\)\. For HTTP requests, this includes the headers\. For WebSockets, this is the total number of bytes received from the client on the connection\.  | 
| sent\_bytes |  The size of the response, in bytes, sent to the client \(requester\)\. For HTTP requests, this includes the headers\. For WebSockets, this is the total number of bytes sent to the client on the connection\.  | 
| "request" |  The request line from the client, enclosed in double quotes and logged using the following format: HTTP method \+ protocol://host:port/uri \+ HTTP version\. The load balancer preserves the URL sent by the client, as is, when recording the request URI\. It does not set the content type for the access log file\. When you process this field, consider how the client sent the URL\.  | 
| "user\_agent" |  A User\-Agent string that identifies the client that originated the request, enclosed in double quotes\. The string consists of one or more product identifiers, product\[/version\]\. If the string is longer than 8 KB, it is truncated\.  | 
| ssl\_cipher |  \[HTTPS listener\] The SSL cipher\. This value is set to \- if the listener is not an HTTPS listener\.  | 
| ssl\_protocol |  \[HTTPS listener\] The SSL protocol\. This value is set to \- if the listener is not an HTTPS listener\.  | 
| target\_group\_arn |  The Amazon Resource Name \(ARN\) of the target group\.  | 
| "trace\_id" |  The contents of the **X\-Amzn\-Trace\-Id** header, enclosed in double quotes\.  | 
| "domain\_name" |  \[HTTPS listener\] The SNI domain provided by the client during the TLS handshake, enclosed in double quotes\. This value is set to \- if the client doesn't support SNI or the domain doesn't match a certificate and the default certificate is presented to the client\.  | 
| "chosen\_cert\_arn" |  \[HTTPS listener\] The ARN of the certificate presented to the client, enclosed in double quotes\. This value is set to `session-reused` if the session is reused\. This value is set to \- if the listener is not an HTTPS listener\.  | 
| matched\_rule\_priority |  The priority value of the rule that matched the request\. If a rule matched, this is a value from 1 to 50,000\. If no rule matched and the default action was taken, this value is set to 0\. If an error occurs during rules evaluation, it is set to \-1\. For any other error, it is set to \-\.  | 
| request\_creation\_time |  The time when the load balancer received the request from the client, in ISO 8601 format\.  | 
| "actions\_executed" |  The actions taken when processing the request, enclosed in double quotes\. This value is a comma\-separated list that can include the values described in [Actions Taken](#actions-taken)\. If no action was taken, such as for a malformed request, this value is set to \-\.  | 
| "redirect\_url" |  The URL of the redirect target for the location header of the HTTP response, enclosed in double quotes\. If no redirect actions were taken, this value is set to \-\.  | 
| "error\_reason" |  The error reason code, enclosed in double quotes\. If the request failed, this is one of the error codes described in [Error Reason Codes](#error-reason-codes)\. If the actions taken do not include an authenticate action or the target is not a Lambda function, this value is set to \-\.  | 

### Actions Taken<a name="actions-taken"></a>

The load balancer stores the actions that it takes in the actions\_executed field of the access log\.
+ `authenticate` — The load balancer validated the session, authenticated the user, and added the user information to the request headers, as specified by the rule configuration\.
+ `fixed-response` — The load balancer issued a fixed response, as specified by the rule configuration\.
+ `forward` — The load balancer forwarded the request to a target, as specified by the rule configuration\.
+ `redirect` — The load balancer redirected the request to another URL, as specified by the rule configuration\.
+ `waf` — The load balancer forwarded the request to AWS WAF to determine whether the request should be forwarded to the target\. If this is the final action, AWS WAF determined that the request should be rejected\.
+ `waf-failed` — The load balancer attempted to forward the request to AWS WAF, but this process failed\.

### Error Reason Codes<a name="error-reason-codes"></a>

If the load balancer cannot complete an authenticate action, the load balancer stores one of the following reason codes in the error\_reason field of the access log\. The load balancer also increments the corresponding CloudWatch metric\. For more information, see [Authenticate Users Using an Application Load Balancer](listener-authenticate-users.md)\.


| Code | Description | Metric | 
| --- | --- | --- | 
| `AuthInvalidCookie` | The authentication cookie is not valid\. | `ELBAuthFailure` | 
| `AuthInvalidGrantError` | The authorization grant code from the token endpoint is not valid\. | `ELBAuthFailure` | 
| `AuthInvalidIdToken` | The ID token is not valid\. | `ELBAuthFailure` | 
| `AuthInvalidStateParam` | The state parameter is not valid\. | `ELBAuthFailure` | 
| `AuthInvalidTokenResponse` | The response from the token endpoint is not valid\. | `ELBAuthFailure` | 
| `AuthInvalidUserinfoResponse` | The response from the user info endpoint is not valid\. | `ELBAuthFailure` | 
| `AuthMissingCodeParam` | The authentication response from the authorization endpoint is missing a query parameter named 'code'\. | `ELBAuthFailure` | 
| `AuthMissingHostHeader` | The authentication response from the authorization endpoint is missing a host header field\. | `ELBAuthError` | 
| `AuthMissingStateParam` | The authentication response from the authorization endpoint is missing a query parameter named 'state'\. | `ELBAuthFailure` | 
| `AuthTokenEpRequestFailed` | There is an error response \(non\-2XX\) from the token endpoint\. | `ELBAuthError` | 
| `AuthTokenEpRequestTimeout` | The load balancer is unable to communicate with the token endpoint\. | `ELBAuthError` | 
| `AuthUnhandledException` | The load balancer encountered an unhandled exception\. | `ELBAuthError` | 
| `AuthUserinfoEpRequestFailed` | There is an error response \(non\-2XX\) from the IdP user info endpoint\. | `ELBAuthError` | 
| `AuthUserinfoEpRequestTimeout` | The load balancer is unable to communicate with the IdP user info endpoint\. | `ELBAuthError` | 
| `AuthUserinfoResponseSizeExceeded` | The size of the claims returned by the IdP exceeded 11K bytes\. | `ELBAuthUserClaimsSizeExceeded` | 

If a request to a Lambda function fails, the load balancer stores one of the following reason codes in the error\_reason field of the access log\. The load balancer also increments the corresponding CloudWatch metric\. For more information, see the Lambda [Invoke](https://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html) action\.


| Code | Description | Metric | 
| --- | --- | --- | 
| `LambdaAccessDenied` | The load balancer did not have permission to invoke the Lambda function\. | `LambdaUserError` | 
| `LambdaConnectionTimeout` | An attempt to connect to Lambda timed out\. | `LambdaInternalError` | 
| `LambdaEC2AccessDeniedException` | Amazon EC2 denied access to Lambda during function initialization\. | `LambdaUserError` | 
| `LambdaEC2ThrottledException` | Amazon EC2 throttled Lambda during function initialization\. | `LambdaUserError` | 
| `LambdaEC2UnexpectedException` | Amazon EC2 encountered an unexpected exception during function initialization\. | `LambdaUserError` | 
| `LambdaENILimitReachedException` | Lambda couldn't create a network interface in the VPC specified in the configuration of the Lambda function because the limit for network interfaces was exceeded\. | `LambdaUserError` | 
| `LambdaInvalidResponse` | The response from the Lambda function is malformed or is missing required fields\. | `LambdaUserError` | 
| `LambdaInvalidRuntimeException` | The specified version of the Lambda runtime is not supported\. | `LambdaUserError` | 
| `LambdaInvalidSecurityGroupIDException` | The security group ID specified in the configuration of the Lambda function is not valid\. | `LambdaUserError` | 
| `LambdaInvalidSubnetIDException` | The subnet ID specified in the configuration of the Lambda function is not valid\. | `LambdaUserError` | 
| `LambdaInvalidZipFileException` | Lambda could not unzip the specified function zip file\. | `LambdaUserError` | 
| `LambdaKMSAccessDeniedException` | Lambda could not decrypt environment variables because access to the KMS key was denied\. Check the KMS permissions of the Lambda function\. | `LambdaUserError` | 
| `LambdaKMSDisabledException` | Lambda could not decrypt environment variables because the specified KMS key is disabled\. Check the KMS key settings of the Lambda function\. | `LambdaUserError` | 
| `LambdaKMSInvalidStateException` | Lambda could not decrypt environment variables because the state of the KMS key is not valid\. Check the KMS key settings of the Lambda function\. | `LambdaUserError` | 
| `LambdaKMSNotFoundException` | Lambda could not decrypt environment variables because the KMS key was not found\. Check the KMS key settings of the Lambda function\. | `LambdaUserError` | 
| `LambdaRequestTooLarge` | The size of the request body exceeded 1 MB\. | `LambdaUserError` | 
| `LambdaResourceNotFound` | The Lambda function could not be found\. | `LambdaUserError` | 
| `LambdaResponseTooLarge` | The size of the response exceeded 1 MB\. | `LambdaUserError` | 
| `LambdaServiceException` | Lambda encountered an internal error\. | `LambdaInternalError` | 
| `LambdaSubnetIPAddressLimitReachedException` | Lambda could not set up VPC access for the Lambda function because one or more subnets have no available IP addresses\. | `LambdaUserError` | 
| `LambdaThrottling` | The Lambda function was throttled because there were too many requests\. | `LambdaUserError` | 
| `LambdaUnhandled` | The Lambda function encountered an unhandled exception\. | `LambdaUserError` | 
| `LambdaUnhandledException` | The load balancer encountered an unhandled exception\. | `LambdaInternalError` | 

### Examples<a name="access-log-entry-examples"></a>

The following are example log entries\. Note that the text appears on multiple lines only to make them easier to read\.

**Example HTTP Entry**  
The following is an example log entry for an HTTP listener \(port 80 to port 80\):

```
http 2018-07-02T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188 
192.168.131.39:2817 10.0.0.1:80 0.000 0.001 0.000 200 200 34 366 
"GET http://www.example.com:80/ HTTP/1.1" "curl/7.46.0" - - 
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337262-36d228ad5d99923122bbe354" "-" "-" 
0 2018-07-02T22:22:48.364000Z "forward" "-" "-"
```

**Example HTTPS Entry**  
The following is an example log entry for an HTTPS listener \(port 443 to port 80\):

```
https 2018-07-02T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188 
192.168.131.39:2817 10.0.0.1:80 0.086 0.048 0.037 200 200 0 57 
"GET https://www.example.com:443/ HTTP/1.1" "curl/7.46.0" ECDHE-RSA-AES128-GCM-SHA256 TLSv1.2 
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337281-1d84f3d73c47ec4e58577259" "www.example.com" "arn:aws:acm:us-east-2:123456789012:certificate/12345678-1234-1234-1234-123456789012"
1 2018-07-02T22:22:48.364000Z "authenticate,forward" "-" "-"
```

**Example HTTP/2 Entry**  
The following is an example log entry for an HTTP/2 stream\.

```
h2 2018-07-02T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188 
10.0.1.252:48160 10.0.0.66:9000 0.000 0.002 0.000 200 200 5 257 
"GET https://10.0.2.105:773/ HTTP/2.0" "curl/7.46.0" ECDHE-RSA-AES128-GCM-SHA256 TLSv1.2
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337327-72bd00b0343d75b906739c42" "-" "-"
1 2018-07-02T22:22:48.364000Z "redirect" "https://example.com:80/" "-"
```

**Example WebSockets Entry**  
The following is an example log entry for a WebSockets connection\.

```
ws 2018-07-02T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188 
10.0.0.140:40914 10.0.1.192:8010 0.001 0.003 0.000 101 101 218 587 
"GET http://10.0.0.30:80/ HTTP/1.1" "-" - - 
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337364-23a8c76965a2ef7629b185e3" "-" "-"
1 2018-07-02T22:22:48.364000Z "forward" "-" "-"
```

**Example Secured WebSockets Entry**  
The following is an example log entry for a secured WebSockets connection\.

```
wss 2018-07-02T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188 
10.0.0.140:44244 10.0.0.171:8010 0.000 0.001 0.000 101 101 218 786
"GET https://10.0.0.30:443/ HTTP/1.1" "-" ECDHE-RSA-AES128-GCM-SHA256 TLSv1.2 
arn:aws:elasticloadbalancing:us-west-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337364-23a8c76965a2ef7629b185e3" "-" "-"
1 2018-07-02T22:22:48.364000Z "forward" "-" "-"
```

**Example Entries for Lambda Functions**  
The following is an example log entry for a request to a Lambda function that succeeded:

```
http 2018-11-30T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188
192.168.131.39:2817 - 0.000 0.001 0.000 200 200 34 366
"GET http://www.example.com:80/ HTTP/1.1" "curl/7.46.0" - -
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337364-23a8c76965a2ef7629b185e3" "-" "-"
0 2018-11-30T22:22:48.364000Z "forward" "-" "-"
```

The following is an example log entry for a request to a Lambda function that failed:

```
http 2018-11-30T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188
192.168.131.39:2817 - 0.000 0.001 0.000 502 - 34 366
"GET http://www.example.com:80/ HTTP/1.1" "curl/7.46.0" - -
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337364-23a8c76965a2ef7629b185e3" "-" "-"
0 2018-11-30T22:22:48.364000Z "forward" "-" "LambdaInvalidResponse"
```

## Bucket Permissions<a name="access-logging-bucket-permissions"></a>

When you enable access logging, you must specify an S3 bucket for the access logs\. The bucket must meet the following requirements\.

**Requirements**
+ The bucket must be located in the same region as the load balancer\.
+ The bucket must have a bucket policy that grants Elastic Load Balancing permission to write the access logs to your bucket\. Bucket policies are a collection of JSON statements written in the access policy language to define access permissions for your bucket\. Each statement includes information about a single permission and contains a series of elements\.

Use one of the following options to prepare an S3 bucket for the access logs\.

**Options**
+ If you need to create a bucket and you plan to use the console to enable access logging, you can skip to [Enable Access Logging](#enable-access-logging) and select the option to have the console create the bucket and bucket policy for you\.
+ If you need to create a bucket for your access logs and you are using the AWS CLI or an API, use the following procedure to create the bucket and add the required bucket policy manually\.
+ If you already have a bucket for your access logs, open the Amazon S3 console per step 1 of the following procedure and then skip to step 4 to add or update the bucket policy\.

**To create an Amazon S3 bucket with the required permissions**

1. Open the Amazon S3 console at [https://console\.aws\.amazon\.com/s3/](https://console.aws.amazon.com/s3/)\.

1. \[Skip to use existing bucket\] Choose **Create Bucket**\.

1. \[Skip to use existing bucket\] In the **Create a Bucket** dialog box, do the following:

   1. For **Bucket Name**, enter a name for your bucket \(for example, `my-loadbalancer-logs`\)\. This name must be unique across all existing bucket names in Amazon S3\. In some regions, there might be additional restrictions on bucket names\. For more information, see [Bucket Restrictions and Limitations](https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html) in the *Amazon Simple Storage Service Developer Guide*\.

   1. For **Region**, select the region where you created your load balancer\.

   1. Choose **Create**\.

1. Select the bucket and choose **Permissions**\.

1. Choose **Bucket Policy**\. If your bucket already has an attached policy, you can add the required statement to the existing policy\.

1. Choose **Policy generator**\. On the **AWS Policy Generator** page, do the following:

   1. For **Select Type of Policy**, choose **S3 Bucket Policy**\.

   1. For **Effect**, choose **Allow**\.

   1. For **Principal**, specify one of the following AWS account IDs to grant Elastic Load Balancing access to the S3 bucket\. Use the account ID that corresponds to the region for your load balancer and bucket\.    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html)

      \* These regions requires a separate account\. For more information, see [AWS GovCloud \(US\-West\)](https://aws.amazon.com/govcloud-us/) and [China \(Beijing\)](http://www.amazonaws.cn/en/)\.

   1. For **Actions**, choose `PutObject` to allow Elastic Load Balancing to store objects in the S3 bucket\.

   1. For **Amazon Resource Name \(ARN\)**, type the ARN of your S3 bucket in the following format\. For *aws\-account\-id*, specify the ID of the AWS account that owns the load balancer \(for example, *123456789012*\)\. Do not specify a wildcard for the account ID, as this would allow any other account to write access logs to your bucket\. To use a single bucket to store access logs from load balancers in multiple accounts, specify one ARN per account in the bucket policy, using the corresponding AWS account ID in each ARN\.

      ```
      arn:aws:s3:::bucket/prefix/AWSLogs/aws-account-id/*
      ```

      Note that if you are using the `us-gov-west-1` region, specify `arn:aws-us-gov` instead of `arn:aws` in the ARN\.

   1. Choose **Add Statement**, **Generate Policy**\. The policy document should be similar to the following:

      ```
      {
        "Id": "Policy1429136655940",
        "Version": "2012-10-17",
        "Statement": [
          {
            "Sid": "Stmt1429136633762",
            "Action": [
              "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::my-loadbalancer-logs/my-app/AWSLogs/123456789012/*",
            "Principal": {
              "AWS": [
                "797873946194"
              ]
            }
          }
        ]
      }
      ```

   1. If you are creating a new bucket policy, copy the entire policy document, and then choose **Close**\.

      If you are editing an existing bucket policy, copy the new statement from the policy document \(the text between the \[ and \] of the `Statement` element\), and then choose **Close**\.

1. Go back to the Amazon S3 console and paste the policy into the text area as appropriate\.

1. Choose **Save**\.

## Enable Access Logging<a name="enable-access-logging"></a>

When you enable access logging for your load balancer, you must specify the name of the S3 bucket where the load balancer will store the logs\. The bucket must be in the same region as your load balancer, and must have a bucket policy that grants Elastic Load Balancing permission to write the access logs to the bucket\. The bucket can be owned by a different account than the account that owns the load balancer\.

**To enable access logging using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. In the navigation pane, choose **Load Balancers**\.

1. Select your load balancer\.

1. On the **Description** tab, choose **Edit attributes**\.

1. On the **Edit load balancer attributes** page, do the following:

   1. Choose **Enable access logs**\.

   1. For **S3 location**, type the name of your S3 bucket, including any prefix \(for example, `my-loadbalancer-logs/my-app`\)\. You can specify the name of an existing bucket or a name for a new bucket\. If you specify an existing bucket, be sure that you own this bucket and that you configured the required bucket policy\.

   1. \(Optional\) If the bucket does not exist, choose **Create this location for me**\. You must specify a name that is unique across all existing bucket names in Amazon S3 and follows the DNS naming conventions\. For more information, see [Rules for Bucket Naming](https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html#bucketnamingrules) in the *Amazon Simple Storage Service Developer Guide*\.

   1. Choose **Save**\.

**To enable access logging using the AWS CLI**  
Use the [modify\-load\-balancer\-attributes](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-load-balancer-attributes.html) command\.

**To verify that Elastic Load Balancing created a test file in your S3 bucket**

After access logging is enabled for your load balancer, Elastic Load Balancing validates the S3 bucket and creates a test file to ensure that the bucket policy specifies the required permissions\. You can use the Amazon S3 console to verify that the test file was created\. Note that the test file is not an actual access log file; it doesn't contain example records\.

1. Open the Amazon S3 console at [https://console\.aws\.amazon\.com/s3/](https://console.aws.amazon.com/s3/)\.

1. For **All Buckets**, select your S3 bucket\.

1. Navigate to the test log file\. The path should be as follows:

   ```
   my-bucket/prefix/AWSLogs/123456789012/ELBAccessLogTestFile
   ```

**To manage the S3 bucket for your access logs**  
After you enable access logging, be sure to disable access logging before you delete the bucket with your access logs\. Otherwise, if there is a new bucket with the same name and the required bucket policy created in an AWS account that you don't own, Elastic Load Balancing could write the access logs for your load balancer to this new bucket\.

## Disable Access Logging<a name="disable-access-logging"></a>

You can disable access logging for your load balancer at any time\. After you disable access logging, your access logs remain in your S3 bucket until you delete the them\. For more information, see [Working with Buckets](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/BucketOperations.html) in the *Amazon Simple Storage Service Console User Guide*\.

**To disable access logging using the console**

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/)\.

1. In the navigation pane, choose **Load Balancers**\.

1. Select your load balancer\.

1. On the **Description** tab, choose **Edit attributes**\.

1. On the **Edit load balancer attributes** page, clear **Enable access logs**\.

1. Choose **Save**\.

**To disable access logging using the AWS CLI**  
Use the [modify\-load\-balancer\-attributes](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify-load-balancer-attributes.html) command\.

## Processing Access Log Files<a name="log-processing-tools"></a>

The access log files are compressed\. If you open the files using the Amazon S3 console, they are uncompressed and the information is displayed\. If you download the files, you must uncompress them to view the information\.

If there is a lot of demand on your website, your load balancer can generate log files with gigabytes of data\. You might not be able to process such a large amount of data using line\-by\-line processing\. Therefore, you might have to use analytical tools that provide parallel processing solutions\. For example, you can use the following analytical tools to analyze and process access logs:
+ Amazon Athena is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL\. For more information, see [Querying Application Load Balancer Logs](https://docs.aws.amazon.com/athena/latest/ug/application-load-balancer-logs.html) in the *Amazon Athena User Guide*\.
+ [Loggly](https://www.loggly.com/docs/s3-ingestion-auto/)
+ [Splunk](https://splunkbase.splunk.com/app/1274/)
+ [Sumo Logic](https://www.sumologic.com/application/elb/)
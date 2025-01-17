# Troubleshoot Your Application Load Balancers<a name="load-balancer-troubleshooting"></a>

The following information can help you troubleshoot issues with your Application Load Balancer\.

**Topics**
+ [A registered target is not in service](#target-not-inservice)
+ [Clients cannot connect to an Internet\-facing load balancer](#client-cannot-connect)
+ [The load balancer sends requests to unhealthy targets](#no-healthy-targets)
+ [The load balancer generates an HTTP error](#load-balancer-http-error-codes)
+ [A target generates an HTTP error](#target-http-errors)

## A registered target is not in service<a name="target-not-inservice"></a>

If a target is taking longer than expected to enter the `InService` state, it might be failing health checks\. Your target is not in service until it passes one health check\. For more information, see [Health Checks for Your Target Groups](target-group-health-checks.md)\.

Verify that your instance is failing health checks and then check for the following:

**A security group does not allow traffic**  
The security group associated with an instance must allow traffic from the load balancer using the health check port and health check protocol\. You can add a rule to the instance security group to allow all traffic from the load balancer security group\. Also, the security group for your load balancer must allow traffic to the instances\.

**A network access control list \(ACL\) does not allow traffic**  
The network ACL associated with the subnets for your instances must allow inbound traffic on the health check port and outbound traffic on the ephemeral ports \(1024\-65535\)\. The network ACL associated with the subnets for your load balancer nodes must allow inbound traffic on the ephemeral ports and outbound traffic on the health check and ephemeral ports\.

**The ping path does not exist**  
Create a target page for the health check and specify its path as the ping path\.

**The connection times out**  
First, verify that you can connect to the target directly from within the network using the private IP address of the target and the health check protocol\. If you can't connect, check whether the instance is over\-utilized, and add more targets to your target group if it is too busy to respond\. If you can connect, it is possible that the target page is not responding before the health check timeout period\. Choose a simpler target page for the health check or adjust the health check settings\.

**The target did not return a successful response code**  
By default, the success code is 200, but you can optionally specify additional success codes when you configure health checks\. Confirm the success codes that the load balancer is expecting and that your application is configured to return these codes on success\.

## Clients cannot connect to an Internet\-facing load balancer<a name="client-cannot-connect"></a>

If the load balancer is not responding to requests, check for the following:

**Your Internet\-facing load balancer is attached to a private subnet**  
Verify that you specified public subnets for your load balancer\. A public subnet has a route to the Internet Gateway for your virtual private cloud \(VPC\)\.

**A security group or network ACL does not allow traffic**  
The security group for the load balancer and any network ACLs for the load balancer subnets must allow inbound traffic from the clients and outbound traffic to the clients on the listener ports\.

## The load balancer sends requests to unhealthy targets<a name="no-healthy-targets"></a>

If there is at least one healthy target in a target group, the load balancer routes requests only to the healthy targets\. If a target group contains only unhealthy targets, the load balancer routes requests to the unhealthy targets\.

## The load balancer generates an HTTP error<a name="load-balancer-http-error-codes"></a>

The following HTTP errors are generated by the load balancer\. The load balancer sends the HTTP code to the client, saves the request to the access log, and increments the `HTTPCode_ELB_4XX_Count` or `HTTPCode_ELB_5XX_Count` metric\.

**Topics**
+ [HTTP 400: Bad Request](#http-400-issues)
+ [HTTP 401: Unauthorized](#http-401-issues)
+ [HTTP 403: Forbidden](#http-403-issues)
+ [HTTP 408: Request Timeout](#http-408-issues)
+ [HTTP 413: Payload Too Large](#http-413-issues)
+ [HTTP 414: URI Too Long](#http-414-issues)
+ [HTTP 460](#http-460-issues)
+ [HTTP 463](#http-463-issues)
+ [HTTP 500: Internal Server Error](#http-500-issues)
+ [HTTP 501: Not Implemented](#http-501-issues)
+ [HTTP 502: Bad Gateway](#http-502-issues)
+ [HTTP 503: Service Unavailable](#http-503-issues)
+ [HTTP 504: Gateway Timeout](#http-504-issues)
+ [HTTP 561: Unauthorized](#http-561-issues)

### HTTP 400: Bad Request<a name="http-400-issues"></a>

Possible causes:
+ The client sent a malformed request that does not meet the HTTP specification\.
+ The client used the HTTP CONNECT method, which is not supported by Application Load Balancers\.
+ The request header exceeded 16K per request line, 16K per single header, or 64K for the entire header\.

### HTTP 401: Unauthorized<a name="http-401-issues"></a>

You configured a listener rule to authenticate users\. Either you configured `OnUnauthenticatedRequest` to deny unauthenticated users or the IdP denied access\.

### HTTP 403: Forbidden<a name="http-403-issues"></a>

You configured an AWS WAF web access control list \(web ACL\) to monitor requests to your Application Load Balancer and it blocked a request\.

### HTTP 408: Request Timeout<a name="http-408-issues"></a>

The client did not send data before the idle timeout period expired\. Sending a TCP keep\-alive does not prevent this timeout\. Send at least 1 byte of data before each idle timeout period elapses\. Increase the length of the idle timeout period as needed\.

### HTTP 413: Payload Too Large<a name="http-413-issues"></a>

The target is a Lambda function and the request body exceeds 1 MB\.

### HTTP 414: URI Too Long<a name="http-414-issues"></a>

The request URL or query string parameters are too large\.

### HTTP 460<a name="http-460-issues"></a>

The load balancer received a request from a client, but the client closed the connection with the load balancer before the idle timeout period elapsed\.

Check whether the client timeout period is greater than the idle timeout period for the load balancer\. Ensure that your target provides a response to the client before the client timeout period elapses, or increase the client timeout period to match the load balancer idle timeout, if the client supports this\.

### HTTP 463<a name="http-463-issues"></a>

The load balancer received an **X\-Forwarded\-For** request header with more than 30 IP addresses\.

### HTTP 500: Internal Server Error<a name="http-500-issues"></a>

Possible causes:
+ You configured an AWS WAF web access control list \(web ACL\) and there was an error executing the web ACL rules\.
+ You configured a listener rule to authenticate users, but one of the following is true:
  + The load balancer is unable to communicate with the IdP token endpoint or the IdP user info endpoint\. Verify that the security groups for your load balancer and the network ACLs for your VPC allow outbound access to these endpoints\. Verify that your VPC has internet access\. If you have an internal\-facing load balancer, use a NAT gateway to enable internet access\.
  + The size of the claims returned by the IdP exceeded the maximum size supported by the load balancer\.
  + A client submitted an HTTP/1\.0 request without a host header, and the load balancer was unable to generate a redirect URL\.
  + A client submitted a request without an HTTP protocol, and the load balancer was unable to generate a redirect URL\.
  + The requested scope doesn't return an ID token\.

### HTTP 501: Not Implemented<a name="http-501-issues"></a>

The load balancer received a **Transfer\-Encoding** header with an unsupported value\. The supported values for **Transfer\-Encoding** are `chunked` and `identity`\. As an alternative, you can use the **Content\-Encoding** header\.

### HTTP 502: Bad Gateway<a name="http-502-issues"></a>

Possible causes:
+ The load balancer received a TCP RST from the target when attempting to establish a connection\.
+ The load balancer received an unexpected response from the target, such as "ICMP Destination unreachable \(Host unreachable\)", when attempting to establish a connection\. Check whether traffic is allowed from the load balancer subnets to the targets on the target port\.
+ The target closed the connection with a TCP RST or a TCP FIN while the load balancer had an outstanding request to the target\. Check whether the keep\-alive duration of the target is shorter than the idle timeout value of the load balancer\.
+ The target response is malformed or contains HTTP headers that are not valid\.
+ The load balancer encountered an SSL handshake error or SSL handshake timeout \(10 seconds\) when connecting to a target\.
+ The deregistration delay period elapsed for a request being handled by a target that was deregistered\. Increase the delay period so that lengthy operations can complete\.
+ The target is a Lambda function and the response body exceeds 1 MB\.
+ The target is a Lambda function that did not respond before its configured timeout was reached\.

### HTTP 503: Service Unavailable<a name="http-503-issues"></a>

The target groups for the load balancer have no registered targets\.

### HTTP 504: Gateway Timeout<a name="http-504-issues"></a>

Possible causes:
+ The load balancer failed to establish a connection to the target before the connection timeout expired \(10 seconds\)\.
+ The load balancer established a connection to the target but the target did not respond before the idle timeout period elapsed\.
+ The network ACL for the subnet did not allow traffic from the targets to the load balancer nodes on the ephemeral ports \(1024\-65535\)\.
+ The target returns a content\-length header that is larger than the entity body\. The load balancer timed out waiting for the missing bytes\.
+ The target is a Lambda function that did not respond before its possible maximum configured timeout was reached\.

### HTTP 561: Unauthorized<a name="http-561-issues"></a>

You configured a listener rule to authenticate users, but the IdP returned an error code when authenticating the user\.

## A target generates an HTTP error<a name="target-http-errors"></a>

The load balancer forwards valid HTTP responses from targets to the client, including HTTP errors\. The HTTP errors generated by a target are recorded in the `HTTPCode_Target_4XX_Count` and `HTTPCode_Target_5XX_Count` metrics\.
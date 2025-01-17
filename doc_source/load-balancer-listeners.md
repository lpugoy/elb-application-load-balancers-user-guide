# Listeners for Your Application Load Balancers<a name="load-balancer-listeners"></a>

Before you start using your Application Load Balancer, you must add one or more *listeners*\. A listener is a process that checks for connection requests, using the protocol and port that you configure\. The rules that you define for a listener determine how the load balancer routes requests to the targets in one or more target groups\.

**Topics**
+ [Listener Configuration](#listener-configuration)
+ [Listener Rules](#listener-rules)
+ [Rule Action Types](#rule-action-types)
+ [Rule Condition Types](#rule-condition-types)
+ [Create an HTTP Listener](create-listener.md)
+ [Create an HTTPS Listener](create-https-listener.md)
+ [Update Listener Rules](listener-update-rules.md)
+ [Update an HTTPS Listener](listener-update-certificates.md)
+ [Authenticate Users](listener-authenticate-users.md)
+ [Delete a Listener](delete-listener.md)

## Listener Configuration<a name="listener-configuration"></a>

Listeners support the following protocols and ports:
+ **Protocols**: HTTP, HTTPS
+ **Ports**: 1\-65535

You can use an HTTPS listener to offload the work of encryption and decryption to your load balancer so that your applications can focus on their business logic\. If the listener protocol is HTTPS, you must deploy at least one SSL server certificate on the listener\. For more information, see [Create an HTTPS Listener for Your Application Load Balancer](create-https-listener.md)\.

Application Load Balancers provide native support for WebSockets\. You can use WebSockets with both HTTP and HTTPS listeners\.

Application Load Balancers provide native support for HTTP/2 with HTTPS listeners\. You can send up to 128 requests in parallel using one HTTP/2 connection\. The load balancer converts these to individual HTTP/1\.1 requests and distributes them across the healthy targets in the target group\. Because HTTP/2 uses front\-end connections more efficiently, you might notice fewer connections between clients and the load balancer\. You can't use the server\-push feature of HTTP/2\.

For more information, see [Request Routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html#request-routing) in the *Elastic Load Balancing User Guide*\.

## Listener Rules<a name="listener-rules"></a>

Each listener has a default rule, and you can optionally define additional rules\. Each rule consists of a priority, one or more actions, and one or more conditions\. You can add or edit rules at any time\. For more information, see [Edit a Rule](listener-update-rules.md#edit-rule)\.

### Default Rules<a name="listener-default-rule"></a>

When you create a listener, you define actions for the default rule\. Default rules can't have conditions\. If the conditions for none of a listener's rules are met, then the action for the default rule is performed\.

The following is an example of a default rule as shown in the console:

![\[The default rule for a listener.\]](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/images/default_rule.png)

### Rule Priority<a name="listener-rule-priority"></a>

Each rule has a priority\. Rules are evaluated in priority order, from the lowest value to the highest value\. The default rule is evaluated last\. You can change the priority of a nondefault rule at any time\. You cannot change the priority of the default rule\. For more information, see [Reorder Rules](listener-update-rules.md#update-rule-priority)\.

### Rule Actions<a name="listener-rule-actions"></a>

Each rule action has a type, an order, and the information required to perform the action\. For more information, see [Rule Action Types](#rule-action-types)\.

### Rule Conditions<a name="listener-rule-conditions"></a>

Each rule condition has a type and configuration information\. When the conditions for a rule are met, then its actions are performed\. For more information, see [Rule Condition Types](#rule-condition-types)\.

## Rule Action Types<a name="rule-action-types"></a>

The following are the supported action types for a rule:

`authenticate-cognito`  
\[HTTPS listeners\] Use Amazon Cognito to authenticate users\. For more information, see [Authenticate Users Using an Application Load Balancer](listener-authenticate-users.md)\.

`authenticate-oidc`  
\[HTTPS listeners\] Use an identity provider that is compliant with OpenID Connect \(OIDC\) to authenticate users\.

`fixed-response`  
Return a custom HTTP response\. For more information, see [Fixed\-Response Actions](#fixed-response-actions)\.

`forward`  
Forward requests to the specified target group\.

`redirect`  
Redirect requests from one URL to another\. For more information, see [Redirect Actions](#redirect-actions)\.

The action with the lowest order value is performed first\. Each rule must include exactly one of the following actions: `forward`, `redirect`, or `fixed-response`, and it must be the last action to be performed\.

### Fixed\-Response Actions<a name="fixed-response-actions"></a>

You can use `fixed-response` actions to drop client requests and return a custom HTTP response\. You can use this action to return a 2XX, 4XX, or 5XX response code and an optional message\.

When a `fixed-response` action is taken, the action and the URL of the redirect target are recorded in the access logs\. For more information, see [Access Log Entries](load-balancer-access-logs.md#access-log-entry-format)\. The count of successful `fixed-response` actions is reported in the `HTTP_Fixed_Response_Count` metric\. For more information, see [Application Load Balancer Metrics](load-balancer-cloudwatch-metrics.md#load-balancer-metrics-alb)\.

**Example Example Fixed Response Action for the AWS CLI**  
You can specify an action when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following action sends a fixed response with the specified status code and message body\.  

```
[
  {
      "Type": "fixed-response",
      "FixedResponseConfig": {
          "StatusCode": "200",
          "ContentType": "text/plain",
          "MessageBody": "Hello world"
      }
  }
]
```

### Forward Actions<a name="forward-actions"></a>

You can use `forward` actions route requests to the specified target group\.

**Example Example Forward Action for the AWS CLI**  
You can specify an action when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following action forwards the request to the specified target group\.  

```
[
  {
      "Type": "forward",
      "TargetGroupArn": "arn:aws:elasticloadbalancing:us-west-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a06"
  }
]
```

### Redirect Actions<a name="redirect-actions"></a>

You can use `redirect` actions to redirect client requests from one URL to another\. You can configure redirects as either temporary \(HTTP 302\) or permanent \(HTTP 301\) based on your needs\.

A URI consists of the following components:

```
protocol://hostname:port/path?query
```

You must modify at least one of the following components to avoid a redirect loop: protocol, hostname, port, or path\. Any components that you do not modify retain their original values\.

*protocol*  
The protocol \(HTTP or HTTPS\)\. You can redirect HTTP to HTTP, HTTP to HTTPS, and HTTPS to HTTPS\. You cannot redirect HTTPS to HTTP\.

*hostname*  
The hostname\. A hostname is case\-insensitive, can be up to 128 characters in length, and consists of alpha\-numeric characters, wildcards \(\* and ?\), and hyphens \(\-\)\.

*port*  
The port \(1 to 65535\)\.

*path*  
The absolute path, starting with the leading "/"\. A path is case\-sensitive, can be up to 128 characters in length, and consists of alpha\-numeric characters, wildcards \(\* and ?\), & \(using &amp;\), and the following special characters: \_\-\.$/\~"'@:\+\.

*query*  
The query parameters\.

You can reuse URI components of the original URL in the target URL using the following reserved keywords:
+ `#{protocol}` \- Retains the protocol\. Use in the protocol and query components
+ `#{host}` \- Retains the domain\. Use in the hostname, path, and query components
+ `#{port}` \- Retains the port\. Use in the port, path, and query components
+ `#{path}` \- Retains the path\. Use in the path and query components
+ `#{query}` \- Retains the query parameters\. Use in the query component

When a `redirect` action is taken, the action is recorded in the access logs\. For more information, see [Access Log Entries](load-balancer-access-logs.md#access-log-entry-format)\. The count of successful `redirect` actions is reported in the `HTTP_Redirect_Count` metric\. For more information, see [Application Load Balancer Metrics](load-balancer-cloudwatch-metrics.md#load-balancer-metrics-alb)\.

**Example Example Redirect Actions Using the Console**  
The following rule sets up a permanent redirect to a URL that uses the HTTPS protocol and the specified port \(40443\), but retains the original hostname, path, and query parameters\. This screen is equivalent to "https://\#\{host\}:40443/\#\{path\}?\#\{query\}"\.  

![\[A rule that redirects the request to a URL that uses the HTTPS protocol and the specified port (40443), but retains the original domain, path, and query parameters of the original URL.\]](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/images/redirect_https_port.png)
The following rule sets up a permanent redirect to a URL that retains the original protocol, port, hostname, and query parameters, and uses the `#{path}` keyword to create a modified path\. This screen is equivalent to "\#\{protocol\}://\#\{host\}:\#\{port\}/new/\#\{path\}?\#\{query\}"\.  

![\[A rule that redirects the request to a URL that retains the original protocol, port, hostname, and query parameters, and uses the #{path} keyword to create a modified path.\]](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/images/redirect_path.png)

**Example Example Redirect Action for the AWS CLI**  
You can specify an action when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following action redirects an HTTP request to an HTTPS request on port 443, with the same host name, path, and query string as the HTTP request\.  

```
[
  {
      "Type": "redirect",
      "RedirectConfig": {
          "Protocol": "HTTPS",
          "Port": "443",
          "Host": "#{host}",
          "Path": "/#{path}",
          "Query": "#{query}",
          "StatusCode": "HTTP_301"
      }
  }
]
```

## Rule Condition Types<a name="rule-condition-types"></a>

The following are the supported condition types for a rule:

`host-header`  
Route based on the host name of each request\. For more information, see [Host Conditions](#host-conditions)\.

`http-header`  
Route based on the HTTP headers for each request\. For more information, see [HTTP Header Conditions](#http-header-conditions)\.

`http-request-method`  
Route based on the HTTP request method of each request\. For more information, see [HTTP Request Method Conditions](#http-request-method-conditions)\.

`path-pattern`  
Route based on path patterns in the request URLs\. For more information, see [Path Conditions](#path-conditions)\.

`query-string`  
Route based on key/value pairs or values in the query strings\. For more information, see [Query String Conditions](#query-string-conditions)\.

`source-ip`  
Route based on the source IP address of each request\. For more information, see [Source IP Address Conditions](#source-ip-conditions)\.

Each rule can include zero or one of the following conditions: `host-header`, `http-request-method`, `path-pattern`, and `source-ip`, and zero or more of the following conditions: `http-header` and `query-string`\.

You can specify up to three match evaluations per condition\. For example, for each `http-header` condition, you can specify up to three strings to be compared to the value of the HTTP header in the request\. The condition is satisfied if one of the strings matches the value of the HTTP header\. To require that all of the strings are a match, create one condition per match evaluation\.

You can specify up to five match evaluations per rule\. For example, you can create a rule with five conditions where each condition has one match evaluation\.

You can include wildcard characters in the match evaluations for the `http-header`, `host-header`, `path-pattern`, and `query-string` conditions\. There is a limit of five wildcard characters per rule\.

### HTTP Header Conditions<a name="http-header-conditions"></a>

You can use HTTP header conditions to configure rules that route requests based on the HTTP headers for the request\. You can specify the names of standard or custom HTTP header fields\. The header name and the match evaluation are case\-insensitive\. The following wildcard characters are supported in the comparison strings: \* \(matches 0 or more characters\) and ? \(matches exactly 1 character\)\. Wildcard characters are not supported in the header name\.

**Example Example HTTP Header Condition for the AWS CLI**  
You can specify conditions when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following condition is satisfied by requests with a User\-Agent header that matches one of the specified strings\.  

```
[
  {
      "Field": "http-header",
      "HttpHeaderConfig": {
          "HttpHeaderName": "User-Agent",
          "Values": ["*Chrome*", "*Safari*"]
      }
  }
]
```

### HTTP Request Method Conditions<a name="http-request-method-conditions"></a>

You can use HTTP request method conditions to configure rules that route requests based on the HTTP request method of the request\. You can specify standard or custom HTTP methods\. The match evaluation is case\-sensitive\. Wildcard characters are not supported; therefore, the method name must be an exact match\.

We recommend that you route GET and HEAD requests in the same way, because the response to a HEAD request may be cached\.

**Example Example HTTP Method Condition for the AWS CLI**  
You can specify conditions when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following condition is satisfied by requests that use the specified method\.  

```
[
  {
      "Field": "http-request-method",
      "HttpRequestMethodConfig": {
          "Values": ["CUSTOM-METHOD"]
      }
  }
]
```

### Host Conditions<a name="host-conditions"></a>

You can use host conditions to define rules that route requests based on the host name in the host header \(also known as *host\-based routing*\)\. This enables you to support multiple domains using a single load balancer\.

A hostname is case\-insensitive, can be up to 128 characters in length, and can contain any of the following characters:
+ A–Z, a–z, 0–9
+ \- \.
+ \* \(matches 0 or more characters\)
+ ? \(matches exactly 1 character\)

You must include at least one "\." character\. You can include only alphabetical characters after the final "\." character\.

**Example hostnames**
+ **example\.com**
+ **test\.example\.com**
+ **\*\.example\.com**

The rule **\*\.example\.com** matches **test\.example\.com** but doesn't match **example\.com**\.

**Example Example Host Header Condition for the AWS CLI**  
You can specify conditions when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following condition is satisfied by requests with a host header that matches the specified string\.  

```
[
  {
      "Field": "host-header",
      "HostHeaderConfig": {
          "Values": ["*.example.com"]
      }
  }
]
```

### Path Conditions<a name="path-conditions"></a>

You can use path conditions to define rules that route requests based on the URL in the request \(also known as *path\-based routing*\)\.

The path pattern is applied only to the path of the URL, not to its query parameters\.

A path pattern is case\-sensitive, can be up to 128 characters in length, and can contain any of the following characters\.
+ A–Z, a–z, 0–9
+ \_ \- \. $ / \~ " ' @ : \+
+ & \(using &amp;\)
+ \* \(matches 0 or more characters\)
+ ? \(matches exactly 1 character\)

**Example path patterns**
+ `/img/*`
+ `/js/*`

The path pattern is used to route requests but does not alter them\. For example, if a rule has a path pattern of `/img/*`, the rule would forward a request for `/img/picture.jpg` to the specified target group as a request for `/img/picture.jpg`\.

**Example Example Path Pattern Condition for the AWS CLI**  
You can specify conditions when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following condition is satisfied by requests with a URL that contains the specified string\.  

```
[
  {
      "Field": "path-pattern",
      "PathPatternConfig": {
          "Values": ["/img/*"]
      }
  }
]
```

### Query String Conditions<a name="query-string-conditions"></a>

You can use query string conditions to configure rules that route requests based on key/value pairs or values in the query string\. The match evaluation is case\-insensitive\. The following wildcard characters are supported: \* \(matches 0 or more characters\) and ? \(matches exactly 1 character\)\.

**Example Example Query String Condition for the AWS CLI**  
You can specify conditions when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following condition is satisfied by requests with a query string that includes either a key/value pair of "version=v1" or any key set to "example"\.  

```
[
  {
      "Field": "query-string",
      "QueryStringConfig": {
          "Values": [
            {
                "Key": "version", 
                "Value": "v1"
            },
            {
                "Value": "example"
            }
          ]
      }
  }
]
```

### Source IP Address Conditions<a name="source-ip-conditions"></a>

You can use source IP address conditions to configure rules that route requests based on the source IP address of the request\. The IP address must be specified in CIDR format\. You can use both IPv4 and IPv6 addresses\. Wildcard characters are not supported\.

If a client is behind a proxy, this is the IP address of the proxy not the IP address of the client\.

This condition is not satisfied by the addresses in the X\-Forwarded\-For header\. To search for addresses in the X\-Forwarded\-For header, use an `http-header` condition\.

**Example Example Source IP Condition for the AWS CLI**  
You can specify conditions when you create or modify a rule\. For more information, see the [create\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/create-rule.html) and [modify\-rule](https://docs.aws.amazon.com/cli/latest/reference/elbv2/modify_rule.html) commands\. The following condition is satisfied by requests with a source IP address in one of the specified CIDR blocks\.  

```
[
  {
      "Field": "source-ip",
      "SourceIpConfig": {
          "Values": ["192.0.2.0/24", "198.51.100.10/32"]
      }
  }
]
```
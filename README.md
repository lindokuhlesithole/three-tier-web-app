<h1 align="center">Three-Tier Web Application on AWS</h1>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white" alt="S3" />
  <img src="https://img.shields.io/badge/AWS-CloudFront-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white" alt="CloudFront" />
  <img src="https://img.shields.io/badge/AWS-API%20Gateway-FF4F8B?style=for-the-badge&logo=amazonaws&logoColor=white" alt="API Gateway" />
  <img src="https://img.shields.io/badge/AWS-Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white" alt="Lambda" />
  <img src="https://img.shields.io/badge/AWS-DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white" alt="DynamoDB" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/serverless-architecture-brightgreen?style=flat-square" alt="Serverless" />
  <img src="https://img.shields.io/badge/python-3.11-blue?style=flat-square&logo=python" alt="Python" />
  <img src="https://img.shields.io/badge/status-production-success?style=flat-square" alt="Status" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-Solutions%20Architect%20Professional-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS SA Pro" />
  <img src="https://img.shields.io/badge/AWS-Security%20Specialty-232F3E?style=for-the-badge&logo=amazonaws&logoColor=FF9900" alt="AWS Security" />
</p>

---

## Build a Three-Tier Web App

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_2b3c4d5e)

---

## Introducing Today's Project!

It is time for me to start creating production-quality serverless three-tier web application from scratch. 😤 Time to leave the tutorials and build something to launch.
Here is my tech stack:
🖼️ Presentation Layer - S3 + CloudFront: S3 frontend with CloudFront distribution to deliver globally. "Works on my machine" no more.
⚡ Application Layer - API Gateway + Lambda: Serverless because I do not want to patch EC2 instances at 2 AM. API Gateway directs traffic, Lambda runs my code and I pay per usage.
🗄️ Data Layer - DynamoDB: DynamoDB because I need a database capable of dealing with fluctuating traffic without scaling manually. Single-digit millisecond latency.

#AWS #CloudEngineering #Serverless #JuniorCloudEngineer #Lambda #DynamoDB #APIGateway #CloudFront #S3 #ThreeTierArchitecture #BuildInPublic #CloudProjects #InfrastructureAsCode #HireMe #TechJobs #AlwaysLearning #CloudComputing #CareerGrowth #AWSCommunity

### Tools and concepts

From this project, I gained experience in implementing a serverless three-tier architecture through S3, CloudFront, API Gateway, Lambda, and DynamoDB.
S3 and CloudFront gave me the skills to host and deliver a static frontend without needing a web server. API Gateway introduced me to creating a secured entry point for backend requests. Lambda demonstrated that it was possible to run backend logic without setting up servers by simply writing code on AWS. DynamoDB introduced me to the idea of having infinite scalability with NoSQL databases without needing to edit any config files.
However, the biggest takeaway from this project is CORS. Most of the time was dedicated to dealing with cross-origin errors rather than building the application itself. The CORS issue has two layers; first, API Gateway takes care of preflight OPTIONS, and second, Lambda needs to set the required response headers. Moreover, API Gateway updates only take place after deploying the project again.

### Project reflection

It took me approximately 4 to 5 hours to develop this application – and trust me, half of them I spent trying to beat CORS. 😅
Setting up the cloud architecture went quickly: S3 bucket, CloudFront distribution, API Gateway, Lambda function and DynamoDB were all ready within an hour or so; it also took me around one hour to implement frontend and backend logic. But then I spent probably forever trying to fix cross-origin issues, redeploying API Gateway, adjusting headers of Lambda function and refreshing my browser over and over again.
The time when user data finally showed up in my CloudFront application was absolutely rewarding. The three hours spent fixing everything were educational for me way more than ten hours without any issues would be.

I chose to do this project today because... Something that would make learning with NextWork even better is...

---

## Presentation tier

At the presentation tier level, I am configuring S3 and CloudFront services.
My frontend will be hosted on the S3 bucket as a static website. I am uploading my HTML, CSS, JS code, and other assets to S3, and then serving them via HTTP without even running one single server.
After that, I am going to configure CloudFront in front of it. CloudFront service provides CDN solution for caching my content around the world on edge locations, so users will always receive my website in just under a second. In addition to that, it provides features like automatic HTTPS, custom domains support, and DDoS protection. And since my origin is S3, I will not need to worry about any server failures.
S3 + CloudFront is my presentation tier now – quick, secure, scalable, and globally accessible. No web servers, no patching, no scaling issues whatsoever. Just my frontend that loads super quickly and looks good.

I accessed my website from the CloudFront distribution's domain name – d1awba178a0c9a.cloudfront.net.
After uploading the frontend files to the S3 bucket and setting up the S3 bucket for static website hosting, I set up a CloudFront distribution using the S3 bucket as the origin. This domain was provided by CloudFront, and it is the one that I entered in the browser for the live "User Information" page to appear.
Alternatively, I could use the S3 website endpoint; however, this would only serve over HTTP without any benefits offered by a CDN. Also, it does not look as professionally as it should when I am presenting it to recruiters.
The sight of the rendered "Enter User ID" input field and the "Get User Data" button is proof that my presentation tier works perfectly. No web servers needed, no need to perform security patches – just an instant frontend.
In the future, I will use the Route 53 service to set up my custom domain name.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_3a4b5c6d)

---

## Logic tier

With regard to the logic layer, I will be configuring API Gateway and Lambda.
The API Gateway serves as a gateway to all the backend requests from my users. The API Gateway provides routing, validation, rate limiting and authentication services for my application. Once the "Get User Data" button is clicked by a user, a JavaScript client will make an HTTP request to the API Gateway. The API Gateway will route the request to a specific lambda function based on the path and the method.
The Lambda functions will hold my actual business logic. Lambda allows me to execute Python functions on-demand without having to provision servers or worry about patching and scaling. Once the API Gateway invokes the appropriate Lambda, it executes my code, processes the request, connects to DynamoDB when necessary and sends the response back to the frontend.
My logic layer consists of API Gateway and Lambda and it is completely serverless, auto-scalable and integrates with the rest of my AWS stack

Lambda function retrieves data through queries on DynamoDB through AWS SDK.
I import boto3 and instantiate DynamoDB client/resource within my Lambda handler. When Lambda is invoked via API Gateway, I retrieve user ID from event parameter – usually it comes either from query or path parameters. After that, I perform a call to dynamodb.get_item() or dynamodb.query() methods using the obtained user ID as a key and my table created specifically for this project as a target.
DynamoDB response will contain an item as a JSON object, which Lambda will transform into an appropriate HTTP response with 200 status code and necessary headers. In case the user ID is non-existent, DynamoDB will return an empty response, and my Lambda function will properly handle it and return 404 or "not found" response.
It takes a matter of single-digit milliseconds since Lambda and DynamoDB are AWS services communicating through AWS internal networks without any internet latencies and connection pools.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_6a7b8c9d)

---

## Data tier

In the data tier, I am configuring DynamoDB.
It is a fully managed NoSQL database service that takes care of all my persistent storage needs. It is used to store user data, including user IDs, names, emails, and any other data that my application might need to access or modify. As it is a NoSQL database, there is no need for me to define strict schemas. I simply create a table, specify a primary key, and begin storing my data.
Why DynamoDB fits this architecture perfectly? The integration with Lambda. My Lambda functions work directly with DynamoDB with the help of IAM roles without the need for managing connections pools and scaling manually. DynamoDB will automatically scale depending on the load generated by my API Gateway and Lambda functions and provide sub-millisecond performance no matter the size of the load.
I am billed based on my usage of read and write capacities and there are no additional costs. DynamoDB completes my data tier — fast, scalable, and serverless

I am using DynamoDB as my source and destination of data.
DynamoDB is my data storage for user information. I have a table in which the items correspond to the user, consisting of user id, name, email etc. When anyone inputs a user ID and clicks "Get User Data", my Lambda function fetches that item from DynamoDB with the help of user id through API Gateway.
Being a NoSQL key-value data store, I don't have to care about any joins or fixed schema. All I have to do is choose my partition key, input my data, and DynamoDB does the rest. DynamoDB scales on its own and works seamlessly with Lambda via IAM permissions; no connection strings, no databases, nothing to manage.
DynamoDB is the brain of my data layer – it remembers all that so that I don't have to. All reads and writes take place in sub-milliseconds, ensuring that I give instantaneous response to my users.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_u1v2w3x4)

---

## Logic and Data tier

Here I will start making the three layers interact.
My current front end in S3 is only a fancy page that does not react when the button "Get User Data" is pressed. API Gateway has some endpoints but no calls are made to them. My Lambda can connect to the DynamoDB database, but nobody is using this functionality yet. In this step, I will connect all three layers.
First, I will modify my frontend JavaScript so that it makes HTTP requests to my API Gateway endpoint. I will configure API Gateway for allowing CORS so that my frontend hosted by CloudFront could call it without any problems. I will ensure that my Lambda function has proper IAM permissions to access the DynamoDB database and deliver its content to the user through API Gateway.
Then, the entire process will work: the user inputs his/her ID -> JavaScript makes a call to API Gateway -> Lambda connects to DynamoDB -> returns the data back to the user. Three layers, one cohesive application.

For testing purposes, I used both the API Gateway console and Postman.
Initially, I leveraged the API Gateway test functionality to send request samples directly through the AWS console. Thus, I could verify that my endpoint is accessible, my Lambda is working as intended and my integration responses have correct formatting – without even going to the frontend.
Next step for me was to perform tests with Postman outside the AWS ecosystem. By sending GET and POST requests to my API Gateway endpoint with various user IDs, I could confirm that my Lambda successfully gets data from DynamoDB and processes it in the right way. In addition, I made sure that my API can handle cases when the requested user ID doesn't exist at all.
Lastly, I inspected the CloudWatch logs of my Lambda to review the detailed execution process, timings and errors. When I've confirmed that my API returns valid JSON consistently, it meant it's time to connect my API with the frontend!

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_a112c3d5)

---

## Console Errors

I encountered an issue with CORS on my CloudFront website at d1awba178a0c9a.cloudfront.net.
My frontend application is hosted at that CloudFront domain while my API Gateway is hosted at an entirely different domain. Upon clicking "Get User Data" my JavaScript attempted to call the API and was blocked by the browser due to CORS policy violation. It was saying that "these two domains are not equal and hence you are not allowed to make this call".
The reason why the API Gateway did not allow calls to my API is because it wasn't enabled for requests coming from my CloudFront origin. All I had to do is to configure CORS on my API Gateway resource with Access-Control-Allow-Origin of d1awba178a0c9a.cloudfront.net and allow GET method.
That way my frontend and backend were next to each other and yet not allowed to communicate.
The most frustrating error when everything works in Postman 🏃‍♀️💨 but not in the browser 🙄❌

I uploaded an updated version of script.js because the old one had a reference to the incorrect API Gateway endpoint — or perhaps even no endpoint at all.
Once I addressed the CORS configuration problem for API Gateway, I noticed that my frontend JavaScript file still did not have the location where the "Get User Data" request should be sent. It needed the API Gateway invoke URL, which would allow me to make an HTTP call from my CloudFront frontend to my backend. Without the correct URL, the button served only as a decorative addition, which does not do anything upon being pressed.
Also, I made sure to update the script.js to process the API Gateway response correctly by parsing the JSON and presenting the data on the page. In other words, it turned my static website into a dynamic website with serverless backend communication.

I came across yet another problem since the CORS issue is still preventing my frontend from communicating with API Gateway.
Despite all my efforts to fix the CORS problem previously, the browser console is now yelling at me the same message, "No Access-Control-Allow-Origin header is present". The issue here is that my frontend on the domain d13i8ajhi56kbm.cloudfront.net is trying to access my API Gateway, which still doesn't recognize my CloudFront domain.
Most probably, the root cause of the problem is that I have enabled the CORS setting on my API Gateway console but haven't done a new API Gateway deployment yet. On API Gateway, any changes (such as CORS settings) will not be applied until a new deployment phase is created. That's why the requests that come from my browser are being processed by an old deployment that blocks all cross-origin requests.
It is rather annoying because it can be solved with just one click in API Gateway.

---

## Resolving CORS Errors

To fix the CORS problem, I made modifications in my API Gateway that enabled API Gateway to accept requests coming from my CloudFront domain.
Specifically, I configured my API Gateway to use my CloudFront distribution domain — https://d13i8ajhi56kbm.cloudfront.net — as the Access-Control-Allow-Origin header in the CORS configuration for API Gateway. In essence, API Gateway will accept requests coming from my frontend to prevent my browser from blocking them because they are cross-origin.
In addition, I also specified that the Access-Control-Allow-Methods header should have the GET request included so that my API Gateway could accept my GET requests from the frontend.
After setting the above configurations, I deployed my API Gateway to a new stage to enable it to implement the CORS configuration. This is because any modifications made to the CORS configuration of API Gateway will only take effect once the deployment step is performed.

I updated my Lambda function since CORS headers should be defined in the Lambda response, rather than API Gateway.
While I already set up CORS in the API Gateway level, the browser will check for the response headers of the actual Lambda function. If I did not include the Access-Control-Allow-Origin header in the Lambda, the browser will ignore the response altogether. Thus, I added the said header into my Lambda function code by specifying https://d13i8ajhi56kbm.cloudfront.net as the value.
I placed the header in all the return statements in the headers object — the 200 response for existing users, the 404 response for non-existing users, and the 500 catch block for error handling. Even one missed path will result to rejection of said response by the browser.
This CORS setup is double-layered. API Gateway takes care of the OPTIONS preflight request, while the Lambda takes care of the response headers. They must be consistent.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_1qthryj2)

---

## Fixed Solution

My proof of fixed problem included checking the fixed implementation in my CloudFront website.
I started a browser and went to https://d13i8ajhi56kbm.cloudfront.net. Thereafter, I entered a user id into the input box and pressed 'Get User Data'. This time, there was no CORS error in the browser console but instead I could see that my Lambda returns real user data from DynamoDB. Console was completely clean - no error messages of any kind, just smooth HTTP 200 response moving across the tiers.
To be sure, I have tested my system on edge cases - entered a user id which does not exist in order to check error handling and saw full execution chain of my program in CloudWatch logs. It was perfect.
When data finally appears on my frontend – that is the exact moment when I realize that integration of my three-tier architecture is done.

![Image](http://nextwork.ai/fulfilled_turquoise_beautiful_bear/uploads/aws-compute-threetier_2b3c4d5e)

---

---
## Author

**Lindokuhle Sithole** - *Cloud Engineer | Cloud DevOps Engineer | Cloud Security Specialist*

Based in Bremen, Germany. BSc Mathematical Science from the University of the Witwatersrand. 5x AWS Certified (Solutions Architect Professional, Security Specialty, CloudOps Engineer Associate, Solutions Architect Associate, Cloud Practitioner) plus CompTIA Security+.

- **LinkedIn:** [linkedin.com/in/lindokuhle-sithole-bb701b19a](https://www.linkedin.com/in/lindokuhle-sithole-bb701b19a)
- **GitHub:** [github.com/lindokuhlesithole](https://github.com/lindokuhlesithole)
- **Email:** sitholelindokuhle371@gmail.com

---


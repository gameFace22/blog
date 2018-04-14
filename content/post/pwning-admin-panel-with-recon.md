---
title: "Pwning admin panel with recon"
date: 2018-04-11T20:12:47+05:30
draft: false
---

Reconnaissance is one of the interesting and most critical parts of penetration testing. Using recon, one could yield API endpoints, sensitive files/folders, juicy subdomains and so on. During a recent engagement, I was able to get access to the administrative panel due to lack of authorization and sensitive files being published publicly.

Let us start with analysing the iOS application statically. On going through the [Info.plist](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/uid/TP40009252-SW1), we can see that there is a URL hardcoded.

```bash
[nishaanthguna:~/pentest]$ cat Info.plist | grep -i "http"
<!DOCTYPE plist PUBLIC .. "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<string>https://admin.company.com/xyz/api</string>
 ```

Traversing the URL, we get a page with link to the Swagger UI.

![swagger](/images/traverse-swagger.jpg)

Looking through the [documentation](https://swagger.io/swagger-ui/), we can see that Swagger UI can be used to visualize and interact with API's resources and that it is automatically generated from the specification. Following was the UI page retrieved while visiting the URL found from the previous step.

![admin-swagger](/images/admin-swagger.jpg)

Weirdly, not only API calls made from the mobile application, but also the endpoints which an admin can use to manage users, distribute content, customize the chatbot the app uses were exposed. With the knowledge of extra endpoints, I fired up Burp to see the network traffic. Initially, my idea was to replace the endpoints of the mobile application with the admin endpoints found from Swagger UI to check if there was proper authorization to differentiate between normal and administrative users.

From the 'Admin User' API documentation, we can see there is an endpoint to print out the admin user count using /admin/users/count. This looks promising since it does not require any request body and is pretty straightforward.

Logged into the mobile application as a normal user, I replaced one of the API calls from /xyz/api/users/account/preferences to /xyz/api/admin/users/count and forwarded the request.

![admin-count](/images/admin-count.jpg)

It worked! This means the server does not have any authorization in place. Basically, we can make a request to any API endpoint including /admin, /chatbot, /moderate considering we know the structure of the request body and the headers involved. Let us try to enumerate more with the endpoints from Swagger and escalate this.

From the Swagger UI attachment, we can see that there is another endpoint to find information about admin users by making a request to /admin/users/{id}

```
GET /xyz/api/admin/users/1 HTTP/1.1
Host: https://admin.company.com
User Agent: MS-RELEASE/1.0.32 (iPhone; iOS 10.1.1; Scale/2.00)
Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkdldCB5b3VyIG93biB0b2tlbiEiLCJpYXQiOjE1MTYyMzkwMjJ9.12neWKBPl2q0alhnEiJ_g018_0YHtZMaFzCjsWs0VE

{
  "ID": 1,
  "Name": "Admin User",
  "Username": "XYZ",
  "EMail": "dev@nonexistingdomain.com",
  "Roles": [
    {
      "ID": 1,
      "Name": "Admin",
      "Menu":[
        {
          "Path": "#/admin",
          "Icon": "fa-user",
          "Order": "1",
          "Roles": "READ,WRITE",
        }
      ]
    }  
}       
 ```

Awesome! Using Burp's Intruder, we can retrieve all the 8 admin's username, e-mail by enumerating the {id} parameter.

![enumerate-admin](/images/enumerate-admin.jpg)

Now that we have the admin usernames to login. Let us grab common password list from [Seclists](https://github.com/danielmiessler/SecLists/tree/master/Passwords/Common-Credentials) and run a quick bruteforce before reporting.

Luckily, one of the admin users had a weak password and the application didn't have any rate limiting in the login page. With the obtained admin access, we could do anything from adding or removing users, modify the content which is displayed in the mobile application, send notifications to the end users and a lot of interesting things.

![admin-panel](/images/admin-panel.jpg)

It was fun to put bits and pieces together to get admin access on the web application. I have also written a primer on static analysis of iOS applications at [SecDevOps.](https://secdevops.ai/ios-static-analysis-and-recon-c611eaa6d108) Do check it out ;)

Please don't hesitate to post feedback or comments below. You can also DM me on Twitter if you'd like.

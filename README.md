# PowerAutomateHTTPautomation
Automating web browsing, form filling, table searching and hair pulling out with the help of power automate, desktop and cloud. Not all of us are allowed python at work :(



The issue at hand is that we have only just managed to convince our infosec department to allow us to use power automate on our work computers and servers.
Power automate is microsofts solution to no/low code programming. It was renamed from Microsoft flow and many of the old forum pages with solutions can be found by googling things for microsoft flow and using archive.org to read those discussions that microsoft has removed.
Unfortunatly this means power automate is fake programming that is more like scratch but... scratch is still turing complete.
Power automate desktop has the HTTP action avaliable, and it looks like this


![image](https://github.com/user-attachments/assets/7ff40fbd-9246-498d-9362-158005704377)


As you can see we have a all the neccisary bits and pieces to script things. Alas we don't have beautiful soup, requests or any of the other wonderful libraries that make life easy when scripting in python (my previous experience with web scripts/scrapes). 
This action does however do some useful things behind the scenes. At first I was dreading manually dealing with cookies, but if you send a GET request to site abc.xyz and it give you a session cookie, that session cookie will still be used when you try to POST to abc.xyz/login. This should really be mentioned in some documentation somewhere but it doesnt exists :(.
Other than that, all the other boxes do pretty much what you'd expect.


I started off using the desktop version and still used it as a connection to some local databases, but eventually I moved to using Power automate cloud. The same option looks like this:

![image](https://github.com/user-attachments/assets/f532fff0-cf0f-4653-849e-c45487ebcdcb)

The headers are put in individually (or JSON), queries can also be put in individually (or JSON), unlike power automate desktop.


## talk more about how power automate actions work etc. 


## What am I automating?
At work we use a fake sql server with a hat on (FSQLWAH) that calls itself a program to recieve, update and send work orders. We are on the remote team so we fix things that can be fixed before field engineers have to go somewhere. Our clients are a variety of retail operations and each contract has a slightly different process.
Several use an web application. This application (called abc.xyz) is where our helpdesk recieves the jobs that they pass to us. This application is also where clients get updates on the status of their jobs. Four clients use this application and one is somewhat more carpal tunnely than others as it requires us to click through an asbestos permit... despite the fact that we work remotely. We get the abc.xyz number passed to us within the FSQLWAH so we can start, pause and complete work orders depending on the status. This is also where out SLAs are monitored so it is important these are updated.

But, all this information is avaliable within FSQLWAH, so why not try to automate it? 
Some of my coworkers had already automated a specific type of preventative maintainance jobs with automating browser actions and clicks, but its slow and somewhat unreliable.
So why not see if it can be automated with HTTP.


## So what HTTP do I need to do.

1. Login to the site (different logins for each client)
2. Search for the work order
3. Add the permit
4. Start work
5. Pause/Complete work
6. Make sure that its finished so we dont miss the SLA


## Issues
No idea how things are encoded in the body or headers.
No idea how cookies work
No idea how the session works
No way to test things as I can only use our actual verisaes that we need to complete to test on.

## Step 1, the login
If I get one login, i'll have them all. The different clients are just different logins that are used on the same site.
Lets start by capturing a request in dev tools.

![Screenshot_2025-06-02_00-22-05](https://github.com/user-attachments/assets/f421f8f0-3452-4072-b34d-c672facca90a)


As we can see, fairly standard headers.
Two cookies, the JSESSIONID (important) and the _enid (doesnt seem to matter).
And the body of the request:

![Screenshot_2025-06-02_00-25-38](https://github.com/user-attachments/assets/00269fc5-b57f-403d-85cb-6cde648480f8)


Ok so theres a CSRF that we will need. The username and password go in the login and password fields towards that bottom.
So lets look and the login page and search for the CSRF:

![Screenshot_2025-06-02_00-29-05](https://github.com/user-attachments/assets/2750a4fd-24ce-4249-92b9-db03c81e12b1)

Easy peasy, all we have to do is grab it from the login page and put it in our POST.
Lets capture a login by sending it to requestcatcher.com:

![image](https://github.com/user-attachments/assets/2b9d472e-df1a-4b2a-bca2-0050ca9b9898)

and comparing this to what we sent.

![image](https://github.com/user-attachments/assets/4d9126bb-d6e8-45b7-8393-8dd1ddb47867)

We can see that content-length is automatically added, as well as a load of X-Ms trackers. The important headers for our request are probably just Accept, Accept-Encoding, Content-Type, Cookie and possibly Referer (if the site is tracking your actions using it).

The body of our request is encoded. Content-type application/x-www-form-urlencoded body will need to follow certain rules. The fields will be seperated by '&' and spaces will be encoded as '+'. Fields that aren't used can probably be blank or left out.

Cookies are often used similarly to CSRF tokens, in my case I needed to POST my login with the cookie that I was provided on my first request of the site. The other cookie, called _enid changes every request but seems unimortant to the functioning of the site.


## Post Login

We do have a slight issue though, a successful login results in a redirect, a 302 response.
Power automate hates redirects for some mysterious reason. Even when successful we get a 200, but luckily we do get a Set-Cookie response header with a lovely authenticated JSESSIONID.
With this JSESSIONID we can request that page we were supposed to be redirected to and hurray! We clearly have managed to log in!
We will be keeping this cookie close to hand from here on out. Every request will be using this cookie in its body.

To find our work order, we will look at how searching a table works. Basically capturing the request and looking at where your search term is should make it fairly obvious how to automate this.
In my case this search was also a www-form-urlencoded. The thing to be careful of is having invisible newlines or any extra spaces. In which case you may not find what you are looking for with your search.

I then crop the table to get the results. Cropping is a bit of a pain but this gave me the link to the work order I was looking for as well as the details of the current work order status.

## multipart/form-data

So far so good, everything has worked out quite nicely. Here is where we hit an issue. One of the posts I want to automate is a multipart/form-data.
This content type uses a boundary that must be unique and different from any other content. The details for exactly how this is formatted were gotten from here: https://www.rfc-editor.org/rfc/rfc7578

This caused me issues for some time. All of my first requests gave me javascript runtime errors or a base 64 reponse of "eyJhY3Rpb24iOiAiZXJyb3IifQ==" which using cyberchef, decodes to {"action": "error"}

Here is what I was able to capture from BurpSuite: 
------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="work_permission_id"','\r\n\r\n','','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="contractorRules"','\r\n\r\n','agree','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="risk_NO_PERMIT_REQUIRED"','\r\n\r\n','NO_PERMIT_REQUIRED','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="desc_of_work"','\r\n\r\n','','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="requires_approval"','\r\n\r\n','false','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="haz_prompt_answer_16029707893"','\r\n\r\n','Y','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="haz_prompt_answer_16029707895"','\r\n\r\n','Y','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="haz_prompt_answer_16029707896"','\r\n\r\n','N','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="haz_prompt_answer_16029707897"','\r\n\r\n','','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="risks_prompt_answer_12404344777"','\r\n\r\n','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="risks_prompt_answer_12404344778"','\r\n\r\n','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="is_technician_authorized"','\r\n\r\n','true','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl','\r\n','Content-Disposition: form-data; name="save_action"','\r\n\r\n','SUBMIT','\r\n','------WebKitFormBoundaryruK8Sn6ZNe4pM5vl--','\r\n'


Notice that each boundary starts with 2 extra hyphens and the end of the form has the boundary, 2 hyphens and a carriage return new line (CRNL).

Attempting to replicate this, we run into an issue. This is what the action looks like if we write in the request as we would expect it to look.

![image](https://github.com/user-attachments/assets/20bcb40b-f190-4a16-a12e-766ede7fdcf1)

but when we look at the code view of the action:

![image](https://github.com/user-attachments/assets/7146c787-bbe4-4a08-b6af-622deb852850)

From this snippit, and repeated errors from the website we are trying to POST to, we can see issues.
From the rfc, we know that the webform expects carriage return, newlines (\r\n).
What we have is simply newlines (\n).

This is an issue. The way I was able to work around this was gotten from https://ecellorscrm.com/2024/08/11/alternatives-to-update-power-automate-from-code-quick-review/.
Unfortunatly, this is only an option for Power Automate cloud, not desktop. I havent found a way to do webforms with only PAD.

If we go to the old designer of Power automate cloud

![image](https://github.com/user-attachments/assets/f7da57c1-7769-4ca6-b10e-0333a773300d)

We can copy and action and put it into notepad.

![image](https://github.com/user-attachments/assets/02991ed3-12c4-43ce-9bd9-0d1cf2324696)

From here we can direcly edit the code of the action. replacing each \n with \r\n.

We now go to clipboard in old Power automate cloud designer.

![image](https://github.com/user-attachments/assets/dcc806f7-2936-4a07-9e3a-eb967834f882)

And in paste (press CTRL V) our edited action.

![image](https://github.com/user-attachments/assets/0ba1b34e-d0da-43f1-9191-95094082e428)


Now, posting our form, we get {"action": "start_work", "tech_id": "22142086860"} as a reponse! No more errors.


There were no other major difficulties in automating my actions, other than regularly having newlines that got added into fields and made things fail.
When in doubt, check the code view for newlines. When copy and pasting, power automate love invisible newlines.

Also power automate hates quotation marks in headers (sometimes). These never bothered me too much as I tended to just delete the header. Most of them weren't neccisary.

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

What headers do we need?
How do they work?
How does the body work?

We do have a slight issue though, a successful login results in a redirect, a 302 response.
Power automate hates redirects for some mysterious reason. Even when successful we get a 200, but luckily we do get a Set-Cookie response header with a lovely authenticated JSESSIONID.
With this JSESSIONID we can request that page we were supposed to be redirected to and hurray! We clearly have managed to log in!
We will be keeping this cookie close to hand from here on out.

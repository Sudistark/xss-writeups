# Interesting case of a DOM XSS in www.figma.com


Recently I was able to find a DOM based xss in www.figma.com in collaboration with [@huli](https://twitter.com/aszx87410) (an awesome ctf player).

The cause of the XSS is really interesting, at first sight if you are not aware of the weird browser quirk everything looks secure as it's going through a sanitization process using the unfamous sanitizer "DomPurify" , props to huli for identifying this.


Figma is really a tough target I feel, they have really done very well securing their site.So if you are looking for a tough target with a good security team then give a shot to Figma program. I was focusing on their desktop app which is build in Electron hoping I could learn more about Electron hacking stuff along with it.So I thought maybe if I can xss somewhere that could be useful in the Desktop app.

After some time looking here there, I was able to find a place where it allowed the user to make the description text bold,italic this looked interesting.

Users can publish their design to the public , it's accessible under this url: `https://www.figma.com/community/file/*`

![image](https://user-images.githubusercontent.com/31372554/276819715-a519019f-626c-44c3-a761-616bc34666dd.png)

In this screenshot you could see that we can some do some styling stuffs such as bold,italic. Upon intercepting the submit request for this:

I noticed raw html tags there.

```json
{
  "name": "Published to Community hub",
  "description": "<p><strong><em>shirley</em></strong></p>"
}
```

If you try to include `<>` directly from the editor (website UI) they will appear as html encoded in the request:

```json
{
  "name": "Published to Community hub",
  "description": "<p><strong><em>shirley&lt;img src=x onerror=alert()&gt;</em></strong></p>"
}
```

So instead I directly edited the raw request and modified the `description` key to include a xss payload:

```json
{
  "name": "Published to Community hub",
  "description": "<p><strong><em>shirley<img src=x onerror=alert()></em></strong></p>"
}
```

The success response  had the full payload as it is there, this indicated that if there was any sanitization it would be happening on the client side only not server side.

![chrome_0Tq9BqZHXy](https://user-images.githubusercontent.com/31372554/276824564-d11e70d0-79cc-469f-9a6b-7a6640fa0b30.png)

From *Inspect Element* I could confirm that the img tag was removed,so surely some sanitization was there.

------------------------

Using DOMInvader , I found where the sanitization is occuring:

![chrome_0o4gHfz2j8](https://user-images.githubusercontent.com/31372554/276832784-5e452e93-9d73-4264-9ce3-0338b1946185.png)
![image](https://user-images.githubusercontent.com/31372554/276832865-0f1341e2-85d2-49a5-bb66-56ef0cd077a0.png)

https://www.figma.com/esbuild-artifacts/ea8217961882eb1214f870449504b1c89251179b/js/figma_app.min.js:3213:35988

```js
        let p = document.createElement("div");
        p.innerHTML = e; // [1]
        let f = IFs.map(y=>`:not(${y})`).join("") // [2]
          , g = p.querySelectorAll(f);
        for (let y of g)
            (h = y.parentNode) == null || h.removeChild(y);
        r.current.innerHTML = HAm.default.sanitize(p.innerHTML) // [3]
```

`e` variable contained the description key value.
On line [1] you could see the user controllable input is assigned to innerHTML property of a newly created div element (`createElement("div")`). As it's not currently added to the dom yet this is fine.


From line [2], the code removes all the tags from the input (description field) which are not in the IFs array (considered it to be an whitelist of allowed tags)

```js
>IFs
(20)['a', 'span', 'sub', 'sup', 'p', 'b', 'i', 'pre', 'code', 'em', 'strike', 'strong', 'h1', 'h2', 'h3', 'ul', 'ol', 'li', 'hr', 'br']
```

After this modification, it is then sanitized HAm is nothing but dompurify object itself

```js
HAm.version
'2.3.1'

```
Also if you search for dompurify in the same js file you will find this:

```js
dompurify/dist/purify.js:
  (*! @license DOMPurify 2.3.1
```

The version used here is pretty old, the latest one is 3.0.6.
But there are no known bypasses so it's fine.

Finding 0day in dompurify isn't an option here neither I am skilled enough to find one so what else can we do in this situation?

The only possible solution I had in my mind was do something via DOM Clobbering. An example case of dom clobbering in the wild can be found here: https://research.securitum.com/xss-in-amp4email-dom-clobbering/

But still I am not good with that, so instead I tried reaching out to some CTF players. I really respect CTF players when it comes to exploiting such bugs they are the ones you should reach out to as they are aware with many weird quirks which not everyone is aware of.



------------------------

Shared the details with Huli, the next day he tells me that he is able to execute js but CSP is blocking it. 

When I saw the payload  I didn't believed it:

![image](https://user-images.githubusercontent.com/31372554/276841524-00bdcff1-b628-48cc-877a-46fa9673c339.png)

```html
<img src=x onerror=alert()>
```

I was really baffled how could this simple payload can bypass a sanitizer like DOMPurify. I really had a hard time understanding this at first even after huli tried explaining it many times.

Some snapshot for you to understand also what actually happened:

![image](https://user-images.githubusercontent.com/31372554/276843048-a122d366-8150-4085-9272-3e638a25cb9e.png)
![image](https://user-images.githubusercontent.com/31372554/276843242-20e09a8c-7fba-4c7b-80cf-4aab57095e1a.png)
![image](https://user-images.githubusercontent.com/31372554/276843373-c4b65d58-14d3-4c0d-a0ae-bd51b5becee1.png)

https://x.com/ZeddYu_Lu/status/1421091362410156032?s=20

![image](https://user-images.githubusercontent.com/31372554/276843791-aa0bad98-153e-4cbe-a84c-5f21ca0689ec.png)

------------------

Try it yourself ,open developer tools try this

```js
 let p = document.createElement("div");
p.innerHTML = "<img src=x onerror=alert()>";
```

Even though the div element isn't added to the DOM it still **executes** . It's like  a magic really.

-----------------


We really tried escalating this, but the strict csp blocked  our all attempts. We decided to report it as it is without CSP bypass, mature programs often accepts xss bugs even without CSP eg: is GoogleVRP

If you are curios here's the csp:

```
script-src 'unsafe-eval' 'nonce-PVEIuETDGJR+8hIA6PqgIQ==' 'strict-dynamic' ; 
```

The reporting experience was very smooth, the Figma sec team is really professional. The bug was fixed (in a week after triage which I consider really great even for bugs like xss) was soon and the team "really liked this cool bug" they said :)

We were awarded 1k$ for this bug and the severity was scored as Medium.
![image](https://user-images.githubusercontent.com/31372554/276845729-faba7890-2c4c-4d33-a666-d24ccd3c6b2d.png)

I asked them though if severity was set to Medium because we were not able to provide a csp bypass and they gave their explanation which I happily agree with.

![image](https://user-images.githubusercontent.com/31372554/276846064-25bb917d-36c2-429c-9c97-a9a9677f783f.png)


Overall it was a great experience submitting a report to Figma, their team is reall great.

Note: If you got more details on that innerHTML quirk would be happy if you could explain in the tweet reply why it really works

-------------------------

**Fix:**

After the fix the code responsible for sanitization was changed to this:

Fixed version
```js
        let p = document.createElement("div");
        p.innerHTML = Hum.default.sanitize(e); // [1]
        let f = IFs.map(y=>`:not(${y})`).join("")
          , g = p.querySelectorAll(f);
        for (let y of g)
            (h = y.parentNode) == null || h.removeChild(y);
        r.current.innerHTML = p.innerHTML
```

Before it was like this (vulnerable version):

```js
       let g = document.createElement("div");
        g.innerHTML = e; // [1]
        let f = nJe.map(v=>`:not(${v})`).join("")
          , b = g.querySelectorAll(f);
        for (let v of b)
            (y = v.parentNode) == null || y.removeChild(v);
        n.current.innerHTML = tAr.default.sanitize(g.innerHTML) // [2]
```
The root cause of xss bug was because of passing user controllable input to innerHTML at line [1] and later sanitization on line  [2]. Although the div element was not added to the dom it still executed, it's one of the weird quirks of browsers.
 
This line is now edited and now the input is sanitized first, then only it's passed to innerHTML.

This changes ensures that the same xss bug can't be trigger now.

As you can noticed that after the sanitization the code is trying to remove all tags which are not in `IFs` array from the sanitized output.
Making any changes to the sanitized data can lead to unexpected problems even xss sometimes like you could see in these findings by Sonar Research team https://www.sonarsource.com/blog/code-vulnerabilities-leak-emails-in-proton-mail/

So @huli had  a suggestion to use the sanitize method twice, for eg:

```js
        let p = document.createElement("div");
        p.innerHTML = Hum.default.sanitize(e); // [1]
        let f = IFs.map(y=>`:not(${y})`).join("")
          , g = p.querySelectorAll(f);
        for (let y of g)
            (h = y.parentNode) == null || h.removeChild(y);
        r.current.innerHTML = Hum.default.sanitize(p.innerHTML)
```

On the last line you could see that sanitize method is called again, this will ensure that even after modification, only the safe html will added to the DOM.

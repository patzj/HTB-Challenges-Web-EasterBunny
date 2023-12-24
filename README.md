# HTB-Challenges-Web-EasterBunny

## Challenge Description

It's that time of the year again! Write a letter to the Easter bunny and make your wish come true! But be careful what you wish for because the Easter bunny's helpers are watching!

![Screen Shot 2023-12-24 at 14 29 15](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/1c1d7969-b2a8-4fcc-9974-bef6c91816e6)

## Reconnaissance

Checking out the app reveals two main functionalities: saving and viewing letters to the Easter Bunny. There is one letter that is hidden, so I assume this is where the flag is located.

![Screen Shot 2023-12-24 at 14 29 28](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/0639046a-074e-4228-b324-235a466489de)

Upon inspecting the code, we can observe that sending a letter is accomplished through an AJAX request using `fetch`, as defined in `main.js`. Internally, after successfully saving the letter to the database, it will be loaded in the background via puppeteer. The purpose of this step is unclear, but the hidden message can be viewed through this process.

```js
// routes/routes.js
if (message.hidden && !isAdmin(req))
    return res.status(401).send({
        error: "Sorry, this letter has been hidden by the easter bunny's helpers!",
        count: count
    });
// ...
```

```js
// utils/authorisation.js
const isAdmin = (req, res) => {
  return req.ip === '127.0.0.1' && req.cookies['auth'] === authSecret;
};
// ...
```

As for viewing letters, the web application displays the letter on the `/letters` route and loads the data via AJAX, similar to the saving process. The letter that is fetched can be controlled by the `id` query parameter and is then passed to the `/message/:id` route to retrieve data from the database. The entire process is defined in `viewletter.js`.

```js
// static/viewletter.js
const loadLetter = () => {
  stampContainer.classList.remove('stamped');
  idLabel.innerText = currentLetter;

  fetch(`/message/${currentLetter}`)
    .then(response => response.json())
// ...
```

It's also worth noting the usage of the `<base>` element, which is employed to resolve relative imports such as scripts and stylesheets. Additionally, we can observe that its value is derived from the request headers and embedded into the HTML using `nunjucks`.

```js
// routes/routes.js
router.get("/letters", (req, res) => {
    return res.render("viewletters.html", {
        cdn: `${req.protocol}://${req.hostname}:${req.headers["x-forwarded-port"] ?? 80}/static/`,
    });
});
// ...
```

```html
<!-- views/base.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="{{cdn}}" />
```

## Scanning

Now that we have identified the crucial components comprising the primary features of the web application, it's time to search for vulnerabilities.

A brief attempt to inject HTML tags into the letter confirmed that it is not susceptible to XSS.

![Screen Shot 2023-12-24 at 14 30 23](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/6533693c-101f-46b1-a2b0-35cf447d347b)

Upon further exploration of the application using Burp Suite, it was observed that the server caches responses. This is indicated by the HTTP status code 304 and the use of **Varnish**, a caching reverse proxy, as evidenced in the headers.

![Screenshot from 2023-12-24 14-30-56](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/1610700c-8bae-42fc-8fed-70d5a05ff4b9)

Going back a bit, we know that the `<base>` is derived from the request headers. Let's try modifying our headers and see how the `href` changes.

Using Burp Suite Repeater, we'll add `X-Forwarded-Host` and `X-Forwarded-Port` headers and resend a request.

![Screenshot from 2023-12-24 14-31-54](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/eb59ddc1-d736-4735-8308-25dbbfb957c1)

We've successfully injected a controlled value into the host but not the port.

Changing the `Host` header to `127.0.0.1` produces the desired results.. With this, we'll be able to modify the `<base>` to a server that we control and load malicious scripts.

Also, take note that because we are using Burp Suite, the __Target__ remains intact and still forwards requests to the correct web application. Just take a look at the upper right part of the Repeater.

![Screenshot from 2023-12-24 14-32-12](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/c44e7484-337f-4e65-986b-e7fe58dfdd93)

With these steps, poison the cache and get the server to use it.

## Exploitation

Going back, we know that after successfully saving (or stamping) a letter, it gets loaded by puppeteer in the background using the `/letters?id=` route. All of this is driven by `viewletter.js`.

The first thing for me to do is prepare a controlled server and a malicious script to inject. I'll be using https://www.pythonanywhere.com/ to set up a Flask application that will serve the malicious script from the `static` folder.

```py
# flask_app.py
from flask import Flask

app = Flask(__name__,static_folder='static')
```

```js
// static/viewletter.js
fetch("http://127.0.0.1/message/3")
    .then(response => response.text())
    .then(data => {
        fetch("http://127.0.0.1/submit", {
            body: data,
            method: "POST",
            headers: { "Content-Type": "application/json" }
        })
    })
```

Here's the expected sequence of events: Since puppeteer has access to the hidden message, we'll retrieve it from there and submit it as a new letter.

Now, let's "poison" the cache and prompt puppeteer to utilize it. Using Repeater, we'll dispatch a GET request to `/letters?id=`, with the ID set to `<last letter ID> + 1`. This ensures that the next ID loaded by puppeteer when saving a new letter is targeted.

![1](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/320e778b-607c-422d-8228-664ad9e7054e)

The next step involves sending a POST request to save a letter using Repeater.

![Screenshot from 2023-12-24 14-35-13](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/a933a2ee-2043-4513-ad60-4c772ad37494)

One indicator of success is when you resend the previous GET request, and you observe that the `X-Cache-Hits` value is now `2` or if it has skipped a value (e.g., from 1 to 3).

These Burp Suite steps may need to be repeated depending on how the `X-Cache-Hits` value changes. Additionally, the "caching" and "saving" processes should be executed in quick succession.

If we observe that the indicators confirm the effectiveness of the poisoned cache, we can take the ID that we poisoned, add 1 to it, and then view it in the browser. For example, if we cached `/letters?id=10`, we will view `letters?id=11` in the browser.

And there we have it. Flag captured.

![2](https://github.com/patzj/HTB-Challenges-Web-EasterBunny/assets/10325457/5640a60f-1a73-452a-9f15-9aed7dd44153)

## Post-Exploitation

While the genuine purpose and benefit of preloading the recently saved letter remain uncertain, at least in my view, it is vital to evaluate whether known vulnerabilities in a particular solution can be exploited.

In this scenario, we managed to alter the application's workflow by injecting our own code through poisoning the cache employed by an internal process.

However, going beyond this, implementing clear and efficient workflows along with responsible data handling practices will result in a more secure and reliable software. 

## References
- https://pptr.dev/
- https://portswigger.net/burp/documentation/desktop/tools/repeater
- https://portswigger.net/web-security/web-cache-poisoning
- https://varnish-cache.org/intro/index.html#intro
- https://www.pythonanywhere.com/
- https://flask.palletsprojects.com/

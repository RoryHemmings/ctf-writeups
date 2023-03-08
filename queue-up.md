# LACTF Queue-up writeup
### Rory (SavagePatch#7404)

## Setup 

For this challenge we are provided the server architecture, which reveals that
there are 2 servers, a queue server and a flag server. Visiting the queue
servers homepage adds a uuid to your cookies, and puts you in a queue to
receive a flag. However this queue takes around 60 days to complete.

There is also a bypass endpoint (/api/<uuid>/bypass) which bypasses the queue
for a given uuid, however it requires an admin authorization key to access.

Poking around the flag server reveals that there is an endpoint
/api/<uuid>/status, which requests and returns the status of a given uuid while
providing the admin secret authorization. Since the uuid is provided by us, we
can provide a payload instead to hijack the request url and call any api with
using the admin secret authorization.

## Exploit

The first thing I did was get my uuid, which can be found in the storage
portion of the developer tools.

In this case it was: 4d2e5bb1-dad3-4cf5-b41d-deee50032f63.


In order to bypass the queue for our uuid we want our exploit attatched url to be: 
`/api/4d2e5bb1-dad3-4cf5-b41d-deee50032f63/bypass` or equivalent.

However the input is run through the following filter to make sure that it is
actually a uuid.

```js
let uuid;
try {
    uuid = req.body.uuid;
} catch {
    res.redirect(process.env.QUEUE_SERVER_URL);
    return;
}

if (uuid.length != 36) {
    res.redirect(process.env.QUEUE_SERVER_URL);
    return;
}
for (const c of uuid) {
    if (!/[-a-f0-9]/.test(c)) {
        res.redirect(process.env.QUEUE_SERVER_URL);
        return;
    }
}
```

If we input the '4d2e5bb1-dad3-4cf5-b41d-deee50032f63/bypass' we will be
redirected because our input is longer than 36 characters, and it contains
non-uuid characters such as /.

Since javascript uses duck-typing, the .length function will run on arrays, not
just strings. Because of this, we can provide an array of of length 36 to
bypass the length requirement.

This is possible because the uuid is coerced into a string by implicitly
calling the toString() function on our input. Since our input is an array, it
is coerced to the elements spearated by commas.

The code for the input substitution is shown below.
```const requestUrl = `http://queue:${process.env.QUEUE_SERVER_PORT}/api/${uuid}/status`;```

In order to turn this into our target url, we can provide an array of length 36
where the first array is the payload, and the last 35 characters are arbitrary
uuid characters. To discard the remaining part of the url, we can add
a ? character to the end so the rest of the string is interpreted as
a parameter.

Since the javascript regex test function checks if any character in a given
string contains uuid characters, we just need to make sure that each element of
our array contains at least 1 uuid character.

We can then send this request to the flagserver via a python script.

```py
#!/usr/bin/python3

import requests

payload = ['4d2e5bb1-dad3-4cf5-b41d-deee50032f63/bypass?', *(['a']*35)]
ret = requests.post('http://localhost:3000/', data={'uuid': payload})
print(ret.text)
```

(In this case I was running the flag server on localhost:3000)

After running this script, we can revisit the original site, which allows us to
skip the line and get the flag.

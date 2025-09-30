# Introduction
---
---
- **CTF Name** : TryHackMe
- **Challenge Name** : Crypto Failures
- **Date** : 17/09/25 10:37:15
- **Difficulty Level** : Medium
- **Tags** : #crypto , #reversing , #bruteforce 
- **Description** : Implementing your own military-grade encryption is usually not the best idea.

![](/assets/crypto-failure/challengeLogo.png)


# Procedure
---
---

## Main challenge
- This challenge is mainly the crypto challenge, i started with port scan and i found 2 open ports: 22, 8888. 
- A normal looking web server was running on the port 8888. 
![](/assets/crypto-failure/Pasted_image_20250919094449.png)

- i started a directory scan and i found few php files only but there is a `.bak` file which means may be the source code is available for us.
![](/assets/crypto-failure/Pasted_image_20250919094546.png)

- Indeed It was the php source code.

```php
<?php
include('config.php');

function generate_cookie($user,$ENC_SECRET_KEY) {
    $SALT=generatesalt(2);
    
    $secure_cookie_string = $user.":".$_SERVER['HTTP_USER_AGENT'].":".$ENC_SECRET_KEY;

    $secure_cookie = make_secure_cookie($secure_cookie_string,$SALT);

    setcookie("secure_cookie",$secure_cookie,time()+3600,'/','',false); 
    setcookie("user","$user",time()+3600,'/','',false);
}

function cryptstring($what,$SALT){
return crypt($what,$SALT);
}

function make_secure_cookie($text,$SALT) {
$secure_cookie='';
foreach ( str_split($text,8) as $el ) {
    $secure_cookie .= cryptstring($el,$SALT);
}
return($secure_cookie);
}

function generatesalt($n) {
$randomString='';
$characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
for ($i = 0; $i < $n; $i++) {
    $index = rand(0, strlen($characters) - 1);
    $randomString .= $characters[$index];
}
return $randomString;
}

function verify_cookie($ENC_SECRET_KEY){
    $crypted_cookie=$_COOKIE['secure_cookie'];
    $user=$_COOKIE['user'];
    $string=$user.":".$_SERVER['HTTP_USER_AGENT'].":".$ENC_SECRET_KEY;

    $salt=substr($_COOKIE['secure_cookie'],0,2);

    if(make_secure_cookie($string,$salt)===$crypted_cookie) {
        return true;
    } else {
        return false;
    }
}

if ( isset($_COOKIE['secure_cookie']) && isset($_COOKIE['user']))  {
    $user=$_COOKIE['user'];
    if (verify_cookie($ENC_SECRET_KEY)) {
    if ($user === "admin") {
        echo 'congrats: ******flag here******. Now I want the key.';
            } else {
        $length=strlen($_SERVER['HTTP_USER_AGENT']);
        print "<p>You are logged in as " . $user . ":" . str_repeat("*", $length) . "\n";
	    print "<p>SSO cookie is protected with traditional military grade en<b>crypt</b>ion\n";    
    }
} else { 
    print "<p>You are not logged in\n";
}
}
  else {
    generate_cookie('guest',$ENC_SECRET_KEY);
    header('Location: /');
}
?>
```

- Analyzing it with my dead brain cells, i got that it is doing a few things.
1. its creating a encrypted cookie called secure_cookie `$user.":".$_SERVER['HTTP_USER_AGENT'].":".$ENC_SECRET_KEY;` using an username 'guest' and attacker controlled user-agent and a secret key. it concats these things together and generates a random 2 character (alnum) salt and sends them to this function `make_secure_cookie`.
2. The `make_secure_cookie` takes our string and salt. it than splits our string into 8 characters chunks and encryptes each 8 character substrings with php's `crypt($str,$salt)` function with that same salt we generated. it concatinates and sends back the whole encrypted string.
3. Now if we have that encrypted cookie set and we send a request to `/`. it validates our encrypted cookie by generating a new cookie using our user cookie (that contains a username in plaintext ) and user-agent and the server's secret key and than matching it with the original secure_cookie it received.
4. For the first flag, we have to change the username to admin in both user cookie and secure_cookie cookie. 
5. For the Second flag, we have to get the server secret key that is being used in the cookie encryption and validation.
6. Enough Analysis ! let's jump in...

## Changing the Username in Cookie
- we know that the encrypted cookie is just a concatinated string of mutliple encrypted chunks of 8 characters. if we change the first 8 character block. it won't affect others. but we need to make sure the validation doesn't break. let's understand how the `crypt()` func works in php.
![](/assets/crypto-failure/Pasted_image_20250919102409.png)
![](/assets/crypto-failure/Pasted_image_20250919102337.png)

- now we know that its not an encryption function, instead its a password hashing function and importantly. its hashing process depends on the length and type of salt provided. in our php source code, we show that it always uses 2 character random salt. so the process is like this.
>`crypt("8 chars strings","2 chars salt")` -> `STD_DES` -> `2 chars salt + 11 characters encrypted value` -> `13 chars output`
>
>![](/assets/crypto-failure/Pasted_image_20250919103051.png)
- now if you look carefully at the cookie and break it in 13 chars chunks. you can see that it always starts with `U4` becuz that's the salt that was used.
- now we know everything to get the first flag. we have the salt, we know the cookie was made using `guest:myuseragent:secretkey`. my user agent is Mozilla/5.0 something.
- i created this simple php script to get the hash of `admin:Mo` cause this is the first 8 characters.

```php
<?php
$firstArg = $argv[1];
$salt = $argv[2];
function cryptstring($what,$SALT){
return crypt($what,$SALT);
}

echo cryptstring($firstArg,$salt);

?>
```
![](/assets/crypto-failure/Pasted_image_20250919103621.png)
- i used the same salt as our cookie has and changed the first chunk of our cookie. than i changed the user cookie to `admin`.

![](/assets/crypto-failure/Pasted_image_20250919103603.png)
- It worked and i got the first flag. it was the easy part now we need to get the secret key used in cookie generation.

## Using Byte-at-a-time oracle attack
- now to get the secret key of the server, we can use a technique called byte-at-a-time oracle cause we can control the user-agent value and its length. let's understand it.

```css
1. we can set a user agent to "AAAAAAA" and send a request without a cookie.
2. it will make the cookie generate like "guest:AAAAAAA:secretkey" with than breaks into 8 - 8 bytes chunks.
3. so the server will encrypt the cookie like this.
	- first chunk       = "guest:AA" = some random 13 chars
	- second chunk = "AAAAAA:?" = some random 13 chars
4. here we know the second chunk has the first character of the secret key. now we can use our php script which i created to try all possible values at the last position and whichever output matches the second chunk of our cookie will be the first character of the secret key.
5. this is how we can reduce the number of "A" and know few chars of the secret key. 
6. the problem here was getting the next chunks character and so for that we have to use the already known secret key value and use it in our php script to get characters via the 8 bytes block oracle.
```

- I created this python script to automate this and it took a lot of time. the reason, i was a single `:` cause the cookie generation was including a colon after our user-agent. it made me think like we can only retrieve 7 character from one block instead of 8. that created a gap of 1 character in each block and after trying for a long time. i solved the problem created by myself lol. feeling dumb and smart at the same time.
- so aftering getting the flag and hitting my head on keyboard. i thought, we could think the `:` as a part of flag and go with the simple `byte-at-a-time oracle` technique. we could easily get the flag and without needing to add 1 extra byte at each chunk. you will get what i mean once you read my python script.

```python
import os
import requests
import urllib
import subprocess

"""
i can't explain it enough, its just working ;_;
"""

URL = "http://10.201.50.77/index.php"
charset = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_-{}`~!@#$%^&*()+=[]\\|:;\'<>,.?/';
def getNewCookie(userAgent):
	while True:
		header = {"User-Agent":userAgent}
		res = requests.get(URL,allow_redirects=False,headers=header)
		decoded_res = urllib.parse.unquote(res.headers["Set-Cookie"])
		secure_cookie = decoded_res.split("=")[1].split(";")[0]
		salt = secure_cookie[:2]
		chunks = secure_cookie.split(salt)[1:]
		chunks2 = list(filter(lambda x:len(x)==11,chunks))
		if len(chunks) == len(chunks2):
			break
	final_chunks = list(map(lambda x: salt + x, chunks))
	print(final_chunks)
	return final_chunks

def blockOracle():
	flag = ''
	Block_Size = 7 # cause of colon appended
	Block_len = 6 # gonna increase it
	userAgent = "AA"
	previous_chars = ""
	counter = 0
	current_block = 1
	found = ""
	while True:
		found = ""
		for i in range(Block_Size):
			suffix = "A" * ((Block_len + counter) - i)
			secure_cookie = getNewCookie(userAgent+suffix)
			chunk = secure_cookie[current_block]
			if current_block == 1:
				previous_chars = suffix + ":"
			for char in charset:
				if current_block == 1:
					cmd_text = f"php experiment.php {previous_chars + found + char} {chunk[:2]}"
				else:
					cmd_text = f"php experiment.php '{previous_chars[i:] + found + char}' '{chunk[:2]}'"
				output = subprocess.run(cmd_text, capture_output=True, check=True, text=True, shell=True)
				output = output.stdout
				print(f"trying with char {char} and ouput is {output} and chunk is {chunk}")
				if output == chunk:
					flag += char
					found += char
					print(f"Found the char {char}")
					print(f"Found the flag {flag}")
					break
		previous_chars = found	
		current_block += 1
		counter += 1

blockOracle()
```

- it will run whole night and eventually, you will see something like this, magic of crypto : `THM{Traditional_Own_Crypto_is_Always_Surprising!_and_this_hopefully_is_not_easy_to_crack_e41d20b5b0989cac65ed4a090cace944bf30e6d3ab88f9d447f52fd2140525b9}`

# Conclusion
---
---
- **Summary of The Challenge** : it was not much hard and not so easy. A perfect medium crypto challenge. it can be easy for crypto guys but for me, its kinda medium and pretty amazing challenge.
- **What i Learned** : byte-at-a-time oracle attack, understanding crypto implementation and working.
- **Tools Used** : feroxbuster, deadscan, burpsuite, python3, etc.

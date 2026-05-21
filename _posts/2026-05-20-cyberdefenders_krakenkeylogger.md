---
layout: page
title: KrakenKeylogger
permalink: /cyberdefenders_krakenkeylogger/
---

Hello,
Today I am going to solve KrakenKeylogger lab at cyberdefenders site.

<strong>1.What is the the web messaging app the employee used to talk to the attacker?</strong>

At the begining I was searching for any app installed in Program Files, later seraching also in different directories but I wasn't able to find anything. Then I found out about wpndatabase.db.

wpndatabase.db is a local database file in Windows 10 and Windows 11 that stores system and application notifications. It belongs to the Windows Push Notification (WPN) service.

File location is <strong>Users/OMEN/AppData/Local/Microsoft/Windows/Notifications/wpndatabase.db</strong>

You can open it with sqlitebrowser

<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/wpndatabase_db.webp' | relative_url }}"
    alt="WPN database"
    width="900">

  <figcaption>
    SQLite database containing notification artifacts.
  </figcaption>
</figure>

When you search throughout all records in you can find 

<pre>
&lt;toast launch=&quot;0|0|Default|0|https://web.[REDACTED].org/|p#https://web.[REDACTED].org/#&quot; displayTimestamp=&quot;2023-07-11T16:57:15Z&quot;&gt;
 &lt;visual&gt;
  &lt;binding template=&quot;ToastGeneric&quot;&gt;
   &lt;text&gt;Nawaf&lt;/text&gt;
   &lt;text&gt;[REDACTED]lt;/text&gt;
   &lt;text placement=&quot;attribution&quot;&gt;web.[REDACTED].org&lt;/text&gt;
   &lt;image placement=&quot;appLogoOverride&quot; src=&quot;C:\Users\OMEN\AppData\Local\Google\Chrome\User Data\Notification Resources\cd18935b-57e3-4838-b5e3-ef360362f771.tmp&quot; hint-crop=&quot;none&quot;/&gt;
  &lt;/binding&gt;
 &lt;/visual&gt;
 &lt;actions&gt;
  &lt;action content=&quot;[REDACTED]#&quot;/&gt;
 &lt;/actions&gt;
&lt;/toast&gt;
</pre>
It becomes clear what app is used.

<strong>2. What is the password for the protected ZIP file sent by the attacker to the employee?</strong>

Have a closer look into toast message. It contains password.

<strong>3. What domain did the attacker use to download the second stage of the malware?</strong>

So in toast message you have a hint that zip file was downloaded. Let's find it by

<pre>find . -name '*.zip'</pre>

What you should find is: 
<pre>./Users/OMEN/Downloads/project templet test.zip</pre>

When you got to that folder you realize it was already unziped. Go to that folder and observe template.lnk file.

It was tricky for me because usually you would use lecmd under windows however on linux you need to instal dotnet env and lecmd. I did it but for some reason some data was truncated. So I searched and realized exiftool can be used to check what is inside.


<figure>
  <img
    src="{{ '/assets/images/posts/krakenkeylogger/template_data.webp' | relative_url }}"
    alt="Template data"
    width="900">

  <figcaption>
    Obfuscated powershell script in template.lnk
  </figcaption>
</figure>

I used ChatGpt to write me same logic in python code and decode url (it's in one of variables in powershell script).

<pre>

def decode_string(s: str) -> str:
    """
    Replicates the PowerShell string deobfuscation logic.
    """

    # Reverse whole string
    s = s[::-1]

    # Reverse every 2-character chunk
    decoded = "".join(
        s[i:i+2][::-1]
        for i in range(0, len(s), 2)
    )

    return decoded


if __name__ == "__main__":

    encoded = 'aht1.sen/hi/coucys.erstmaofershma//s:tpht'

    decoded = decode_string(encoded)

    print("\nDecoded:")
    print(decoded)
</pre>

Script prints url used by attacker






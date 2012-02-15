This project aims to document the protocol for uploading music files to Google Music. This project is related to the [MusicAlpha](https://github.com/antimatter15/musicalpha) project as well as the [Unofficial-Google-Music-API](https://github.com/simon-weber/Unofficial-Google-Music-API). Since there is no official documentation, much of the process involves guesswork, but the process is largely functional.


##Overview

First is the process of logging in. This login process is surprisingly simple and involves a simple HTTPS POST request, and the response is just as simple: cookie values separated by newlines. This must be conducted over HTTPS, or else the response simply doesn't work. Also, the values seem equivalent to browser login cookies (the only one which is needed is the SID, everything else can pretty much be discarded), as in, a SID attained through the web based login is just as valid as one procured through this process. Note that an SID alone is not sufficient to authenticate to the web client, however; there is a process for upgrading the session, but it is so far undocumented.

The next steps are done over SSL with Google's Protocol Buffers, a binary data format. Also, Music Manager is configured to ensure it is communicating with Google's certificates; the executable had to be patched before Fiddler could be used to intercept the communications.

The protobuff step is uploader authentication. It involves sending the NIC's MAC address and the computer's hostname to Google's servers, which in return, respond with some protobuf encoded status message consisting of indecipherable numbers. Google restricts users to "up to 10 devices" (these can be managed from the web interface in the [settings page](https://music.google.com/music/listen?u=0#settings_pl)). This is also why Google Music isn't supported on virtual machines: the NIC usually has a default MAC which someone else already registered (change your virtual NIC MAC to get around this).

Once you have authenticated, you can query for things like the user's current quota status. That current quota status includes the maximum number of files of that current payment level, the total number of uploaded files and the available tracks.

Now the actual upload process can begin. For each song to upload, the client sends select song metadata and a hash (called the ClientId) for each file. Google checks the hash against what is already in the library, and will actively reject duplicate requests. Note that the client is trusted to calculate the correct hash - spoofing the hash will allow duplicate uploads. Accepted requests are given a ServerId to use - these are the same that identify the songs in the library later.

Tracks are initialized in bulk since there is a period of time which must be waited before proceeding to the next step. This seems to be about 3 seconds after the server has responded with ServerIds, presumably for the uploadsj and android servers to sync up.

Once that duration has elapsed, the client opens an unencrypted HTTP channel to Google Music and POSTs some JSON, repeating some audio information (file name, bitrate), including some useless things and notably, including the ServerID. The server responds with more JSON, including the url to PUT the file at.

After the PUT is finished, the song is uploaded.


##Authentication

This is the first thing the client does.

```
POST https://www.google.com/accounts/ClientLogin HTTP/1.1
User-Agent: Music Manager (1, 0, 24, 7712 - Windows)
Host: www.google.com
Accept: */*
Content-Type: application/x-www-form-urlencoded
Content-Length: 76
```

It's a POST request done over HTTPS with this content

```
Email=username%40gmail.com&Passwd=thisisntactuallymypassword&service=sj&accountType=GOOGLE
```

Also an interesting tidbit, the `service=sj` thing. At first I thought it was stood for steve jobs, but that wouldn't make too much sense. I had it stuck in the back of my mind for a bit and then it struck me while looking at the uploadsj part of the URL. I think it's short for "skyjam", the internal code name for Google Music.

The server thinks it's okay and then says

```
HTTP/1.1 200 OK
Content-Type: text/plain
Cache-control: no-cache, no-store
Pragma: no-cache
Expires: Mon, 01-Jan-1990 00:00:00 GMT
Date: Tue, 31 Jan 2012 18:55:59 GMT
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Content-Length: 881
Server: GSE
```

It responds with the SID=, LSID=, Auth= cookie values delimited by newlines.

```
SID=DQAAREDACTEDkV3izSm
LSID=DQAAAMMREDACTEDLLorHpA
Auth=DQAAREDACTEDep1i-o
```

There's also a trailing newline, if that matters. The SID is all that is needed for uploading; the other cookies are used when bumping auth to be used with the web client.

Also, omitting the GOOGLE and sj parts seems to return something without an auth token. It's not yet clear if that resultant SID is still valid.

##/upsj/upauth

`POST https://android.clients.google.com/upsj/upauth HTTP/1.1`

This is the first thing that a computer does once the authentication token has been generated. However, it is likely that this process only needs to be executed once. The purpose of this step seems to register the current client as a "device" from which Google Music content can be managed. Google places a restriction such that the Music library can only be accessed from "up to ten devices", each of which can be manually deauthorized from the settings page of the web interface. It is this restriction that also seems to be the reason Google does not allow the Music Manager software to run on Virtual Machines, for they each use stock MAC addresses that fail to differentiate between installations.

The contents of that POST request are reproduced below, decoded with our `.proto` file which may or may not reflect the original internal one. As you can see, there are two string fields named the "address" and "hostname", respectively. The address is the device's MAC address, and represents a unique identifier for a client. The hostname is the computer's name, and is used as user-facing text to identify the device. In later stages of the upload process (The JSON-encoded `/uploadsj/rupio` part), the address is also known as an `UploaderID`. From the web interface, the `/music/services/loadsettings` request refers to the address and hostname as `id` and `name`, respectively.

```
address: "02:11:DD:3B:1A:3A"
hostname: "my-pc"
```

The response consists of a series of numbers which are yet to be interpreted in any significant way. However, the contents do not seem to deviate from the values depicted below.

```
1: 6
6 {
  1: 0
  2: 0
  3: 5
  4: 6000
  5: 0
  6: 3000
}
11: 8
12: 10
```

However, their values do not seem to play any role in any subsequent steps, so they can be safely discarded.

##/upsj/clientstate

`POST https://android.clients.google.com/upsj/clientstate HTTP/1.1`

This doesn't matter nearly as much as the last stage, and the upload should be able to commence regardless of whether or not you send this message. All it does is give an update with regard to your quota details, ie. how many songs you can upload, how many you have, etc.

The server responds with that same series of cryptic numbers denoting state as well as what I believe to be the number current upload quota state.

```
8 {
  1: 20000 //maximum number of songs that can be held for your current payment plan
  2: 1696 //total number of songs you have uploaded?
  3: 1402 //available songs (songs able to be played by the web client?)
}
```

As Simon Weber noted, it seems to correspond with the `/music/services/getstatus` json request.

```
"availableTracks":1671,
"totalTracks":1723
```

##/upsj/metadata?version=1

This step sends metadata to Google, and requests that these files be given an upload session.

The other important part of this step is the ClientId. This is a hash of the file, calculated by taking an md5 sum of the file, stripped of tags, then encoded with base64. The padding "===" from the result is discarded, leaving 22 characters. 

The process of discarding tags is not yet fully understood. It seems that ID3 tags at the beginning and/or end of the file will always be removed. However, APE tags at the end of the file have been left on while hashing; this is likely just an oversight on Google's end.

The ClientId is used to check for duplicate files in the library, but _not_ across other's libraries. Tests have shown that duplicate ClientIds are no problem across different accounts, so a semi-compliant implementation of the hash (eg not removing tags first) should work fine in practice.


`POST` to https://android.clients.google.com/upsj/metadata?version=1 

```
1 {
  2: "lJE0p7HzgoeiBToyLpAoIA" #ClientId
  3: 1271984386 #04 / 22 / 10 @ 7:59:46pm EST - Probably Creation Date
  4: 1272400853 #04 / 27 / 10 @ 3:40:53pm EST - Probably Last Played Date
  6: "Carl Sagan - Glorious Dawn (ft Stephen Hawking)" #Title, this is quite self explanatory
  7: "Colorpulse" #Artist, TPE1
  8: ""
  9: ""
  10: ""
  11: 0
  12: ""
  13: 0
  14: "Other" #Genre
  15: 213000 #duration of the file in milliseconds
  16: 0
  20: 0
  26: 0
  27: 0
  28: 0
  31: 1
  32: 5115029 #no idea what this is
  37: 0
  38: 0
  44: 192 #bitrate, it seems
  53: "-383260437"
  61: 1
}

2: "00:1E:EC:6F:49:3\n"
```


Here is a request. Note that for a duplicate ClientId, the server will send back the serverId of the duplicate file.

```
u0: 1
response {
  ids: "lJE0p7HzgoeiBToyLpAoIA"
  uploads {
    id: "lJE0p7HzgoeiBToyLpAoIA"
    u0: 4
    serverId: "31c4330d-b519-368d-9fd0-b800d2bde10c"
  }
}
state {
  u0: 0
  u1: 0
  u2: 5
  u3: 6000
  u4: 0
  u5: 3000
}
```


#/uploadsj/rupio

This is the response to a successful request:

```
{
  "sessionStatus": {
    "state": "FINALIZED",
    "externalFieldTransfers": [
      {
        "name": "magic.mp3",
        "status": "COMPLETED",
        "bytesTransferred": 481237,
        "bytesTotal": 481237,
        "putInfo": {
          "url": "http://uploadsj.clients.google.com/uploadsj/rupio?upload_id=AEnB2UREDACTED-adSnQ&file_id=000"
        },
        "content_type": "audio/mpeg"
      }
    ],
    "additionalInfo": {
      "uploader_service.GoogleRupioAdditionalInfo": {
        "completionInfo": {
          "status": "SUCCESS",
          "customerSpecificInfo": {
            "ServerFileReference": "011a1REDACTEDaf881-19",
            "ResponseCode": 200
          }
        }
      }
    },
    "upload_id": "AEnB2REDACTEDh-adSnQ"
  }
}
```


##/upsj/uploadstate?version=1

`POST https://android.clients.google.com/upsj/uploadstate?version=1 HTTP/1.1`

Here's the protobuf-decoded thing, containing the uploader ID and something else which I have no idea about.

```
1: 1
2: "00:1F:3A:6D:42:21"
```

In return, the server returns more gobblygoop.

```
1: 8
6 {
  1: 0
  2: 0
  3: 5
  4: 6000
  5: 0
  6: 3000
}
11: 10
```


##/upsj/getjobs?version=1

`POST https://android.clients.google.com/upsj/getjobs?version=1 HTTP/1.1`

This seems to retrieve a list of upload jobs. It takes the Uploader ID.

```
1: "00:1F:3A:6D:42:21"
```

In return, the server responds with:

```
1: 5
6 {
  1: 0
  2: 0
  3: 5
  4: 6000
  5: 0
  6: 3000
}
7 {
  1 {
    1: "2xgHehGNCIrgHcBbiFnDhg"
    2: "3ed1f828-b44d-33aa-b11c-8986b8232344"
    5: 4
  }
  ...
  1 {
    1: "AlIp3uQMfEVwXBtKZr+gvA"
    2: "c5fb4b55-834a-3c51-ad67-acac072c201a"
    5: 4
  }
  2: 10
}
```

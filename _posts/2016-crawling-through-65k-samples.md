---
layout: post
title: Crawling through ~65k malware samples
categories:
- blog
---



...

To understand certain aspects of the log output, it's important to know a few key things about Laika BOSS
* It only identifies files it knows about
* Modules only run on relevant files
* It is loaded with a default set of sample Yara rules

First, and most importantly, it can only identify files it knows about. As each file is parsed by the tool, if file signature metadata is attached to a file object, then that file object will be explicitly identified with the file type. This file signature metadata is found in the field 'fileType[]'. File signature metadata is primarily attached via Yara rules; if a Yara rule for a particular file object either doesn't exist or doesn't append the file metadata to that object, then the file object will not have a defined file signature. As of now, there are 31 file signatures recognized by the tool. For reference, they are copied below
```
file_type = "pem"
file_type = "der"
file_type = "crt"
file_type = "pkcs7"
file_type = "pe"
file_type = "zip"
file_type = "rar"
file_type = "cab"
file_type = "tar"
file_type = "arj"
file_type = "ole"
file_type = "officex"
file_type = "rtf"
file_type = "pdf"
file_type = "chm"
file_type = "hlp"
file_type = "wri"
file_type = "lnk"
file_type = "class"
file_type = "eml"
file_type = "mime"
file_type = "tnef"
file_type = "fws"
file_type = "cws"
file_type = "zws"
file_type = "swf"
file_type = "tiff"
file_type = "mp3"
file_type = "wmv"
file_type = "avi"
file_type = "mov"
```

Second, many modules only run on relevant files-- this is typically determined by the file signature of each scanned object. This means that the tool will only run the PE metadata module on PE files, the ZIP explosion module on ZIP files, etc. There are some instances where modules are run on all file objects regardless of file signature. In the original Laika BOSS repository, these are the Yara scanning and file hashing modules. For my repository, I've added a textual histogram module.

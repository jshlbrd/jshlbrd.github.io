---
layout: post
title: Crawling through ~65k malware samples
categories:
- blog
---

Recently I mentioned on Twitter that I was interested in taking the VXShare torrent malware samples, running them through Lockheed Martin's Laika BOSS, and sharing the results. That project is still a work in progress, but I thought I would share some analysis of one of the VXShare torrent datasets.

If you aren't familiar with Laika BOSS, you can access the repo here: [https://github.com/lmco/laikaboss](https://github.com/lmco/laikaboss)

Note that I'm using my fork of the Laika BOSS repository, which includes many modules that are not implemented in the source repository. Most of these modules can be accessed here: [https://github.com/jshlbrd/laikaboss-modules](https://github.com/jshlbrd/laikaboss-modules)

## TLDR: WTF is Laika BOSS?

Laika BOSS is a file-scanning tool built on Python and Yara that produces extensive file metadata and file alerts. Since it's built on Python, it's relatively easy to expand its capabilities. The combination of extensive file metadata and easy expandability makes it a great malware hunting tool.

## Understanding the data

To best understand this blog post, it's important to know a couple key things about Laika BOSS

* It only identifies files it knows about
* Modules only run on relevant files

First, and most importantly, it can only identify files it knows about. As each file object is parsed by the tool, it is checked for file signature metadata; if file signature metadata is found, then a file type is explicitly assigned to the object. The file type is found in the field 'fileType[]' and is primarily attached via Yara signatures; if a Yara signature for a particular file object doesn't exist or if the signature doesn't append the file type to an object, then the object will not have a defined file type. As of now, there are 31 file signatures recognized by the tool. For reference, they are copied below.

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

Second, many modules only run on relevant files-- this is typically determined by the file type of each scanned object. This means that the tool will only run the PE metadata module on PE files, the ZIP explosion module on ZIP files, etc. There are some instances where modules are run on all file objects regardless of file type. In the source Laika BOSS repository, these are the Yara scanning and file hashing modules. For my repository, I've added a textual histogram module.

If you're still with me, then let's look at some of the data!

## Running through the data

This dataset contains files from VXShare torrent 220, which contains 65,536 of unorganized files. One file was not parsed correctly, so the Laika BOSS log data contains metadata for 65,535 files. Additionally, this analysis relied on converting the log data to a JSON object in a Python script. There were 841 samples that couldn't be parsed, so they were excluded from analysis.

With all that, the resulting dataset includes 64,694 files.

Here's some file size metadata for the compressed VXShare samples and the resulting Laika BOSS output.

```
9920MB VirusShare_00220.zip
598MB VirusShare_00220.laika
164MB VirusShare_00220.laika.zip
```

The VXShare samples are 9920MB (~9.9GB) compressed and the resulting uncompressed Laika BOSS log file is 598MB-- that's a lot of useful file metadata at 6% of the original sample size! Compressing the log reduces its size a considerable amount, 164MB or 1.6% of the original sample size.

Here's how many files are in the dataset.

```
64694 root file objects
197642 total file objects
```

In the context of this blog post, a 'root' file object is one that was not exploded from another file object. The dataset began with 64,694 files; however, the total file object count is 197,642. How did this happen? Laika BOSS processed the root file objects and identified child objects within them, which were then dispatched through the system. The total file object count includes the ~65k root file objects, and all exploded file objects that originates from those root files.

Let's look at how many root file objects Laika BOSS successfully identified. To do this, we're looking for every file object that has a depth of zero (meaning the object has not been exploded and therefore is a root object).

Here's a breakdown of each file type in the dataset, by count and percentage.

```
56326 0.85948 empty
7471 0.11400 pe
474 0.00723 rar
404 0.00616 zip
143 0.00218 jar
16 0.00024 cab
8 0.00012 ole
7 0.00011 swf
6 0.00009 cws
1 0.00002 mime
1 0.00002 officex
1 0.00002 pdf
1 0.00002 zws
```

The ratio of unidentified file objects to identified file objects is extremely high, so high that it makes the distribution of file types difficult to judge. Kind of makes you wonder, what kinds of samples are people uploading to VXShare?! Here's the same dataset with the empty file type stripped out, shown as a percentage of file objects identified.

```
0.81127 pe
0.05147 rar
0.04387 zip
0.01553 jar
0.00174 cab
0.00087 ole
0.00076 swf
0.00065 cws
0.00011 mime
0.00011 officex
0.00011 pdf
0.00011 zws
```

With the empty file objects removed, the distribution is easier to understand. Not surprisingly, PE files make up the majority of identifiable file objects in the dataset. Personally, I was disappointed to see a low representation of files usually encountered in phishing emails-- ole, officex, and pdf file types.

Here's a similar breakdown for all file objects, root and children.

```
172405 0.87 empty
8698 0.04 pe
5456 0.03 crt
5453 0.03 der
5451 0.03 pkcs7
4236 0.02 class
717 0.00 zip
498 0.00 rar
155 0.00 jar
57 0.00 ole
49 0.00 hlp
34 0.00 swf
27 0.00 cab
18 0.00 fws
15 0.00 cws
11 0.00 lnk
5 0.00 pdf
5 0.00 officex
5 0.00 mov
5 0.00 chm
3 0.00 pem
2 0.00 wmv
2 0.00 tar
2 0.00 rtf
1 0.00 zws
1 0.00 mp3
1 0.00 mime
```

Again, the percentage with empty file objects removed.

```
0.34 pe
0.22 pkcs7
0.22 der
0.22 crt
0.17 class
0.03 zip
0.02 rar
0.01 jar
0.00 zws
0.00 wmv
0.00 tar
0.00 swf
0.00 rtf
0.00 pem
0.00 pdf
0.00 ole
0.00 officex
0.00 mp3
0.00 mov
0.00 mime
0.00 lnk
0.00 hlp
0.00 fws
0.00 cws
0.00 chm
0.00 cab
```

The distribution heavily favors empty file objects and PE files. Interestingly, new file types are introduced when looking at all file objects-- pkcs7 and class file objects make up a large portion of the dataset.

Here's a deeper look at the exploded file objects, by count, file type, and the module that exploded the file object.

```
58043 empty EXPLODE_ZIP
46015 empty META_PE
10126 empty EXPLODE_RAR
5448 der EXPLODE_PKCS7 [1]
5448 crt EXPLODE_PKCS7 [1]
5242 pkcs7 META_PE [1]
4236 class EXPLODE_ZIP [3]
1681 empty EXPLODE_OLE
1032 pe EXPLODE_RAR
303 zip EXPLODE_ZIP [2]
209 pkcs7 EXPLODE_ZIP
209 empty EXPLODE_PDF
195 pe EXPLODE_ZIP
49 hlp META_PE
34 ole EXPLODE_RAR
24 rar EXPLODE_RAR
15 ole EXPLODE_ZIP
12 jar EXPLODE_ZIP [3]
11 swf EXPLODE_SWF
11 lnk EXPLODE_RAR
11 fws EXPLODE_SWF
10 zip EXPLODE_RAR
9 swf EXPLODE_RAR
9 cab EXPLODE_RAR
8 crt EXPLODE_ZIP
7 swf EXPLODE_ZIP
7 fws EXPLODE_RAR
7 cws EXPLODE_ZIP
5 der EXPLODE_ZIP
5 chm EXPLODE_RAR
4 mov EXPLODE_ZIP
3 pem EXPLODE_ZIP
2 tar EXPLODE_ZIP
2 pdf EXPLODE_ZIP
2 pdf EXPLODE_RAR
2 officex EXPLODE_ZIP
2 empty EXPLODE_RTF
2 empty EXPLODE_PKCS7
2 cws EXPLODE_RAR
1 wmv EXPLODE_ZIP
1 wmv EXPLODE_RAR
1 rtf EXPLODE_ZIP
1 rtf EXPLODE_RAR
1 officex EXPLODE_RTF [4]
1 officex EXPLODE_RAR
1 mp3 EXPLODE_ZIP
1 mov META_PE
1 empty EXPLODE_EMAIL
1 cab EXPLODE_ZIP
1 cab EXPLODE_OLE
```

Now, these results are a bit more interesting! There are many ways to interpret this data, but here's my immediate reaction:

* The dominate child object appears to be digital signatures exploded from PE files [1]
* There may be a zip bomb in the dataset [2]
* There is a high ratio of Java class files to JAR files [3]
* The officex file found in one of the RTF files is an interesting outlier [4]

The second observation leads to an interesting thought-- how would you identify a zip bomb in the dataset? One way is to look at the distribution of file object depth.

```
110551 1
64694 0
20644 2
1729 3
24 4
```

The greatest file object depth is 4-- meaning that the likelihood of their being a zip or any other compression bomb in the dataset is low. This could be further verified by running similar analytics on only zip files.

Now, we've learned a lot about the dataset while barely scratching the surface of the metadata contained within it. To finish this post, I'll share some simple, module-specific results that may pique your interest.

Since there are a lot of PE files, we'll look at those first. I wonder how many imphashes appear in them?

```
1034 a8286b574ff850cd002ea6282d15aa40
657 be3154a7cc7ab2b0631b9c25c67b9da0
518 7fa974366048f9c551ef45714595665e
332 8ffc31bccd11f7f873be952d93bdc291
318 561178c1feba20d211c82b55ebe80883
193 cce52e446d06e1e9faf34bd0e686462a
183 099c0646ea7282d232219f8807883be0
165 884310b1928934402ea6fec1dbd3cf5e
152
129 d524f1ae55f37f3df54f67a58d24d838
...
1 014fc544a80298571ec66694ff31b27e
1 013fb0f525502cd2d3af38d41cfda66c
1 00d96cbdae68312fa839b9f65f76e19e
1 00d6958894c654bf26b2ae0e5710b5c9
1 00c5fd00087020a0645079ce30f4148b
1 00bc1df626804ffae57a2a7a7968fbf6
1 00b9faa2e237d528e17ba35c19620895
1 007daffc89d9f0e602bbd98cac070cf4
1 0052d6c6cb42e6e310bbd010022ca7b4
1 00043da75f4cf67f17ef9a8d7e0d5bac
```

A lot! There were 2,445 imphashes, far too many to include on this page. ~86% of them were unique and were only seen once. The results above show that, at the high end, one imphash appeared over 1,000 times. A cursory look on Virus Total suggests that the imphash may be related to adware / spyware.

What about the distribution of language codes in the PE files?

```
2223 0000
841 0409
641 0804
471 0809
179 0C0A
79 0419
13 0009
10 0C07
6 0412
4 0407
3 0413
2 1009
2 0C0C
2 0C09
2 0800
2 041F
2 0416
2 0415
2 040A
2 0404
2 0401
2 0019
1 1
1 0816
1 0422
1 0411
1 0410
1 040C
1 0403
1 0400
```

The compile time on a PE file can sometimes be a useful artifact, I wonder what we'll find there?

```
1024 2013:03:12 01:51:45-07:00
664 1992:06:19 15:22:17-07:00
653 2015:08:03 02:35:07-07:00
393 2009:12:05 14:52:12-08:00
197 2014:07:23 02:35:20-07:00
173 2014:02:12 10:41:16-08:00
152 2009:12:05 14:50:46-08:00
135 2014:03:29 02:42:03-07:00
87 2015:01:01 19:45:54-08:00
66 2014:11:06 05:19:35-08:00
...
1 1987:09:10 18:35:02-07:00
1 1980:09:04 13:24:33-07:00
1 1972:06:07 08:53:38-07:00
1 1970:11:18 22:48:03-08:00
1 1970:01:08 09:36:00-08:00
1 1970:01:07 01:44:32-08:00
1 1970:01:01 10:12:16-08:00
1 1970:01:01 02:23:43-08:00
1 1969:12:31 16:00:49-08:00
1 1969:12:31 16:00:16-08:00
```

Yep, nothing suspicious. :)

Let's finish with OLE files. What's the distribution of word count in these files?

```
2 96
1 0
1 2
1 868
1 1061
1 646
1 1324
1 130
1 84
1 22
1 540
1 61
```

One file object has zero words in it-- that's probably a file worth looking into.

What about file authors?

```
2 User
1 Windows User
1 1
1 Van Asselt Adviseurs & Accountants
1 lenovo
2 kang
1 Юрий
1 Lacey Wieser
1 MC SYSTEM
1 www.diyxp.net
1 Microsoft
1 seri
```

Finally, let's look for potentially malicious macro code in the OLE files. We can use the IOC field in the META_OLEVBA module to grab domains, IP addresses, and files that are inside macro code in the OLE file object.

```
1 Urlmon.dll
1 cureall.exe
1 shell32.dll
1 amnestic.exe
1 hxxp://geobrugg[.]co[.]kr/bbs/factuur2390.exe
1 factuur2390.exe
```

I think that's where I'll leave this post. If there's any interest in this, then maybe this could become a regular series.

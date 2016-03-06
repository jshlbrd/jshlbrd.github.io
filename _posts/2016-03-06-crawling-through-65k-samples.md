---
layout: post
title: Crawling through ~65k malware samples
categories:
- blog
---

Recently I mentioned on Twitter that I was interested in taking the VXShare torrent malware samples, running them through Lockheed Martin's Laika BOSS, and sharing the results. That project is still a work in progress, but I thought I would share some analysis of one of the VXShare torrent datasets.

If you aren't familiar with Laika BOSS, you can access the repo here: https://github.com/lmco/laikaboss

Note that I'm using my fork of the Laika BOSS repository, which includes many modules that are not implemented in the source repository. Most of these modules can be accessed here: https://github.com/jshlbrd/laikaboss-modules

## TLDR: WTF is Laika BOSS?

Laika BOSS is a file-scanning tool built on Python and Yara that produces extensive file metadata and file alerts. Since it's built on Python, it's relatively easy to expand its capabilities. The combination of extensive file metadata and easy expandability makes it a great malware hunting tool.

## Understanding the data

To best understand this blog post, it's important to know a couple key things about Laika BOSS
* It only identifies files it knows about
* Modules only run on relevant files

First, and most importantly, it can only identify files it knows about. As each file object is parsed by the tool, it is checked for file signature metadata; if file signature metadata is found, then a file type is explicitly assigned to the object. The file type is found in the field 'fileType[]' and is primarily attached via Yara signatures; if a Yara signature for a particular file object either doesn't exist or doesn't append the file type to that object, then the object will not have a defined file type. As of now, there are 31 file signatures recognized by the tool. For reference, they are copied below.

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

Second, many modules only run on relevant files-- this is typically determined by the file type of each scanned object. This means that the tool will only run the PE metadata module on PE files, the ZIP explosion module on ZIP files, etc. There are some instances where modules are run on all file objects regardless of file type. In the original Laika BOSS repository, these are the Yara scanning and file hashing modules. For my repository, I've added a textual histogram module.

If you're still with me, then let's look at some of the data! Before we begin, my analysis relied on converting the log data to a JSON object in a Python script. There were ~800 samples that couldn't be parsed, so they were excluded from analysis.

## Running through the data

This dataset contains files from VXShare torrent 220, which contains 65,536 unorganized files. To begin, I was first interested in how many of these files Laika BOSS recognized. To identify this, two fields need to be checked-- depth and fileType[]. Depth represents which layer each file object exists in, and increments as file objects are exploded. Any file object with a depth of zero should be the highest level file object, meaning that's the originating VXShare sample (aka, a root file). The fileType[] field is more self-explanatory, it represents the type of file that each file object is. Finding an empty fileType[] field means that Laika BOSS doesn't have a file signature for that particular file type.

```
empty		0.85948
pe			0.11400
rar			0.00723
zip			0.00616
jar			0.00218
cab			0.00024
ole			0.00012
swf			0.00011
cws			0.00009
mime		0.00002
officex	0.00002
pdf			0.00002
zws			0.00002
```

The ratio of unidentified files to identified files is extremely high, so much that it makes the distribution of file types difficult to judge. Kind of makes you wonder, what kinds of samples are people uploading?! Here's the same dataset with the empty fileType[] stripped out.

```
pe 			0.81127
rar 		0.05147
zip 		0.04387
jar 		0.01553
cab 		0.00174
ole 		0.00087
swf 		0.00076
cws 		0.00065
mime 		0.00011
officex 0.00011
pdf 		0.00011
zws 		0.00011
```

With the empty files removed, the distribution is easier to understand. Not surprisingly, PE files make up the majority of identifiable files in the torrent, and compressed files follow. Personally, I was disappointed to see a poor representation of files usually encountered in spearphishing attacks-- namely, the ole, officex, and pdf file types.

Since we ignored any exploded file objects before, let's take a look at them now. Consider that because there are an unknown number of child file objects, we cannot compare them to the total number of root files scanned. This distribution also does not account for recursive file explosions or child object explosions-- e.g., a ZIP file may contain multiple files within it, and some of those files may contain files within themselves. That said, here are all file objects that were exploded from another file object, along with the source module that exploded them.

```
der, EXPLODE_PKCS7		5448	[1]
crt, EXPLODE_PKCS7		5448	[1]
pkcs7, META_PE				5242	[1]
class, EXPLODE_ZIP		4236	[3]
pe, EXPLODE_RAR				1032
zip, EXPLODE_ZIP			303		[2]
pkcs7, EXPLODE_ZIP		209
pe, EXPLODE_ZIP				195
hlp, META_PE					49
ole, EXPLODE_RAR			34
rar, EXPLODE_RAR			24
ole, EXPLODE_ZIP			15
jar, EXPLODE_ZIP			12		[3]
swf, EXPLODE_SWF			11
lnk, EXPLODE_RAR			11
fws, EXPLODE_SWF			11
zip, EXPLODE_RAR			10
swf, EXPLODE_RAR			9
cab, EXPLODE_RAR			9
crt, EXPLODE_ZIP			8
swf, EXPLODE_ZIP			7
fws, EXPLODE_RAR			7
cws, EXPLODE_ZIP			7
der, EXPLODE_ZIP			5
chm, EXPLODE_RAR			5
mov, EXPLODE_ZIP			4
pem, EXPLODE_ZIP			3
tar, EXPLODE_ZIP			2
pdf, EXPLODE_ZIP			2
pdf, EXPLODE_RAR			2
officex, EXPLODE_ZIP	2
cws, EXPLODE_RAR			2
wmv, EXPLODE_ZIP			1
wmv, EXPLODE_RAR			1
rtf, EXPLODE_ZIP			1		[3]
rtf, EXPLODE_RAR			1		[3]
officex, EXPLODE_RTF	1		[3]
officex, EXPLODE_RAR	1
mp3, EXPLODE_ZIP			1
mov, META_PE					1
cab, EXPLODE_ZIP			1
cab, EXPLODE_OLE			1
```

Now, these are some interesting results! There are many ways to interpret this data, but here's my immediate reaction:

* The dominate child object appears to be digital signatures exploded from PE files [1]
* There may be a zip bomb in the dataset [2]
* There is a high ratio of JAVA class files to jar files [3]
* The officex file found in one of the two RTF files is an interesting outlier [4]

I should wrap this post up by exploring some of these file types. Since there are a lot of PE files, we'll look at those first. I wonder how many unique imphashes appear in the PE files?

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

A lot! There were 2,445 imphashes, far too many to include on this page. ~86% of them were unique and were only seen once. The results above show that, at the high end, one imphash appeared over 1000 times. A cursory look on Virus Total suggests that this imphash may be related to adware / spyware.

Finally, let's see what's in those OLE files. Specifically, I want to see if there is any macro code in them that calls out to malicious IP addresses / domains or drops files to disk. We can use the IOC field in the META_OLEVBA module to identify those.

```
Urlmon.dll 																			1
cureall.exe 																		1
shell32.dll 																		1
amnestic.exe 																		1
hxxp://geobrugg[.]co[.]kr/bbs/factuur2390.exe 	1
factuur2390.exe 																1
```

There weren't many OLE files in the dataset, so the lack of results isn't very surprising. With that said, this can be useful data to pivot from if we're interested in exploring these files further.

I think that's where I'll leave this post. If there's interest in this, I'll continue to take a look at the VXShare torrent datasets and blog about them.

---
author: Axiom
pubDatetime: 2024-03-25T22:21:00.737Z
title: Rapture
slug: rapture
featured: true
draft: false
tags:
  - OSINT
  - Privacy
  - Homelab
  - Security
  - Data
description: A comprehensive guide to self-hosting Rapture, the "no-nonsense data breach search interface"
---

## Table of Contents

## Introduction

Open-source intelligence (OSINT) can be described as data that is publicly available and its analysis. As a user of the internet, regardless of your occupation and/or interests, you have (likely unwittingly) leveraged the powers of open-source intelligence, as data powers the digital world. For the cybersecurity professional, the ability to collect and analyze open-source intelligence is indispensable. The vast presence and inherent publicity of open-source intelligence can enable cybersecurity professionals to engage in in-depth analysis of threats and perform due diligence in other adjacent fields. This can also benefit an individual or organization by assisting them in the enumeration/identification of their surface area of exposure. Even in this digital world, there are troves of valuable information that often go unnoticed. That is the information present within data breaches/leaks. These breaches contain various data points that can prove to be incredibly valuable in many different ways.

search.0t.rocks (previously known as search.illicit.services) was a search engine that enables users to efficiently search for specific records in compromised data. The original version/implementation hosted by MiyakoYakota leveraged the open-source search platform, Apache Solr, and served nearly 15 billion records towards the end of its operation. Some alternatives exist such as Dehashed and HaveIBeenPwned; however, these services often fall short. HaveIBeenPwned is free (API key is paid), but only presents whether or not a certain email is present in a data breach. It does not present any further information (apart from information relating to a data breach). Dehashed is a paid service and I have personally found the interface to be counterintuitive on occasion. IntelX's pricing is exorbitant with the cheapest SMB & Enterprise tier being €2.500/$2,721.26 (as of the date/time of initially posting this). There are free tiers, but their capabilities are incredibly limited. This proves to be a barrier to most individuals. Search.0t.rocks was an entirely free service that presented this information in a consistent and minimal manner in spite of services like IntelX; however, it has recently shut down (once again). It is unlikely that it will be up and running again, so the developers have provided an open-source repository that enables individuals to build and serve it locally.

Some barriers still exist, specifically relating to adding data to the self-hosted service as the user is personally required to clean and present data. There are also some (albeit only a few) issues with self-hosting search.0t.rocks. Some issues commonly arise during setup and configuration, and there is a lot of extra code that is unnecessary for the average user looking to self-host. Unfortunately, the code is a bit hard to understand which may discourage future maintenance, so I decided to rebuild the software from the ground-up, and I have titled this project Rapture. Rapture's code base is incredibly lightweight, enabling the software to undergo further maintenance, development, and expansion with ease. The code, setup/configuration, and interface remain consistent with the core tenants of simplicity and usability exhibited by search.0t.rocks. It also leverages the Apache Solr search platform.

One might argue for the use of Glogg, Klogg, or a similar application that enables individuals to index and query large files; however, this is disadvantageous when handling multiple data breaches. You are required to index a file every time before querying it, and queries take a long time. With Rapture, which leverages Apache Solr, you only have to index the breach file a single time, queries are incredibly fast, you can host and access the service over the network, and perform boolean queries. These capabilities render Rapture's user experience superior to the proposed alternatives. These differences are critical in time-sensitive investigations.

This article is a comprehensive guide to configuring Rapture locally, cleaning and converting data into a valid format, and importing the data into Rapture. Although this guide was designed for the configuration of Rapture, these steps are for the most part, cross-compatible with search.0t.rocks. These steps can also be performed in a headless manner, via remote connection to a machine.

## Disclaimers

Although a disclaimer is already included within the Rapture repository, I feel it is important to reiterate considering the nature of this documentation:

It is the end user's responsibility to obey all applicable local, state, and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program. This software comes AS IS with NO WARRANTY and NO GUARANTEES of any kind. By using Rapture, you agree to the previous statements.

The information below (within this disclaimer section) has been paraphrased by myself, but provided by Rob Volkert through OSINTCURIOUS ([Direct](https://www.osintcurio.us/2019/05/21/basics-of-breach-data/) | [Archive](https://web.archive.org/web/20240322013722/https://www.osintcurio.us/2019/05/21/basics-of-breach-data/)). If you are looking to view the recommended guidelines in more detail, please view the full post via the link provided.

Although this data is publicly available, it is incredibly important that you revisit any applicable legislation. There are also commonplace guidelines recognized by the community that your activities should remain consistent with:

- You should not illegally profit from any compromised data
- You should not sell compromised data
- You should not solicit an individual to compromise data
- You should report any instances of illegal activity to the applicable law enforcement agencies

## Getting Started

### Prerequisites

- Git
- Docker (w/ Docker Compose)
- Java Runtime (8+) `sudo apt install openjdk-19-jre-headless`

### Installation

The `setup.sh` script that is integrated within Rapture and automatically executed, relies on and/or creates a Solr collection by the name `BigData` in order to be cross-compatible with pre-existing search.0t.rocks installations/configurations.

1. Fetch repository from Github:

   `git clone https://github.com/ax-i-om/rapture.git`

2. Change directory

   `cd rapture`

3. Build via Docker Compose

   `docker compose build`

4. Run via Docker Compose

   `docker compose up -d`

If you are following these instructions to setup **search.0t.rocks**, rather than Rapture, then you must execute the `run.sh` script that is included after step 4. If you are unable to successfully execute the `run.sh` script included with **search.0t.rocks** repository, please see the [troubleshooting](#troubleshooting) section below.

All necessary installation steps have been completed, and you should now have an instance of Rapture accessible on port [6175](http://localhost:6175) (or otherwise configured port, may take a minute to initialize). If you chose to install search.0t.rocks instead, you should now have an instance accessible on port [3000](http://localhost:3000) (or otherwise configured port). You can now move on to preparing your data for importation!

## Data Conversion

Now that you have successfully configured and launched an instance of Rapture and the technologies it depends on, it is time to prepare the data you wish to index. This part is where the difficulties reside, and the manners in which the datasets are prepared vary depending on the data points and structures presented. I would highly recommend that you familiarize yourself with a high-level programming language such as Python, as the ability to create and modify specialized scripts for preparing data is essential when handling large datasets. These examples will include hand-crafted samples that can be experimented on. All of the information/data presented in these examples were fabricated/manipulated/randomly generated; however, the formats of the datasets may share similar characteristics and fields present in many breaches. The data presented may also be formatted to be consistent with the _"dirty"_ formats that may be present within real-world datasets. Fortunately in the case of many publicly-available datasets, they have already been cleaned to some extent which greatly reduces the amount of work for us.

In this section of the guide, I will provide various examples of how to use the Python language to format datasets that necessitate various levels of preparation. This section of the article is not designed to _give_ you the answers on how to clean datasets, but to demonstrate the processes and workflows of cleaning such data. The ability to recognize patterns in datasets and employ solutions that leverage the patterns present is crucial. Be advised that there may not always be a _perfect_ solution, and this process consists strongly of trial and error.

You may notice some patterns among the scripts presented in this section, as each script used is a derivative of a base script/algorithm that sequentially iterates through these large files. I will only explain concepts such as appending an ID key/value once. Further explanation would be redundant, and such implementations can easily be derived by viewing the provided solutions.

This section is subject to expansion in the future.

### Example One

For the first example, we will be handling a sample dataset that has already been 'cleaned,' but we will format it to be compatible with Rapture. The specific dataset provided below (_ex1.csv_) is in CSV format; however, other file/data formats are supported such as JSON and XML.

```csv
email,password,firstName,lastName,userID,FBID
rapture@demo.com,VK9RSuK0wSSXNc0gF8iYW1f6,axiom,estimate,4,
rapture@demo.com,N7cblKU9ypU727lwiTr9espw,garble,vortex,5,
rapture@demo.com,eP0u9cM0jAa2QeUVI3d88rYn,vertiable,slap,6,
rapture@demo.com,QxtyRMAx3KniskzjGDg6tHdl,axiom,terrific,7,
rapture@demo.com,cirSMQZp7Enh98KLb6r8JT1I,garble,eighty,8,
rapture@demo.com,9J53HQEetTv5E2xCJKe4tdaP,veritable,tumble,9,
rapture@demo.com,3sZ7NPb54Fk0Qy2LXlLejwCu,axiom,slipper,10,
rapture@demo.com,OxoZGbn3v0tvBMyWA0Jds0Ea,garble,chord,11,
rapture@demo.com,et8ggkgUeyQ3ge7ua2YNsOLd,veritable,baffle,12,
```

Only the fields specified during the execution of `setup.sh/run.sh` are valid, and fields are case sensitive; therefore, `firstName` is a valid key/field, whereas `firstname` and `FIRSTNAME` are invalid. Likewise, the `passwords` field is valid; however, the singular iteration of the word (`password`) is invalid.

#### Issues

Even though this dataset appears to be well organized, it cannot be successfully posted to Rapture in its current state, as the file requires an `id` field. Although other values may be omitted, each record requires an `id` to be present. It is up to you how you format the `id` value; however, it is important that each and every occurrence of an `id` is unique. In the case of a duplicate `id`, data will be overwritten. If you fail to properly provide a record identifier, you will get the following error message when attempting to import your data:

`Document is missing mandatory uniqueKey field: id`

We can also observe that the `email` and `password` fields do not match the fields we specified while executing the `setup.sh` script; therefore, we must modify these field names to `emails` and `passwords` respectively.

Although I might add such fields in the future, the `setup.sh` script did not specify a `userID` or `FBID` field, further rendering this dataset incompatible with Rapture in its current state.

#### Solution One

In a CSV file, the first row contains the headers. As we will be preparing a script that iterates through the datasets line by line, we will ensure the first action we take is to modify the headers accordingly depending on our goal.

As each record requires a unique identifier (`id`), we will prefix the headers with `id,`. Next, we will append an `s` to the end of the `email` and `password` headers so that they correspond with the fields we specified in `setup.sh`. Finally, we will remove the `userID` and `FBID` headers from this file, as I will be omitting these data points in this solution. If you wish to preserve these data points, please see [Solution Two](#solution-two). If you decide to follow this solution and remove the `userID` and `FBID` headers, be aware that you must also remove the corresponding values for each and every corresponding record (even if they do not have a value), otherwise you will see an error as such:

`CSVLoader: input=null, line=1,expected 5 values but got 7`

Since the CSV headers only occupy the first row, we can simply set the first iteration of our script to handle this modification, and then every subsequent iteration to modify the records accordingly. For each record, we must prefix a unique record identifier. As previously mentioned, using duplicate record identifiers will lead to the unintentional overwriting of data. It is recommended to use a common scheme for specifying a record identifier. For the intents of this guide, I will use the following data to create a unique identifier:

**record: 0** - This is the iteration. The first iteration (record 0), refers to the CSV headers (first line). So when the `record` iterator is equal to `0`, we will modify the headers. After every iteration (every line of the CSV file), the value of `record` will increment by 1. So when we reach the first row following the headers, the value of **record** will be equal to 1.

**breachDate: 01JAN1337** - This variable corresponds with the date in which the data breach occurred. Since my data sample was fabricated, I am also using a fabricated value.

**cleanDate: 15MAR2024** - This variable corresponds with the date in which the data was cleaned/prepared to be imported into Rapture. This way, if you import multiple batches of the same breach on different dates, you will maintain a unique record identifier.

**title: R4PTUR3** - The title typically corresponds with the website/service that the breach data corresponds with. Again, since this sample data was fabricated, I am also using a fabricated value here.

Using this data in our script will generate a record identifier that appears like so: `R4PTUR3-01JAN1337-15MAR2024-1`, where the `1` value will increment by 1 for every iteration. So the 100th converted record will have the unique record identifier of: `R4PTUR3-01JAN1337-15MAR2024-100`. Since we are removing the `userID` and `FBID` fields/headers, we must also remove the corresponding data for each row. This is actually quite simple due to the comma delimiting leveraged by the CSV (comma-separated values) format. In the header, we can observe that the 4th instance of a comma (,) is used to delimit the `userID` value, and subsequently the `FBID` value. Therefore, we can simply locate the index of this fourth comma and then slice the string, only preserving the contents up to the index of the fourth comma. We can (and will) apply this process to both the headers and the records.

After slicing the string to remove the undesired values, we prefix the string with the unique record identifier we generated. We then use the `replace()` function to locate the `email` and `password` strings in the header and simply replace them with `emails` and `passwords` respectively. We then append the record to a new file, and append a newline ("\n").

After performing the steps presented above, our header should look like so:

**BEFORE:** `email,password,firstName,lastName,userID,FBID`

**AFTER:** `id,emails,passwords,firstName,lastName`

After applying the modifications, the records should also be formatted like so:

**BEFORE:** `rapture@demo.com,VK9RSuK0wSSXNc0gF8iYW1f6,axiom,estimate,4,`

**AFTER:** `R4PTUR3-01JAN1337-15MAR2024-1,rapture@demo.com,VK9RSuK0wSSXNc0gF8iYW1f6,axiom,estimate`

Here is what the full, converted sample should look like after performing these steps:

```csv
id,emails,passwords,firstName,lastName
R4PTUR3-01JAN1337-15MAR2024-1,rapture@demo.com,VK9RSuK0wSSXNc0gF8iYW1f6,axiom,estimate
R4PTUR3-01JAN1337-15MAR2024-2,rapture@demo.com,N7cblKU9ypU727lwiTr9espw,garble,vortex
R4PTUR3-01JAN1337-15MAR2024-3,rapture@demo.com,eP0u9cM0jAa2QeUVI3d88rYn,vertiable,slap
R4PTUR3-01JAN1337-15MAR2024-4,rapture@demo.com,QxtyRMAx3KniskzjGDg6tHdl,axiom,terrific
R4PTUR3-01JAN1337-15MAR2024-5,rapture@demo.com,cirSMQZp7Enh98KLb6r8JT1I,garble,eighty
R4PTUR3-01JAN1337-15MAR2024-6,rapture@demo.com,9J53HQEetTv5E2xCJKe4tdaP,veritable,tumble
R4PTUR3-01JAN1337-15MAR2024-7,rapture@demo.com,3sZ7NPb54Fk0Qy2LXlLejwCu,axiom,slipper
R4PTUR3-01JAN1337-15MAR2024-8,rapture@demo.com,OxoZGbn3v0tvBMyWA0Jds0Ea,garble,chord
R4PTUR3-01JAN1337-15MAR2024-9,rapture@demo.com,et8ggkgUeyQ3ge7ua2YNsOLd,veritable,baffle
```

This data can be successfully imported into rapture and subsequently queried. Below, I provide the script used to achieve this result.

```python
from itertools import islice

def fInstance(base: str, search: str, n: int) -> int:
	x = base.find(search)
	while x >= 0 and n > 1:
		x = base.find(search, x+len(search))
		n -= 1
	return x

def main():
	path = './ex1.csv'
	saveAs = open('ex1-notpreserved.csv','a+')
	batch_size = 100

	breachDate = "01JAN1337"
	cleanDate = "15MAR2024"
	title = "R4PTUR3"
	record = 0

	with open(path, 'r') as file:
		while True:
			lines = list(islice(file, batch_size))
			if not lines:
				break
			for line in lines:
				if record == 0:
					resLine = "id,"+line[:fInstance(line, ",", 4)] + "\n"
					resLine = resLine.replace("email", "emails")
					resLine = resLine.replace("password", "passwords")
					saveAs.write(resLine)
				else:
					recordIdentifier = title + "-" + breachDate + "-" + cleanDate + "-" + str(record)
					saveAs.write(recordIdentifier + ","+line[:fInstance(line, ",", 4)] + "\n")
				record += 1

if __name__ == "__main__":
	main()
```

#### Solution Two

If you wish to preserve the `userID` and `FBID` values, you must add the corresponding fields to the Solr schema. I also rename the `FBID` field to `facebookID` to remain cohesive with the other field naming schemes. You must also modify the headers as mentioned above, by adding the `id` header and appending an `s` to the `email` and `password` headers. Since we are not required to discovered the _nth_ instance of a comma in a line (since we are not removing any fields/values), we can omit the `fInstance()` function.

Here is what the full, converted sample should look like after performing these steps:

```csv
id,emails,passwords,firstName,lastName,userID,facebookID
R4PTUR3-01JAN1337-15MAR2024-1,rapture@demo.com,VK9RSuK0wSSXNc0gF8iYW1f6,axiom,estimate,4,
R4PTUR3-01JAN1337-15MAR2024-2,rapture@demo.com,N7cblKU9ypU727lwiTr9espw,garble,vortex,5,
R4PTUR3-01JAN1337-15MAR2024-3,rapture@demo.com,eP0u9cM0jAa2QeUVI3d88rYn,vertiable,slap,6,
R4PTUR3-01JAN1337-15MAR2024-4,rapture@demo.com,QxtyRMAx3KniskzjGDg6tHdl,axiom,terrific,7,
R4PTUR3-01JAN1337-15MAR2024-5,rapture@demo.com,cirSMQZp7Enh98KLb6r8JT1I,garble,eighty,8,
R4PTUR3-01JAN1337-15MAR2024-6,rapture@demo.com,9J53HQEetTv5E2xCJKe4tdaP,veritable,tumble,9,
R4PTUR3-01JAN1337-15MAR2024-7,rapture@demo.com,3sZ7NPb54Fk0Qy2LXlLejwCu,axiom,slipper,10,
R4PTUR3-01JAN1337-15MAR2024-8,rapture@demo.com,OxoZGbn3v0tvBMyWA0Jds0Ea,garble,chord,11,
R4PTUR3-01JAN1337-15MAR2024-9,rapture@demo.com,et8ggkgUeyQ3ge7ua2YNsOLd,veritable,baffle,12,
```

This data **CAN NOT** be successfully imported into rapture and subsequently queried. If you attempt to import this data, you will receive the following error message:

`ERROR: [doc=R4PTUR3-01JAN1337-15MAR2024-1] unknown field 'userID'`

In order to fix this, we must add the necessary fields to our Solr schema. We can easily achieve this using curl.

`curl 'http://127.0.0.01:8983/solr/BigData/schema?wt=json' -X POST -H 'Accept: application/json' --data-raw '{"add-field":{"stored":"true","indexed":"true","name":"userID","type":"string"}}'`

`curl 'http://127.0.0.01:8983/solr/BigData/schema?wt=json' -X POST -H 'Accept: application/json' --data-raw '{"add-field":{"stored":"true","indexed":"true","name":"facebookID","type":"string"}}'`

These commands add the `userID` and `facebookID` fields, respectively. After executing these commands, we can successfully import and query the converted data. Below, I have provided the corresponding script used to convert the data while preserving the `userID` and `facebookID` fields.

```python
from itertools import islice

def main():
	path = './ex1.csv'
	saveAs = open('ex1-preserved.csv','a+')
	batch_size = 100

	breachDate = "01JAN1337"
	cleanDate = "15MAR2024"
	title = "R4PTUR3"
	record = 0

	with open(path, 'r') as file:
		while True:
			lines = list(islice(file, batch_size))
			if not lines:
				break
			for line in lines:
				if record == 0:
					resLine = "id," + line
					resLine = resLine.replace("email", "emails")
					resLine = resLine.replace("password", "passwords")
					resLine = resLine.replace("FBID", "facebookID")
					saveAs.write(resLine)
				else:
					recordIdentifier = title + "-" + breachDate + "-" + cleanDate + "-" + str(record)
					saveAs.write(recordIdentifier + ","+line)
				record += 1
	saveAs.close()

if __name__ == "__main__":
	main()
```

### Example Two

In the second example, we will be handling a real-world dataset, specifically consisting of `.txt` files. The records that are contained within this dataset titled `2_2.txt` which primarily consists of data compiled/aggregated from malware/stealer logs. The data presented here has been manipulated, and any sensitive credentials or personally identifiable information has been replaced.

```
https://osu.ppy.sh/home/download:axiom:VK9RSuK0wSSXNc0gF8iYW1f6
https://www.facebook.com/:rapture@demo.com:N7cblKU9ypU727lwiTr9espw
https://auth.riotgames.com/login:rapture@demo.com:eP0u9cM0jAa2QeUVI3d88rYn
https://accounts.google.com/signin/v2/challenge/pwd:rapture@demo.com:QxtyRMAx3KniskzjGDg6tHdl
https://discord.com/channels/881160035171001562/359675218011979118:rapture@demo.com:cirSMQZp7Enh98KLb6r8JT1I
https://www.facebook.com/:rapture@demo.com:9J53HQEetTv5E2xCJKe4tdaP
https://login.live.com/login.srf:rapture@demo.com:3sZ7NPb54Fk0Qy2LXlLejwCu
https://www.twitch.tv/directory/game/VALORANT:axiom:OxoZGbn3v0tvBMyWA0Jds0Ea
https://login.live.com/login.srf:rapture@demo.com:et8ggkgUeyQ3ge7ua2YNsOLd
```

#### Issues

The example dataset provided above seems fairly straightforward at first glance. It is a `.txt` file where the records (which appear to be _domain:username/email:password_) are delimited by a colon (:). This presents the first issue, as there are colons present in the address bar. By using RegEx, this is pretty easily handled; however, not all of the URLs share the same format. There are various protocols indicated in the address bar throughout the dataset including: `http`, `https`, `file`, `android`, and `ftp`. Some of the URLs/addresses completely omit the protocol such as `login.live.com`. Some of the addresses are present on a local area network, and designated by `localhost` or a private IP address, and may or may not specify a port number. Some addresses are a public IP address that may or may not specify a port number, and there are also addresses that only start with `//`. I spent hours working on a RegEx pattern that covered all these bases, and got pretty close; however, there was one exception that I just could not solve so I ended up scrapping it. I tried to fix it by leveraging various AI-powered tools, and tried to simply generate a pattern with AI. These AI-powered endeavors were unsuccessful. I may attempt to develop a suitable RegEx pattern again in the future; however, this example is based on a real-world dataset consisting of hundreds of files so I ended up changing my workflow. Instead of trying to handle all these different edge cases, I simply split up my workload based on the count of colons (:) present in each line. If there are more than three colons present in a line, I write it to `rejected.txt` for further review. Any lines with three or less colons present were handled/process. As I write this, I realize another option for handling the colon delimiting would be to parse the line from the end rather than from the beginning.

Now that we have decided how to handle the addresses (for now), we can move on to processing the credentials present in each record. I also tried to use RegEx patterns to identify whether the second field contained an email address, or if it was a password. This was a failure, as there were many malformed email addresses. I had attempted to implement prompting here that would ask the user for input when the script encountered a string that contained an "@" character but was not matched by the Regular Expression. The prompt would ask the user to review the string, and decide whether it was an email, username, or if they didn't know. The script would then write the credential according to the user input; however, this is incredibly tedious when handling millions of records, and may be even more innaccurate. For the application in question, this distinction is likely counterintuitive so I ended up passing the value to both the `usernames` and `emails` field.

There are also some instances of backslashes and double quotes in these fields, so I will handle these occurrences accordingly by using the `replace()` function.

#### Solution

Here I have provided a script that implements this workflow. This solution also skips blank lines entirely.

```python
import re
from itertools import islice

OKDOMAINEXP = r'(((http|https|ftp|file|android):\/\/)|(\/\/)).*?(?=:)'
LOGINEXP = r'.*?(?=:)'
PASSWORDEXP = r'(?<=:).*'

def main():
    path = './2_2.txt'

    saveAs = open('ready.json','w')
    rejected = open('rejected.txt','a+')
    batch_size = 100

    breachDate = "COLLECTION"
    cleanDate = "25MAR2024"
    title = "R4PTUR3"
    record = 0

    with open(path, 'r') as file:
        saveAs.write('[')
        while True:
            lines = list(islice(file, batch_size))
            if not lines:
                break
            for line in lines:
                if line.count(":") > 3:
                    rejected.write(line)
                    continue
                recordIdentifier = title + "-" + breachDate + "-" + cleanDate + "-" + str(record + 1)
                try:
                    if record != 0:
                        saveAs.write(",")
                    domain = re.search(f'{OKDOMAINEXP}', line).group(0)
                    next = re.sub(re.escape(domain) + ":", '', line)
                    login = re.search(f'{LOGINEXP}', next).group(0)
                    password = re.search(f'{PASSWORDEXP}', next).group(0)
                    saveAs.write('\n\t{\n')
                    saveAs.write('\t\t"id":"' + recordIdentifier + '",\n')
                    saveAs.write('\t\t"domain":"' + domain.replace('\\', '\\\\').replace('"', '\\"') + '",\n')
                    saveAs.write('\t\t"emails":"' + login.replace('\\', '\\\\').replace('"', '\\"') + '",\n')
                    saveAs.write('\t\t"usernames":"' + login.replace('\\', '\\\\').replace('"', '\\"') + '",\n')
                    saveAs.write('\t\t"passwords":"' + password.replace('\\', '\\\\').replace('"', '\\"') +'"\n')
                    saveAs.write("\t}")
                except:
                    if len(line) > 1:
                        rejected.write(f'{line}')
                record += 1
        saveAs.write("\n]")
    saveAs.close()
    rejected.close()

if __name__ == "__main__":
```

By passing the sample dataset presented above to this script, we can observe that no lines were rejected. The results depicted in `ready.json` can be observed here:

```json
[
	{
		"id":"R4PTUR3-COLLECTION-25MAR2024-1",
		"domain":"https://osu.ppy.sh/home/download",
		"emails":"axiom",
		"usernames":"axiom",
		"passwords":"VK9RSuK0wSSXNc0gF8iYW1f6"
	},
        ...
	{
		"id":"R4PTUR3-COLLECTION-25MAR2024-9",
		"domain":"https://login.live.com/login.srf",
		"emails":"rapture@demo.com",
		"usernames":"rapture@demo.com",
		"passwords":"et8ggkgUeyQ3ge7ua2YNsOLd"
	}
]
```

This file can be indexed without issue.

## Data Importation

The process of importing data to Solr is very straightforward, as long as you have properly prepared your data. In order to start, you must first download and extract the Solr binaries from [https://solr.apache.org/downloads.html](https://solr.apache.org/downloads.html). In this article, I will be using version 9.5.0.

Ensure that you have installed a suitable version of the Java runtime. If you have not installed a suitable version of the Java runtime, Solr will likely inform you of this and present what versions are acceptable.

The file types considered/accepted by the Solr POST tools are as follows:

```txt
XML,JSON,JSONL,CSV,PDF,DOC,DOCX,PPT,PPTX,XLS,XLSX,ODT,ODP,ODS,OTT,OTP,OTS,RTF,HTM,HTML,TXT,LOG
```

Although there are plenty of acceptable file types, we will primarily use either XML, JSON, or CSV. Any errors that arise during the use of these commands are most likely caused by malformed data. After the successful usage of these commands, you will be able to query the indexed documents via Rapture or search.0t.rocks.

### bin/post

```
~/Desktop/solr-9.5.0/bin/post -c BigData -p 8983 -host 127.0.0.1 formattedData.json
```

**~/Desktop/solr-9.5.0/bin/post** - path to post binary

**-c BigData** - specify collection

**-p 8983** - specify which port Solr is running on

**-host 127.0.0.1** - specify the host address running Solr

**formattedData.json** - contents to post to Solr

### bin/solr

Although the above command leveraging the Solr POST tool works, the `bin/post` script is deprecated in favor of the `bin/solr` POST command. The syntax of the command does vary; however, it is still incredibly straightforward. I have converted the command displayed above to instead use the `bin/solr` POST command.

```
~/Desktop/solr-9.5.0/bin/solr post -c BigData -url http://127.0.0.1:8983/solr/BigData/update?commit=true formattedData.json
```

**~/Desktop/solr-9.5.0/bin/solr** - path to Solr binary

**post** - specify the POST command

**-c BigData** - specify collection

**-url http://127.0.0.1:8983/solr/BigData/update?commit=true** - specify the update URL for the Solr collection

**formattedData.json** - contents to post to Solr

## Troubleshooting

- I have been contacted various times by individuals experiencing issues while trying to execute the `run.sh` setup script that is included with the search.0t.rocks repository. In commit [4c71edd](https://github.com/MiyakoYakota/search.0t.rocks/commit/4c71edd10e727fbcfd98920bd54d3d1258673e6f) titled _Update run.sh_, the curl commands switched from using a hard-coded host address and port number (`localhost:8983`)to using variables specified by the user: (`$ip:$port`) This layout prevented the variables from expanding, meaning curl was using $ip:$port as the URL, literally. This lead to the error: curl: (3) URL using bad/illegal format or missing URL for each curl statement. This issue is fixed by switching the single quotes (') surrounding the curl's positional URL argument to double quotes (") so that the variables could successfully expand. I opened a pull request that fixes the issue ([Pull Request #6](https://github.com/MiyakoYakota/search.0t.rocks/pull/6)), but it has not yet been merged. In the meantime, please use the `run.sh` file provided in the pull request or fix your existing script according to the information presented herein.

- Sometimes errors might occur when using the Solr binaries if you move them from their original directory (wherever you extracted the contents of the downloaded .tgz file to).

## Future plans

Although Rapture is technically viable and functioning in its current state, it is by no means finished. There are various plans for Rapture that will significantly improve the user experience. In the case any significant changes are made, I will properly update this article/guide with the corresponding information. The current plans for Rapture and this article are as follows:

- Add more examples for cleaning data
- Create a script that simplifies the process of manipulating the Solr schema
  - Adding, modifying, and removing fields
  - ASCII Folding
  - Etc...
- Add password extended search
- Add extended search for license plate/VIN by leveraging the VIN library in [Bitcrook](https://github.com/ax-i-om/bitcrook)
- Add `NOT` queries
- Further optimization/performance enhancement

## AI Disclosure

The banner/header image featured in this article was generated using OpenAI's DALL-E 2 AI system.

---
title: "OSCP and OSWP review"
header:
  teaser: /assets/images/oscp-oswp-review/teaser.png
  image: /assets/images/oscp-oswp-review/splash.png
toc: true
toc_sticky: true
tags:
  - certification
  - network protocols
  - windows
  - linux
  - Wi-Fi
---

# Offensive Security certifications

Among the offensive security trainings/certification providers, Offensive Security is probably one of the most known, especially for its well-recognised Offensive Security Certified Professional (OSCP) certification. My objective this year was to get two certifications from Offensive Security: the OSCP, which is considered as the first one to get for a penetration testing skillset, and the OSWP, which is a course specialised in Wi-Fi security.

Here I will share my experience and my opinion/recommendation on these two certifications that I achieved earlier this year.

# OSCP

First, it is worth knowing that the OSCP course has been updated while I was working on it. After starting with the old course theory and lab, I decided to continue and end with the updated version: about half of my time was spent on the old lab, and the other half on the new lab. Apparently, there is no difference in the exam difficulty: the difference is that we have more content in the theory and more machines in the lab. I also noticed that some of the machines of the old lab were sometimes updated or removed.

## My background prior to OSCP

Before taking OSCP I trained a lot on the [HackTheBox](https://www.hackthebox.eu/) platform: about 50 machines that I rooted sometimes all by myself, sometimes with more or less clues from the HTB forum or sometimes by following walkthroughs, mainly from [IppSec videos](https://www.youtube.com/channel/UCa6eh7gCkpPo5XXUDfygQQA). Although I also had experience in real pentesting with my job, HackTheBox helped a lot more because the hacking journey there is exactly the same as the machines that you will encounter during the OSCP exam: given an IP address, you need to get user and root access on either a Windows or a Unix machine, knowing that it is vulnerable to known exploit(s).

## The theory

By reading reviews of other OSCP students, I see a lot of different opinions on the syllabus content and the exercises that it proposes. What I would advise, even if you already know most of the concepts presented in the syllabus, is not to skip the theory and to read/experiment with it carefully, as the content gives an indication of what you could expect in both the lab and the exam. However, I skipped almost all of the exercises, except the ones I wanted to perform to experiment with a certain tool/technique. As a reminder, there is a possibility to gain 5 bonus points to the exam (rated on 100 points) if we write a report containing a walkthrough of all the exercises. The two reasons I did not take time to write it are the following:
- The exercices are numerous and a lot of them are not interesting if you have at least some experience in the IT/pentesting domain;
- There is only one exam situation where these 5 bonus points could help you.

For the second reason, this is because of the rating system: the exam is calculated on 100 points with 5 machines to root, and requires 70 points to be successful. 2 are worth 25 points, 2 are worth 20 points and 1 is worth 10 points. The only situation where these 5 points help to reach this threshold of 70 points is when we have 25 + 20 + 20 (3 machines rooted).

If there is a part of the theory that is mandatory, that would be the Buffer Overflow, as you will expect one machine to cover this technique during the exam. Mastering the Windows Buffer Overflow technique explained in the theory guarantees you 25 points, so it is crucial not to take that part lightly.

To finalise, I noticed that some content of the theory is nice to know/experiment with and really helpful for the lab but useless for the exam (e.g. client-side and Active Directory attacks), but I will come back on that in the next part.

## The lab

The most interesting feature of the OSCP course is the lab: this is a large reality-like network, organised in servers, clients, with an Active Directory, etc. as well as simulated clients that use this infrastructure (and that are prone to client-side attacks!). During the time I had in the lab (90 days), I did not have time to root all the machines, I was even far from that.

The lab is where you will spend most of your time to train for the OSCP, it contains a lot of machines that are similar to the machines that you could expect for your exam. I will not go into details for my journey through the lab and its machines, I will just highlight an important aspect here that you will need to take into account. As mentionned, the lab includes a lot of machines that are connected together: a part of them cannot be hacked without prior information that you will find after a post-exploitation phase on the other machines (as expected in reality when you are dealing with an internal network). Some of them need client-side attacks and some of them also require pivoting. All these techniques are really nice to train to strenghten your Red Teaming skills, but they are useless if your purpose is to focus solely on the exam.

My advice would then depend on the time you have to train for the exam. If your time is limited, prioritise machines that can be rooted independently with known exploits and privilege escalation that do not required unknown keys/passwords. This may be frustrating as you never know which ones fall in that category of course, although some of these machines are often recommended by other OSCP reviews, i.e. the big5 of the machines: Pain, Gh0st, Sufferance, FC4 and Humble.

There are also 3 exercices to train your Windows Buffer Overflow technique. There are not difficult, but allow you to assert if you are ready for the exam, so I would recommend to take time to train on these.

Outside of my limited time dedicated to the lab, my best source of training was HackTheBox, especially easy-medium (old) boxes. List of recommended boxes on this platform are available with a bit of research, an example that I used was [this one](https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview#). To keep a track of my activity, I recorded hours and content of my training until the exam. You can see these details below.

![Hours](/assets/images/oscp-oswp-review/hours.png)
*Time spent on my OSCP training*

## The exam

If you did enough training, feel confident hacking machines (i.e. you successfully hacked several machines by your own in both the OSCP lab and HTB) and if you have the right mindset, then you are more than ready for the exam. The methodology I suggest is similar to what you can already find in other OSCP reviews:

1. Run a scan on all the machines in scope (4, excluding the Buffer Overflow machine) with a tool such as [AutoRecon](https://github.com/Tib3rius/AutoRecon), which will already perform relevant scans depending on the services found, e.g. launch DirBuster when detecting a web application;
2. Begin the Buffer Overflow machine while the scans are running. This should take you 30m-1h. Even if it is taking you more time than that, it is perfectly fine, you have 24h in total for this exam which is way more time than required to root all the machines;
3. Attack the 20 or 25 points machines. If you feel stuck at some point, switch the machine and keep the 10 points machines to gain free progress later (as it was really an easy machine in my case). In my exam, I found a 20 points machine more difficult than the 25 points machine, so do not pay too much attention to this difference of 5 points;
4. Take regular breaks to eat and even to sleep if necessary. This 24h period has been defined so that you have time to do that.

In my case, I was able to reach the 70 necessary points after ~7h, and I spent 2 hours on the Buffer Overflow because of a simple mistake! So never feel discouraged if you are stuck at some point :) I rooted all the machines after ~13h (I used Metasploit for the last missing privilege escalation exploit).

The report is not difficult to write if you carefully follow the instructions (include screenshots, explain step-by-step, etc.) and if you documented your hacking journey properly.

## Recommendation

I would highly recommend this course to anyone who wants to work in the offensive security domain, i.e. pentesting of web applications, networks, etc. and Red Teaming. The hands-on experience that you will get is really worth the time you will spend to pass this certification.

# OSWP

This review will be way shorter than my OSCP review, for three reasons:
- This certification only focuses on wireless security and therefore has a scope and content that is more limited compared to the OSCP;
- The OSWP course is outdated and does not provide a lot of useful content for Wi-Fi security nowadays;
- The exam is easy and does not really consists in a challenge if you read/practice the syllabus content carefully.

## The course (theory)

The OSWP course consists only of theory and exercises (it does not feature a lab as opposed to the OSCP). The Wi-Fi attack exercises require the set up of you own lab that consists in a Wi-Fi card able to inject packets (such as an ALFA card) and a router where WEP security can be configured.

The course is interesting to understand the foundation of the wireless security mechanisms, but does not bring a lot of value regarding the current Wi-Fi security landscape. The content covers mostly WEP security, although WEP is not used anymore. Nowadays the simple fact that WEP would be in use in a wireless network already consists in a critical vulnerability. The more recent WPA/2 protocol is only covered by the PSK 4-way handshake bruteforce attack. Examples of missing content that would be necessary to cover nowadays are:
- WPA/2 Enteprise: what are the attack vectors against certificate-based authentication and RADIUS server?
- Wi-Fi Protected Setup (WPS)-based attacks: for example bruteforcing of the PIN code in place;
- WPA3: Although it is not widely used, what are the differences/improvement compared to WPA2 and what are the existing attacks against it?

Such outdated content makes the exam easier to pass, but makes our skills more difficult to apply in reality.

## The exam

The exam is 4 hours long and consists in cracking 3 Wi-Fi passwords, by reproducing techniques that were presented in the theory. The report is also very straightforward to write from there, so I did not see any real challenge here, especially when doing the comparison with the OSCP exam.

## Recommendation

I would not recommend this course, as it gives too few skills that could be reused in a real Wi-Fi network pentesting. The course explore the WEP protocol in-depth, which becomes more and more outdated with time, and does not give all the knowledge you would require to tackle a full Wi-Fi pentesting (we are far from there!). I guess that this course will be updated at some point, and would recommend to at least wait for such an updated version before spending time on this certification.


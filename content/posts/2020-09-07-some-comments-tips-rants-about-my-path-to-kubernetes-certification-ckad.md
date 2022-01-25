---
author: João Antunes
date: 2020-09-07 18:30:00+01:00
layout: post
title: "Some comments, tips and rants about my path to Kubernetes certification"
summary: "Sharing my experience about my path to learn about Kubernetes and prepare for the Certified Kubernetes Application Developer (CKAD) exam."
images:
- '/images/2020/09/07/path-to-ckad.jpg'
categories:
- smalltalk
tags:
- smalltalk
- kubernetes
- ckad
- certifications
slug: some-comments-tips-rants-about-my-path-to-kubernetes-certification-ckad
---

## Intro

As I'm not the most regular blogger, maybe it wasn't that apparent, but in the last couple of months I've been even quieter over here, as I studied for the [Certified Kubernetes Application Developer (CKAD)](https://www.cncf.io/certification/ckad/) exam.

Some who know me will maybe remember that I'm not the biggest fan of certifications, so will be a bit surprised I took this one. It's rather simple to explain: my company challenged me to do it 😛. As Kubernetes was a subject I already had some curiosity before, but never invested the time to learn much about it, I thought it would be a good opportunity to learn something new. The certification itself was secondary.

Given the [Linux Foundation Global Certification and Confidentiality Agreement](https://docs.linuxfoundation.org/tc-docs/certification/lf-cert-agreement/#2-confidentiality-and-intellectual-property-ownership), I can't go into much detail about the exam itself, but I can talk about from where I'm coming from, how I prepared and some takeaways.

## Starting point

Coming into the certification, I had very little knowledge about Kubernetes. I had watched some videos about it, had some ideas about pods, some of their patterns (e.g. sidecar), but not much else Kubernetes specific. I did have previous knowledge about Docker (as you might have already seen in previous posts).

When I think about certification, I think about someone that already works with a given technology and certifies themselves on it to "prove" that indeed that skillset is possessed. So coming in from a different angle, as I did, feels a bit weird, but a learning challenge is a learning challenge 😛.

## Learning about k8s

Given my lack of background with k8s, I had to start from the beginning.

### The Linux Foundation training material

I was provided with some [training material from The Linux Foundation](https://training.linuxfoundation.org/), which I took advantage of but didn't really like it.

For starters, the initial content was about setting up a Kubernetes cluster, which I find completely irrelevant for those interested in the CKAD. That content is applicable for CKA (Certified Kubernetes Administrator), not really CKAD. From an application developer standpoint, using something like Docker Desktop ([for Windows in my case](https://docs.docker.com/docker-for-windows/kubernetes/)) or [MiniKube](https://kubernetes.io/docs/tasks/tools/install-minikube/) should be good enough. That doesn't mean it's not relevant to have an idea about Kubernetes' architecture, but configuring everything, not really.

After that, each content chapter started with an intro video that didn't say much, followed by what seemed a bit like a copy paste of excerpts of the documentation.

At the end of each chapter, there were some lab exercises, which for me were not very well structured, worsened by the fact that the learning platform is one of the two: bad or not correctly used.

I talked to another colleague which had done the certification previously, and he seemed to like the training material, so your mileage may vary. In my case... not a fan 🙂.

### Pluralsight

As I finished The Linux Foundation's initial training material, I was going to jump to the next one, which seemed a bit adjacent to the topic, but was part of the recommended material so, I was going for it.

Thankfully, as I talked to another colleague and mentioned my disappointment with the provided training resources, he mentioned he had noticed there was a [CKAD path available in Pluralsight](https://app.pluralsight.com/paths/certificate/certified-kubernetes-application-developer-ckad), which is a platform I often use, but didn't remember to look for that.

Good thing he mentioned this, because the content there was much more to my liking. Again, YMMV, but as I tend to enjoy content from Pluralsight, that was right up my alley.

### Other notes

The aforementioned resources were the ones I used, but there are more available. As I searched for tips on preparing for the exam, came across some recommendations of Udemy courses, but didn't try any of those as I already had the Pluralsight subscription (and also because I avoid using Udacity, for reasons described in the following linked posts by [Troy Hunt](https://www.troyhunt.com/the-piracy-paradox-at-udemy/) and [Rob Conery](https://medium.com/@robconery/how-udemy-is-profiting-from-piracy-5638b929ffca#.nzz4cq24z)).

## Practicing

Finishing up on the learning material, it was time to start practicing. The exam is a practical one, so no multiple-choice there. We get a terminal in the browser and need to work our way through the problems.

I had no real clue how to prepare, so I started by taking one of the many sample applications I have on my GitHub, to get it working in Kubernetes. I even posted a screenshot to social media as I was messing with things.

{{< embedded-image "/images/2020/09/07/messing-with-kubernetes-and-aspnet-core.jpg" "messing with Kubernetes and ASP.NET Core" >}}

Doing this allowed me to practice and better understand most of the concepts I learned, applying it to a project I knew well the inner workings.

Still, I wasn't sure if this was enough preparation for the exam. I felt confident about my understanding of the concepts, but from there to being agile solving problems during an exam, there's still some way to go.

At this point, I talked to a colleague that had already done the exam (the same I mentioned before), asking for some guidance regarding the preparation. Among other things, he pointed me to [killer.sh](https://killer.sh), a Kubernetes exam simulator, which provides an environment simulating the CKAD (and CKA) exam. It's not exactly the same, but close enough for practice purposes.

After all of this I have to say, killer.sh was a life-saver! My practice until this point was mostly creating the Kubernetes objects descriptions in YAML files, applying the changes with `kubectl apply -f something.yaml`, as I believe that to be the best approach for a real world scenario (i.e. to have things in source control). After doing a test run on killer.sh, it was obvious this approach wasn't gonna cut it. Almost all of the exercises I did during the exam simulation were correct, but still, after the targeted 2 hours, I failed to get the minimum score to pass (66% when I did the exam).

While I still believe using YAML files is the right approach in a real world scenario, in the exam with time quickly running out, we need to go faster. This means using a lot  more `kubectl`, including pre-generating a YAML file and then making changes on them, which is faster than rolling the complete YAML manually, or even searching for it in K8s docs to copy paste.

After this eye opening failure with killer.sh, I searched for some more sample exercises on the interwebs, and came across a couple of interesting ones:

- https://github.com/dgkanatsios/CKAD-exercises
- https://medium.com/bb-tutorials-and-thoughts/practice-enough-with-these-questions-for-the-ckad-exam-2f42d1228552

After all this practice and reviewing some content on the previously mentioned resources, tried a killer.sh test run again, closer to the exam, with much better results 🥳.

## From practice to the real thing

So, with all this studying and practice, how did it go during the exam?

Not bad actually. I wasn't able to finish all the questions, but was close, miles better than my first attempt on the killer.sh simulator. For the ones I did, I was mostly confident on their correctness. It's one of the good things of a practical versus a theoretical/multiple-choice exam, we can inspect things running, to see if it's working as we expect.

## Quick tips

I'm not going to write many tips here, as there are already a ton of posts of this sort. My idea for this post was just to share my experience (and some rants), but given I'm writing anyway, I'll drop some tips on the things I feel were most useful for me to get it done successfully.

### Write as little YAML as possible

As I mentioned earlier, writing the YAML files can slow you down, even if you copy paste from the Kubernetes docs, as you need to find the YAML first, so as much as possible use the `kubectl` commands to go faster. You can just execute the command, applying things immediately, or you can generate a YAML out of that command, meaning you'll only need to go into the file to do some tweaks, as the rest will already be ready.

Some examples of this YAML file generation I'm talking about:

```bash
# create a pod
k run some-pod --image=nginx --dry-run=client -o yaml > pod.yaml

# create a deployment
k create deploy some-deployment --image=nginx --dry-run=client -o yaml > deploy.yaml

# create a job
k create job some-job --image=busybox --dry-run=client -o yaml > job.yaml -- sh -c "echo hello!"

# create a service (ClusterIP, which is the default)
k expose pod some-pod --name some-service --port 3333 --target-port 80 --dry-run=client -o yaml > service.yaml

# create a service (LoadBalancer)
k expose pod some-pod --name some-service --port 3333 --target-port 80 --type LoadBalancer --dry-run=client -o yaml > service.yaml
```

Those `--dry-run` and `-o` are important, as they mean the command will not be applied immediately, but instead generate the YAML file. If you don't need to do any adaptations, then just ignore the YAML generation and apply changes immediately.

These are just some examples of commands, there are tons more, so I encourage you to practice as many as possible, to get fast.

### Text editors , IDEs and other tools

If you're like me and are accustomed to using something like Visual Studio Code for these kinds of things... don't! 🙃

During the exam you don't have it, so you better practice with that in mind. You can go with `vim`, which is likely the most common, or `nano`, which was what I used, cause I don't really care to learn all those shortcuts.

On the same note, if you normally reach out to something like Postman or Insomnia REST Client, better get used to `curl` and `wget`, cause that's what is commonly installed in Linux boxes, as well as containers.

As I spend most on my time on Windows, I did all my practice with WSL2 + Docker for Windows.

### Linux related bits

Did you notice the `k` instead of `kubectl` in the commands above? That's also a good tip. which is in fact recommended in the [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) (which we can use during the exam), to help us go faster, with less typing 🙂.

```bash
alias k=kubectl
```

You might consider creating even more alias for other common commands.

`kubectl edit` by default uses `vim`, so if you're of the minority like me and prefer `nano`, you can change it with:

```bash
export KUBE_EDITOR=nano
```

Besides this, it's probably useful to have some more knowledge of shell commands. This may be natural for many, coming from Linux or MacOS, but if you're coming from Windows, maybe you're not as used to doing things in the command line, particularly bash, so it's important to keep in mind.

If you're on Windows, prefer practicing on WSL over PowerShell, to get used to it.

A basic command example that may be useful: `grep`. When you're inspecting the output of some command, it might be huge, so `grep` can help you locate things. Why do I mention `grep` specifically, when there are a bunch more commands? Because I always forget it's case sensitive 😅.

```bash
cat something.yaml | grep -i something
```

The `-i` is important! 🤣

## Some random rants

Now, let's take a minute for some rants!

Besides the previous rant about the training material, there are a couple more things about the exam that annoyed me, nothing too major, but I get easily annoyed 😛.

- Need to do the exam on Chrome or Chromium - this is the tiniest of the annoyances, but still worth a mention.
- Need for a "clutter-free work area" - this makes sense of course, until it's too much! I knew this beforehand, so had already prepared the desk, but apparently it wasn't enough. Ended-up removing the speakers, removing a USB hub (granted it doesn't have the most normal shape, but still) and, worse of all, having to remove the label from the water bottle 😑.
- At some point during the exam, my connection with the proctor was interrupted, for about 20/25 minutes, pausing the exam. Things happen, so no gripe there, but it's still a bit of a stressful situation during an exam. During that time I sent a couple of emails to the provided contacts, but got no response. Thankfully, the proctor eventually got back online and I was able to continue the exam.
- Grading process taking 36 hours (if only) - it's stated in the [instructions](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad) that the results will be emailed 36 hours from the time the exam is completed. Right from the get go I don't understand that number, considering the grading is an automated task, not manual. Adding to that, I received my score after more than 48 hours. I was beginning to think I might have to do the exam again, as something could have gone wrong due to the proctor connection situation, but fortunately no, it was just late, phew 😅!

## Will I start doing more certifications?

Will I start doing more certifications? No, not really. Don't get me wrong, I don't find them useless, but it's not something I want to invest time and money on.

I'd say I have a couple of main gripes with certifications:

1. I prefer to focus on the learning side, without the rest of the certification hassle.
2. At the rate at which the technologies I normally use evolve, certifications become obsolete rather quickly.

Regarding 1, my train of thought is a bit like, school/university is done and dusted, I just want to continue to learn new things without going through the unneeded stressful part of dealing with scores, exams and all the process associated with it.

Adding to this, many times the things one needs to know or do for an exam don't perfectly match what we'd do in the real world, so instead of just focusing on learning the subject, we need to prepare in a way that will get us ready to pass the exam.

About point 2, of course it depends on the actual technologies, but I remember a couple of years ago, where I was working we were already using ASP.NET Core, but some colleagues were getting pre-Core ASP.NET certifications, don't really know what for. Those kinds of things, getting certifications for certifications sake is what gets me ranting in the first place 😛.

On this obsoletion topic, gotta say that I didn't feel it was the case with the CKAD, as it seems to be updated regularly to target the latest versions of Kubernetes.

Do I regret doing this certification? No, but mostly because of the learning part, having finally invested time in a subject I already had interest in. As for the certification itself, I'll probably push the certificate information to LinkedIn and mostly forget about it.

**Important disclaimer:** of course this only applies to my specific context. I understand that for other people or roles, certifications might play an important part in unlocking opportunities.

## Outro

Hope I didn't get you bored too much with all this small talk. A colleague poked me to share my experience, so here it is 🙂.

Didn't want to write a specifically tips & tricks post about the CKAD, as there are already plenty of those, so went with a more story telling approach with the occasional tip mixed in.

Hope any of this is useful to someone, if for nothing else, for some entertainment 🙃.

Links in the post:

- [Certified Kubernetes Application Developer (CKAD)](https://www.cncf.io/certification/ckad/)
- [The Linux Foundation Training](https://training.linuxfoundation.org/)
- [MiniKube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
- [Docker for Windows - Kubernetes](https://docs.docker.com/docker-for-windows/kubernetes/)
- [Pluralsight CKAD path](https://app.pluralsight.com/paths/certificate/certified-kubernetes-application-developer-ckad)
- [killer.sh - Kubernetes Exam Simulator](https://killer.sh/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [The Linux Foundation - Important Instructions: CKA and CKAD](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad)
- [Repository with CKAD exercises](https://github.com/dgkanatsios/CKAD-exercises)
- [Practice Enough With These 150 Questions for the CKAD Exam](https://medium.com/bb-tutorials-and-thoughts/practice-enough-with-these-questions-for-the-ckad-exam-2f42d1228552)

Thanks for stopping by, cyaz!


---
author: "Rahul Bajaj"
title: "Diving into the Open Source Ocean: A Recap of the Summit's Key Moments"
description: ""
date: 2023-05-19T08:59:28-04:00
categories: ["Event report"]
tags: ["conference", "supplychainsecurity"]
cover:
  image: "/images/ossna2023.jpg"
  alt: "ossna"
---

From May 10th to 12th, the Vancouver Convention Center came alive with the [Open Source Summit North America](https://events.linuxfoundation.org/open-source-summit-north-america/), a three-day event that brought together open source software enthusiasts from around the world. With a focus on exploring the newest trends and embracing opportunities in the ever-changing field of open source technology, this premier conference drew over two thousand attendees. In this blog post, we'll dive into my own presentation and its key points, share some highlights from the inspiring talks I attended, and take a closer look at the exciting conversations and new discoveries that unfolded at the Red Hat booth.

## Demystifying Unreproducible Builds: What, Why, and How?

Soon after the keynote, the conference dispersed into multiple tracks. I presented in the Supply Chain Security track, where I discussed the [Reproducible Builds](https://reproducible-builds.org/) initiative. My presentation covered several key topics:
1. Exploring software supply chains and the challenges of ensuring supply chain security, despite their benefits.
2. Understanding reproducible builds and how they mitigate the risk of software supply chain attacks on packages within a Linux distribution.
3. Identifying the software that should be prioritized first for reproducibility within a Linux distribution.
4. Examining the time it takes for developers to address unreproducible builds and identifying potential causes.
5. Investigating the influence of external ecosystem factors on reproducibility in package builds.

By addressing these topics, I aimed to shed light on the importance of reproducible builds for enhancing supply chain security and ensuring the integrity of software packages within the Linux ecosystem.

During my talk, I had the pleasure of meeting [Mattia Rizzolo](https://twitter.com/mapreri?lang=en), a dedicated core maintainer of the Reproducible Builds project who had traveled all the way from Italy. It was fascinating to connect with someone so deeply involved in the subject matter. We discussed my research paper, which explored reproducible builds and was recently submitted to the Journal of Empirical Software Engineering. Mattia expressed genuine interest in reviewing it and even offered his assistance in addressing any potential rebuttals that may arise during the review process.

![Mattia Rizzolo](/images/mattia.jpg "A picture with Mattia Rizzolo")

Moreover, I was thrilled when Mattia appreciated the taxonomy of issues we had developed in the paper, which aimed to identify the causes of unreproducible builds. He went a step further and suggested that I open a pull request on the Reproducible Builds website to showcase this taxonomy. It was truly gratifying to receive recognition from a respected member of the project's core team.

As our conversation continued, I couldn't resist asking Mattia about his thoughts on the future of the Reproducible Builds project. He acknowledged that at present, many projects seem more inclined towards adopting trendy technologies and focusing on new developments, such as Software Bill of Materials (SBOM) files to secure the software supply chain. However, he raised a critical question: Can we truly guarantee that the SBOM file generated at the project level will remain unchanged for the customers and end users who consume it?

## Engaging sessions that caught my attention

### Culture Change: Building an OSPO in a Proprietary Software Company - Ruth Suehle, SAS

In her talk, [Ruth](https://www.linkedin.com/in/ruthsuehle/) highlights that organizations in the software industry widely use open source software but questions their active contribution to the community. She cautions against the practice of solely forking and internally maintaining open source projects, which harms the community.

![Ruth Suehle](/images/ruth.jpg "A picture with Ruth Suehle")

Furthermore, Ruth advocates for establishing an Open Source Program Office (OSPO) team in proprietary software companies to address how they can contribute back. This team aligns values and actions with open source principles, sets guidelines, ensures compliance, and fosters collaboration. Lastly, Ruth emphasizes that embracing open source values leads to meaningful contributions and innovation, despite the time required for cultural change.

### Things I wish I had learned earlier about containers - Alex Juarez, Red Hat

During the conference, [Alex](https://www.linkedin.com/in/jandro/) delivered an informative session that delved into the fundamental aspects of containers, making it an ideal starting point for beginners in the field. He covered several essential topics, including container runtimes, namespaces, cgroups, and binding volumes in containers. Additionally, he discussed the process of inspecting container images to identify exposed ports. If you're interested in exploring the details further, you can find Alex's slides for the session here.

### Implementing the OpenSSF Best Practices Badges & Scorecards Into Your Project - CRob, Intel & David A. Wheeler, The Linux Foundation

Projects like Scorecard, operating under the OpenSSF umbrella, offer a valuable means to assess whether a software repository on GitHub adheres to important rules that mitigate supply chain attacks. These checks include the presence of a license file, the implementation of a CI pipeline for the project, and the utilization of a dependency manager such as Dependabot to analyze any vulnerabilities that may arise in current dependencies.

After the talk, I had a brief discussion with [David Wheeler](https://www.linkedin.com/in/david-a-wheeler-27798688/) regarding Scorecard's limitations in evaluating proprietary software. Some of the checks were found to be incompatible with local systems, unlike the deployed version that performs checks on multiple repositories hosted on GitHub.

![David Wheeler](/images/david.jpg "A picture with David Wheeler")

Meeting David Wheeler in person was a source of great excitement for me. His blog post in 2021 about reproducible builds served as a motivating factor for my research. In his blog, he highlighted the importance of prioritizing certain software for reproducibility over others. This passage stimulated my interest in exploring the types of software that software developers should focus on in terms of reproducibility.

## Interesting conversations at the Red Hat Booth

![Red Hat Booth](/images/rh-booth.jpg "Red Hat Booth")

During my time at the Red Hat booth, I had the opportunity to engage in thought-provoking discussions with industry professionals. 

One such encounter was with [Jerry Cooperstein](https://www.linkedin.com/in/jerry-cooperstein-6504014/), an ex-Red Hatter who played a significant role in designing learning content for Red Hat's courses and certifications. Currently, he leads efforts in designing courses and certifications for the Linux Foundation. Here are a few intriguing topics we covered:
1. Jerry expressed his skepticism towards multiple-choice question (MCQ) based certifications that lack practical application and fail to assess the practical aspects of technology. He emphasized the importance of incorporating practical evaluations into certification programs.
2. We delved into a discussion about the effectiveness of different learning mediums. Jerry firmly believed that books and written content leave a more profound impact on learners compared to video courses. According to him, entertainment should be separate from the learning process.
3. On a lighter note, Jerry shared an interesting anecdote about the placement of the control key on keyboards. He mentioned that, in the past, the control key was located where the Caps Lock key is positioned today. However, due to the strain it caused on the pinky finger, the key's placement was eventually changed.

During my conversation with the security software engineer at the [Tenable booth](https://www.tenable.com/), we delved deeper into the realm of container security. Our discussion covered various important topics, including the commonly performed security checks and best practices in the industry.

We explored the significance of following CIS benchmark guidelines, utilizing tools like Trivy or Clair to detect vulnerabilities, using muti-build container files, sanboxing containers (gVisor), and ensuring the use of security contexts for pods. These practices form the foundation for maintaining a secure container ecosystem.

Intriguingly, our conversation expanded to address what differentiates companies like Tenable in this highly competitive industry. While acknowledging the similarity in security practices, the engineer highlighted a few additional insights that set companies apart. Some of these included:

1. Zero-day vulnerability: We discussed the importance of actively monitoring and addressing zero-day vulnerabilities, which are unknown and unpatched vulnerabilities that can be exploited by attackers.
2. Lateral movement: This topic focused on understanding and mitigating the techniques used by attackers to move laterally within a network or system after gaining initial access. By identifying and preventing lateral movement, organizations can limit the impact of potential security breaches.
3. Active checking for vulnerabilities: We explored the proactive approach of actively scanning and checking for vulnerabilities in containerized applications. By actively assessing and addressing vulnerabilities, organizations can reduce the risk of exploitation.
4. Exclusion warriors: This term refers to individuals or teams responsible for managing exclusion lists or rules in security systems. Their role is to specify what should be excluded from security scans or monitoring, ensuring efficient and accurate security operations.

I had seen the dockers adoption of wasm in Kubecon last year, but did pay much attention to the same. During a conversation at the [Fermyon](https://www.fermyon.com/) both, I was introduced to webassembly. I discovered for myself that WebAssembly (often abbreviated as Wasm) is a binary instruction format and virtual machine that enables the execution of high-performance, low-level code on the web. It is a portable, safe, and efficient technology designed to run in web browsers alongside JavaScript and other web languages.

WebAssembly allows developers to compile code from various programming languages, such as C, C++, Rust, and more, into a compact binary format that can be executed by the web browser. This format is designed to be fast to load and execute, making it suitable for performance-critical applications on the web. It was a good learning experience over all.

## Conclusion

Overall, the Open Source Summit North America provided valuable insights, networking opportunities, and knowledge sharing, making it a remarkable event for open source enthusiasts and professionals alike.
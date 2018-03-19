# Adversarial Machine Learning

## Conferences
* [Evading Machine Learning Malware Detection](https://www.blackhat.com/docs/us-17/thursday/us-17-Anderson-Bot-Vs-Bot-Evading-Machine-Learning-Malware-Detection-wp.pdf), Anderson et al., Blackhat US 2017

  Use Reinforcement Learning to evade Machine Learning model, but don't break functionality.

* [EvadeML](http://evademl.org/gpevasion/), Weilin Xu, USENIX Enigma 2017

  Use Genetic Algorithm to generate evasive samples to bypass PDF detection model.

## Papers
* [Adversarial Examples Are Not Easily Detected](https://arxiv.org/pdf/1705.07263.pdf), Nicholas Carlini and David Wagner, University of California, Berkeley, 2017

In order to better understand the space of adversarial examples, we survey ten recent proposals that are designed for detection and compare their efficacy. We show that all can be defeated by constructing new loss functions. We conclude that adversarial examples are significantly harder to detect than previously appreciated, and the properties believed to be intrinsic to adversarial examples are in fact not. Finally, we propose several simple guidelines for evaluating future proposed defenses.

* [On the (Statistical) Detection of Adversarial Examples](https://arxiv.org/pdf/1702.06280.pdf), Kathrin Grosse, et al., Penn State University, 2017

* [Generative Adversarial Nets](https://papers.nips.cc/paper/5423-generative-adversarial-nets.pdf), Ian J. Goodfellow, et al. 2014

We propose a new framework for estimating generative models via an adversarial process, in which we simultaneously train two models: a generative model G that captures the data distribution, and a discriminative model D that estimates the probability that a sample came from the training data rather than G. The training procedure for G is to maximize the probability of D making a mistake.

* [Adversarial Machine Learning at Scale](https://arxiv.org/pdf/1611.01236.pdf), Alexey Kurakin, Ian Goodfellow, Samy Bengio, 2016

* [Adversarial Spheres](https://arxiv.org/pdf/1801.02774.pdf), Justin Gilmer, 2018

* [Certifying Some Distributional Robustness with Principled Adversarial Training](https://openreview.net/pdf?id=Hk6kPgZA-)

## Blogs
* [Attacking Machine Learning with Adversarial Examples](https://blog.openai.com/adversarial-example-research/)
* [对深度学习的逃逸攻击 — 探究人工智能系统中的安全盲区](http://bobao.360.cn/learning/detail/4569.html)
* [深度学习框架中的魔鬼 — 探究人工智能系统中的安全问题](http://blogs.360.cn/blog/devils-in-the-deep-learning-framework/)

## Github
* [Detecting malware outbreaks using adversarial autoencoder](https://adc.github.trendmicro.com/separk/aae), Sean Park

Recently several deep learning approaches have been attempted to detect malware binaries using convolutional neural networks and stacked deep autoencoders. Although they have shown respectable performance on a large corpus of dataset, practical defense systems require precise detection during the malware outbreaks where only a handful of samples are available. This project demonstrates the effectiveness of the latent representations obtained through the adversarial autoencoder for malware outbreak detection, which solves fine-grained unsupervised clustering problem on a limited number of unlabeled samples. Using program code distributions mapped to semantic latent vectors, the model provides a highly effective neural signature that helps detecting newly arrived malware samples mutated with minor functional upgrade, function shuffling, or slightly modified obfuscations. In order to get the cluster number, the latent representation (z) is fed to any distance based clustering algorithms such as HDBSCAN.

## Other Summary
* [Awesome Adversarial Machine Learning](https://github.com/yenchenlin/awesome-adversarial-machine-learning),  yenchenlin's summary


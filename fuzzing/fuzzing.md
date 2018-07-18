# Fuzzing

## 1. AFL-based Fuzzing

- [AFL Fuzzer Offical Site](http://lcamtuf.coredump.cx/afl/)
- [AFL Fuzzer Github](https://github.com/mirrorer/afl)

- [Project Triforce: Run AFL on Everything!](https://www.nccgroup.trust/us/about-us/newsroom-and-events/blog/2016/june/project-triforce-run-afl-on-everything/)
- [AFL/QEMU fuzzing with full-system emulation](https://github.com/nccgroup/TriforceAFL)

## 2. Grammar-based Fuzzing

- [Grammar-based Whitebox Fuzzing](http://moflow.org/ref/Grammar-based%20Whitebox%20Fuzzing.pdf)

### 2.1 Browser Fuzzing

- [Vulnerability Discovery Against Apple Safari: Evaluating Complex Software Targets for Exploitable Vulnerabilities](http://blog.ret2.io/2018/06/13/pwn2own-2018-vulnerability-discovery/)
- [Smashing	The	Browser: From Vulnerability Discovery To Exploit](https://hitcon.org/2014/downloads/P1_06_Chen%20Zhang%20-%20Smashing%20The%20Browser%20-%20From%20Vulnerability%20Discovery%20To%20Exploit.pdf)

#### 2.1.1 DOM Fuzzing

- [Domato: Google DOM Fuzzer](https://github.com/google/domato)
- [Code Coverage Explorer for IDA Pro](https://github.com/gaasedelen/lighthouse)
- [State Fuzzer](https://github.com/demi6od/ChromeFuzzer)

#### 2.1.2 JS Engine Fuzzing

- [JavaScript Fuzzing in Mozilla](https://nth10sd.github.io/js-fuzzing-in-mozilla/?full#cover)
- [dharma: A generation-based, context-free grammar fuzzer](https://github.com/MozillaSecurity/dharma)
- [funfuzz: JavaScript engine fuzzers](https://github.com/MozillaSecurity/funfuzz)

## 3. Machine Learning based Fuzzing

- [NEUZZ: Efficient Fuzzing with Neural Program Learning](https://arxiv.org/abs/1807.05620)

Fuzzing has become the de facto standard technique for finding software vulnerabilities. However, even the state-of-the-art fuzzers are not very efficient at finding hard-to-trigger software bugs. Coverage-guided evolutionary fuzzers, while fast and scalable, often get stuck at fruitless sequences of random mutations. By contrast, more systematic techniques like symbolic and concolic execution incur significant performance overhead and struggle to scale to larger programs. 
We design, implement, and evaluate NEUZZ, an efficient fuzzer that guides the fuzzing input generation process using deep neural networks. NEUZZ efficiently learns a differentiable neural approximation of the target program logic. The differentiability of the surrogate neural program, unlike the original target program, allows us to use efficient optimization techniques like gradient descent to identify promising mutations that are more likely to trigger hard-to-reach code in the target program. 
We evaluate NEUZZ on 10 popular real-world programs and demonstrate that NEUZZ consistently outperforms AFL, a state-of-the-art evolutionary fuzzer, both at finding new bugs and achieving higher edge coverage. In total, NEUZZ found 36 previously unknown bugs that AFL failed to find and achieved, on average, 70 more edge coverage than AFL. Our results also demonstrate that NEUZZ can achieve average 9 more edge coverage while taking 16 less training time than other learning-enabled fuzzers.

# Content
- [Security testing](#security-testing)
- [Static testing](#static-testing)
- [Fuzz testing](#fuzz-testing)

# Security Testing

In May of 2018, a [Code Injection Vulnerability was discovered in the Desktop version of the Signal app](https://thehackernews.com/2018/05/signal-desktop-hacking.html). An attacker could execute code on a victim's machine by sending a specially crafted message. Depending on the payload, the victim's app could be made to hand over the `/etc/passwd` file, or even send all the chats in plain text, _without any human intervention_! This was ironic since Signal is known for its end-to-end encryption feature. Since that time, [multiple vulnerabilities](https://www.cvedetails.com/vulnerability-list/vendor_id-17912/Signal.html) have been discovered in the app (specifically, its Desktop version), which Signal has been quick to fix.

*Why did these vulnerabilities exist, and what could they have done to prevent it? In this chapter, we answer these questions and introduce the concept of security testing.*

After reading this chapter, you should be able to:
- Explain the key difference between traditional software testing and security testing,
- Understand the cause of security vulnerabilities in Java applications,
- Implement the Secure Software Development Life Cycle,
- Explain the various facets of Security Testing,
- Evaluate the key differences between SAST and DAST.

## Software vs. security testing

We start this chapter with what you already know: *software testing*. The key difference between *software testing* and *security testing* is as follows:

> The goal of software testing is to check the correctness of the implemented functionality, while the goal of security testing is to find bugs (i.e. vulnerabilities) that can potentially allow an *intruder* to make the software misbehave.

Security testing is all about finding those edge cases in which a software *can be made to* malfunction. What makes a security vulnerability different from a typical software bug is the assumption that *an intruder may exploit it to cause harm*. Often, a software bug is exploited to be used as a security vulnerability, e.g. a specially crafted input that triggers a buffer overflow may be used to extract sensitive information from a system's memory. This was done in the [Heartbleed vulnerability](https://heartbleed.com/) that made $$2/3^{rd}$$ of all web servers in the world leak passwords. On the other hand, not all security vulnerabilities are software bugs, and sometimes may even be non-functional, e.g. authentication cookies never being invalidated allows a [Cross Site Request Forgery](https://owasp.org/www-community/attacks/csrf) attack, in which an attacker exploits a valid cookie to forge a victim's action.  

 Similar to traditional software testing, thorough testing *does not* guarantee the absence of security vulnerabilities. In fact, new *variants of exploits* can pop up at any time and can hit even a time-tested software. This is why security testing is not a one-off event, but has to be incorporated in the whole Software Development Life Cycle.


>> We discuss the **Secure Software Development Life Cycle** later in this chapter.

Security testers are always at an arms-race with the attackers. Their aim is to find and fix the vulnerabilities before an adversary gets the chance to exploit them.
In this sense, security testers have to protect a large *attack surface* (see figure), all at the same time, while an adversary only needs to find one entry-way (i.e. via an exploit) to defeat the defense. The goal of security testing is to limit the exposed attack surface and to increase the efforts required by attackers to exploit it.


![Representation of attack surface and exploits](img/security-testing/attack-surface.png)


## Understanding Java vulnerabilities

In this chapter, we investigate the threat landscape of Java applications because of its popularity: 3 billion devices run Java globally [according to Oracle](https://www.oracle.com/java/). Additionally, Java is memory-safe: it handles memory management and garbage collection itself, unlike C that requires developers to handle these tasks themselves. This is a common reason to assume that Java apps do not suffer from, e.g. buffer overflows. However, several core Java components are built upon native C code for optimization purposes, making them potential targets.

> The Java Virtual Machine, the sandbox that enables Java programs to execute platform-independently, is itself written in C.

In order to understand the threat landscape for Java applications, we must analyse the kind of security vulnerabilities that have been discovered over the years. There exist online repositories that consolidate such vulnerability information. The [NIST National Vulnerability Database](https://www.cvedetails.com/) is one such example.

The [National Vulnerability Database](https://www.cvedetails.com/) is the largest repository of security vulnerabilities found in open source software. Each vulnerability is assigned a unique ***CVE (Common Vulnerabilities and Exposures)*** identifier, a ***CWE (Common Weakness Enumeration)*** that determines the _type_ of vulnerability, and a ***CVSS (Common Vulnerability Scoring System)*** score that determines the _severity_ of the vulnerability. Additionally, the products including their affected versions are also listed.


### JRE vulnerabilities

![Vulnerabilities reported in JRE](img/security-testing/jre-vuln.png)


The plots show the number of vulnerabilities (left) and type of vulnerabilities (right) in the Java Runtime Environment (JRE) from 2007 to 2019. The spike in 2013 and 2014 is due to the exploitation of the *Type Confusion Vulnerability* (explained later), that allows a user to bypass the Java Security Manager and perform highly privileged actions. Functional bugs, such as uncaught exceptions, are common causes of crashing apps in Java.

### Android vulnerabilities


![Vulnerabilities reported in android](img/security-testing/android-vuln.png)


The second set of plots show vulnerabilities discovered between 2009 and 2019 in Android OS, which is mostly written in Java.

What is interesting to see in these plots is that the top 3 vulnerability types are related to *bypassing controls*, *executing code in unauthorized places*, and causing *denial of service*. Hence, we see that although memory corruption is not a major threat for Java applications, the effects caused by classical buffer overflows in C applications can still be achieved in Java by other means.

## Vulnerability use cases

Let's take the following commonly exploited vulnerabilities in Java applications, and analyse how they work:
1. Code injection vulnerability
  * Update attack
2. Type confusion vulnerability
  * Bypassing Java Security Manager
3. Buffer overflow vulnerability
4. Arbitrary Code Execution (ACE)
5. Remote Code Execution (RCE)

### Code injection vulnerability

The code snippet below has a *Code Injection* vulnerability.  

``` java

Socket socket = null;
BufferedReader readerBuffered = null;
InputStreamReader readerInputStream = null;

/*Read data using an outbound tcp connection */
socket = new Socket("host.example.org", 39544);

/* Read input from socket */
readerInputStream = new InputStreamReader(socket.getInputStream(), "UTF-8");
readerBuffered = new BufferedReader(readerInputStream);

/* Read data using an outbound tcp connection */
String data = readerBuffered.readLine();

Class<?> tempClass = Class.forName(data);
Object tempClassObject = tempClass.newInstance();

IO.writeLine(tempClassObject.toString());

// Use tempClass in some way

```

The `Class.forName(data)` is the root cause of the vulnerability. If you look closely, the object's value is loaded dynamically from `host.example.org:39544`. If the host is controlled by an attacker, they can introduce new functions or overload existing ones with their malicious code in the class that is returned. This new code becomes part of the application's logic at run-time. A famous version of this attack is an **Update attack** in Android applications, where a plugin seems benign, but it downloads malicious code at run-time. A recent example of update attack is the [CamScanner app](https://www.kaspersky.com/blog/camscanner-malicious-android-app/28156/) -- a famous phone-based document scanner app, which at some point started downloading malicious modules.


Static analysis tools often fail to detect this attack, since the malicious code is not part of the application logic at compile time.
>Due to the variations that Code Injection can present itself in, it is the top entry in the [OWASP Top 10 list of vulnerabilities](https://owasp.org/www-project-top-ten/). To limit its effect, developers can disallow 'untrusted' plugins, and can limit the privileges that a certain plugin has, e.g. by disallowing plugins to access sensitive folders.


### Type confusion vulnerability

This vulnerability was present in the implementation of the `tryfinally()` method in the *Reflection API* of the [Hibernate ORM library](https://access.redhat.com/security/cve/cve-2014-3558). Due to insufficient type checks in this method, an attacker could cast objects into arbitrary types with varying privileges.

The Type confusion vulnerability is explained in [this blog by Vincent Lee](https://www.thezdi.com/blog/2018/4/25/when-java-throws-you-a-lemon-make-limenade-sandbox-escape-by-type-confusion), from where we take the example below.

``` java
class Cast1 extends Throwable {
  Object lemon;
}

class Cast2 extends Throwable {
  Lime lime;
}

public static void throwEx() throws Throwable {
  throw new Cast1();
}

public static void handleEx(Cast2 e) {
  e.lime.makeLimenade();
}
```

Suppose that an attacker wants to execute the `makeLimenade()` method of the `Lime` object, but only has access to a `lemon` of type `Object`. The attacker exploits the fact that `throwEx()` throws a `Cast1` (lemon) object, while `handleEx()` accepts a `Cast2` (lime) object.

For the sake of brevity, consider that the output of `throwEx()` is an input to `handleEx()`. In the old and vulnerable version of Java, these type mismatches did not raise any alerts, so an attacker could send a `lemon` object that was then cast into a `Lime` type object, hence allowing them to call the `makeLimenade()` function from *(what was originally)* a `lemon`.

In a real setting, an attacker can use this _type confusion_ vulnerability to escalate their privileges by **bypassing the Java Security Manager (JSM)**. The attacker's goal is to access `System.security` object and set it to `null`, which will disable the JSM. However, the `security` field is private and cannot be accessed by just any object, e.g. let's suppose that an attacker has access to an object called `Obj`. So, they will exploit the _type confusion_ vulnerability to cast `Obj` into an object with higher privileges that has access to the `System.security` field. Once the JSM is bypassed, the attacker can execute whatever code they want to.


### Arbitrary Code Execution (ACE)

A common misconception is that Java, unlike C, is not vulnerable to **Buffer overflows**. In fact, any component implemented in native C code is just as vulnerable to exploits as the original C code would be. An interesting example here is of graphics libraries that often use native code for fast rendering.

An earlier version of a GIF library in the Sun JVM contained a memory corruption vulnerability: A valid GIF component with the block's width set to 0 caused a _buffer overflow_ when the parser copied data to the under-allocated memory chunk. This overflow caused multiple pointers to be corrupted, and resulted in **Arbitrary Code Execution** (see [CVE-2007-0243](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-0243) for more details).

A similar effect was caused by an [XML deserialization bug in the XStream library](https://access.redhat.com/security/cve/cve-2013-7285): while deserializing XML into a Java Object, a malicious XML input caused the memory pointer to start executing code from _arbitrary memory locations_ (which could potentially be controlled by an attacker).

When an ACE is triggered remotely, it is called a **Remote Code Execution** (RCE) vulnerability. The underlying principle is the same: it is also caused by *Improper handling of 'special code elements'*. We have seen it in the [Spring Data Commons](https://pivotal.io/security/cve-2018-1273) library, a part of the Spring framework that provides cloud resources for database connections.

An Oracle report in 2018 stated that [most of the Java vulnerabilities can be remotely exploited](https://www.waratek.com/alert-oracle-guidance-cpu-april-2018/). With 3 billion devices running Java, this creates a large attack surface.


## The Secure Software Development Life Cycle (Secure-SDLC)

Security testing is a type of non-functional testing, but if it fails to fix security vulnerabilities, (i) there is a high impact on the functionality of the application, e.g. a denial of service attack that makes the entire application unreachable, and (ii) it also causes reputation and/or monetary loss, e.g. loss of customers.

There is an interesting debate about *who gets the responsibility for security testing*.
The pragmatic approach is to **include security testing in each phase of the SDLC**.

The figure below shows the Secure-SDLC variant of the traditional SDLC taken from [this article by Rohit Singh](https://www.dignitasdigital.com/blog/easy-way-to-understand-sdlc/).


![The secure SDLC](img/security-testing/ssdlc.png)


At the *planning phase*, risk assessment should be done and potential abuse cases should be designed that the application will be protected against. In the *analysis phase*, the threat landscape should be explored, and attacker modelling should be done.

>An example of an attacker model is that the vendor supplying app-plugins has been infected, so all plugins received from the vendor might be malicious.

The *design* and *implementation* plans of the application should include insights from the attacker model and abuse cases.

>For example, the choice of certain libraries, and the permissions assigned to certain modules should be guided by the threat landscape under consideration.

Security testing should be a part of the *testing and integration* phases. Code reviews should also be done from the perspective of the attacker (using abuse cases). Finally, during the *maintenance phase*, in addition to bug fixes, developers should keep an eye on the CVE database and update *(if possible)* the vulnerable components in the application.

*Just like the traditional SDLC is not a one-time process, the Secure-SDLC is also a continuous process*. Therefore, security testing should also be integrated into the *Continuous Integration* framework as well.

Currently, most companies solely do *penetration testing* which tests the entire application at the very end of the SDLC. The problem with penetration testing is that it tests the application as a whole, and does not stress-test each individual component. When security is not an integral part of the design phase, the vulnerabilities discovered in the penetration testing phase are patched in an ad-hoc manner that increase the risk of them falling apart after deployment (*e.g. refer to the Signal app's desktop version*).



## Facets of security testing

As such, the term *security testing* is very broad and it covers a number of overlapping concepts. We classify them as follows:

|         |    White-box    |    Black-box    |
|------------|----------------------------------------------------|-------------------------------------------------------------------------|
|    **Static Application Security Testing**    | Code checking, Pattern matching, ASTs, CFGs,  DFDs  |  Reverse engineering|
|    **Dynamic Application Security Testing**    | Tainting, Dynamic validation, Symbolic  execution | Penetration testing,  Reverse engineering, Behavioural analysis,  Fuzzing |

You should already be familiar with _white/black-box_ testing, static testing and some of the mentioned techniques.


>> In this chapter, we specifically focus on how to use these techniques to find security vulnerabilities.


In the context of automated security testing, _static_ and _dynamic_ analysis are called _Static Application Security Testing (SAST)_ and _Dynamic Application Security Testing (DAST)_, respectively.
Before we dive into further explanation of SAST and DAST techniques, let's look at the assessment criteria for evaluating the quality of security testing techniques.

### Quality assessment criteria
The quality of testing tools is evaluated in a number of ways. You have already learnt about _code coverage_ in a previous chapter. Here, we discuss four new metrics that are often used in the context of security testing.

Designing an ideal testing tool requires striking a balance between two measures: (a) Soundness, and (b) Completeness.
**Soundness** dictates that there should be no False Negatives (FN) — no vulnerability should be skipped. This implies that no alarm is raised *IF* there is no existing vulnerability in the *System Under Test (SUT)*. **Completeness** dictates that there should be no False Positives (FP) — no false alarm should be raised. This implies that an alarm is raised *IF* a valid vulnerability is found.

>Here, a 'positive' instance indicates a _bug_ and a 'negative' instance indicates _benign code_. So, True Positives (TP) are _actual bugs_, and True Negatives (TN) are _actual benign code snippets_. Similarly, False Positives (FP) are _false bugs_ (or _false alarms_), and False Negatives (FN) are _bugs that weren't found_ (or _missed bugs_).

A perfect testing tool is both sound and complete. However, this is an undecidable problem — given finite time, the tool will always be wrong for some input. In reality, tools often compromise on either FPs or FNs.

Low FNs are ideal for security critical applications where a missed vulnerability can cause significant loss, e.g. banking apps. Low FPs are ideal for applications that do not have a lot of manpower to evaluate the correctness of each result.

Additionally, an ideal testing tool is (c) **interpretable**: an analyst can trace the results to a solid cause, and are (d) **scalable**: the tool can be used for large applications without compromising heavily on performance.



## Static Application Security Testing (SAST)


SAST techniques aim to find security bugs without running the application. They can find bugs that can be observed in the source code and for which _signatures_ can be made, e.g. SQL Injection and basic Cross-Site Scripting. _SpotBugs_, _FindSecBugs_, and _Coverity_ are static analysis tools specially meant to test security problems in applications.

SAST is not only limited to code checking — it includes any approach that does not require running the SUT. For example, **Risk-based testing**  is a business-level process where we model the worst-case scenarios (or abuse cases) using threat modelling. An application is tested against the generated abuse cases to check its resilience against them. Risk-based testing can be done both statically (if the abuse-case targets problems found in source code) and dynamically (for run-time threats).

Below, we discuss SAST techniques and provide examples of the security bugs that they can find.

1. Code checking for security
  * Pattern matching via _RegEx_
  * Syntax analysis via _Abstract Syntax Trees_
3. Structural testing for security
  * Control Flow Graphs (CFGs)
  * Data Flow Diagrams (DFDs)

### Code checking for security

#### Pattern matching via RegEx
Pattern matching can find simplistic security issues, such as:
1. *Misconfigurations*, like `port 22` being open for every user,
2. *Potentially bad imports*, like importing the whole `System.IO` namespace,
3. *Calls to dangerous functions*, like `strcpy` and `memcpy`.

#### Syntax analysis via AST

Abstract Syntax Trees can also find security misconfigurations in the codebase, and sometimes are more appropriate than using regular expressions. Code that violates security specifications can be detected by walking the AST. For example, for a rule specifying how the print function should be used, i.e. `printf(format_string, args_to_print)`, and the following code snippet, an error will be raised because of a missing parameter that can be detected by counting the child-nodes.

![AST rule enforcement](img/security-testing/ast-usecase2.png)


This is an example of the famous *[Format string attack](https://owasp.org/www-community/attacks/Format_string_attack)*, which exploits a vulnerability in the `printf()` function family: in the absence of a format string parameter like `%s`, an attacker can supply their own format string parameter in the input, which will be evaluated as a pointer resulting in either arbitrary code execution or a denial of service.


### Structural testing for security

#### Control Flow Graphs (CFGs)
Recall that **Control Flow Graphs** show how the control is transferred among different pieces of code in an application. A CFG is an *over-estimation* of what any potential execution might look like — it is the union of all possible combinations of execution paths.

For security testers, a CFG is an overall picture of an application's behaviour, in a graphical format. It can help testers pin-point strange control transfers, e.g.
* *an unintended transition going from a low- to a high- privileged code block*, or
* *certain unreachable pieces of code* that can result in application hanging and eventually, a denial of service.

Existing literature has used CFGs to detect the maliciousness of an application based on how its CFG looks like. For example, [this work by Bruschi _et al._](https://link.springer.com/chapter/10.1007/11790754_8) detects self-mutating malware by comparing its CFG with CFGs of known malware, and [this work by Sun _et al._](https://link.springer.com/chapter/10.1007/978-3-642-55415-5_12) uses CFGs to measure code reuse as a means to detect malware variants.

#### Data Flow Diagram (DFD)

A DFD is built on top of a CFG and shows how data traverses through a program. Since Data Flow Analysis (DFA) is also a static approach, a DFD tracks all possible values a variable might have during any execution of the program. This can be used to detect _sanitization problems_, such as the deserialization vulnerability that caused an ACE, and some simplistic _code injection vulnerabilities_.

**How DFA works:** A user-controlled variable whose value we intend to track is called a ***Source***, and the other variables are called ***Sinks***. We say that the *Source variables are tainted* and *all Sinks are untainted*. For a Source to be connected to a Sink, it must first be untainted by proper input validation, for example.

In DFA, we prove that (a) _No tainted data is used_, and (b) _No untainted data is expected_. An alert is raised if either of the two conditions are violated. Consider the following scenario for a single Source and Sink. There exists a direct path between a Source and a Sink, which violates the first rule. This indicates that a malicious user input can cause an SQL injection attack. The solution to fix this violation is to include an input clean-up step between the Source and Sink variables.



![Source/sink example](img/security-testing/source-sink-example.png)

The code snippet below shows a real case that DFA can detect. The variable `data` is tainted, as it is received from a user. Without any input cleaning, it is directly used in `println()` method that expects untainted data, thus raising an alert.


``` Java
/* Uses bad source and bad sink */
public void bad(HttpServletRequest request, HttpServletResponse response)
  throws Throwable {

  String data;

  /* Potential flaw: Read data from a queryString using getParameter */
  data  = request.getParameter("name");

  if (data != null) {
    /* Potential flaw: Display of data in web pages after using
    * replaceAll() to remove script tags,
    * will still allow XSS with string like <scr<script>ipt>. */
    response.getWriter().println("<br>bad(): data = " +
    	data.replaceAll("(<script>)", ""));
  }
}

```



>> A dynamic version of Data Flow Analysis is called Taint analysis, where the tainted variables' values are tracked in memory. We cover it in the DAST section of this chapter. 



##### Reaching Definitions Analysis
 One application of DFA is called the **Reaching Definitions**. It is a top-down approach that identifies all the possible values of a variable. For security purposes, it can detect the _Type Confusion vulnerability_ and _Use-after-free vulnerability_ (which uses a variable after its memory has been freed).

 Consider the following code snippet and its CFG given below:


``` java
int b = 0;
int c = 1;

for (int a = 0; a < 3; a++) {
  if (a > 1)
    b = 10;
  else
    c = b;
}
return b, c;
```

![Making a DFD](img/security-testing/dfd-code2.png)


The solid transitions show *control transfers*, while the dotted transitions show *data transfers*. Suppose that we want to perform reaching definitions analysis of the three variables: `a`, `b`, and `c`. First, we label each basic block, and draw a table that lists the variable values in each block.

![Performing reaching definitions analysis](img/security-testing/dfd-code3.png)


If a variable is not used in the block, or the value remains the same, nothing is listed. At the end, each column of the table is merged to list the full set of potential values for each variable.


| code blocks | **a** | **b** | **c** |
|:----:|:--------:|:----:|:---:|
| **b1** | - | 0 | 1 |
| **b2** | 0, **a**++ | - | - |
| **b3** | - | - | - |
| **b4** | - | 10 | - |
| **b5** | - | - | **b** |
| **b6** | - | - | - |

Remember, if the value of a variable is controlled by a user-controlled parameter, it cannot be resolved until run-time, so it is written as is.
If a variable `X` copies its value to another variable `Y`, then the reaching definitions analysis dictates that the variable `Y` will receive all potential values of `X`, once they become known.
Also, whether a loop terminates is an undecidable problem (also called the *halting problem*), so finding the actual values that a looping variable takes on is not possible using static analysis.

The analysis results in the following values of the three variables. If you look closely, some values are impossible during actual run-time, but since we trace the data flow statically, we perform an over-estimation of the allowed values. This is why, static analysis, in particular DFA, is `Sound` but `Incomplete`.

``` java
a = {0, 1, 2, 3, ...}       // 3 and above are impossible
b = {0, 10}                // 0 is impossible       
c = {1, b} -> {0, 1, 10}  // 1, 10 are impossible
```

## Dynamic Application Security Testing (DAST)

Application crashes and hangs leading to Denial of Service attacks are common security problems. They cannot be detected by static analysis since they are only triggered when the application is executed.
DAST techniques execute an application and observe its behaviour. Since DAST tools typically do not have access to the source code, they can only test for functional code paths, and the analysis is only as good as the behaviour triggering mechanism. This is why search-based algorithms have been proposed to maximize the code coverage, e.g. see [this work by Gross _et al._](https://dl.acm.org/doi/abs/10.1145/2338965.2336762) and [this work by Chen _et al._](https://ieeexplore.ieee.org/abstract/document/8418633).

DAST tools are typically difficult to set up, because they need to be hooked-up with the SUT, sometimes even requiring to modify the SUT's codebase, e.g. for instrumentation. They are also slower because of the added abstraction layer that monitors the application's behaviour. Nevertheless, they typically produce less false positives and more advanced results compared to SAST tools. Even when attackers obfuscate the codebase to the extent that it is not statically analysable anymore, dynamic testing can still monitor the behaviour and report strange activities. _BurpSuite_, _SonarQube_, and _OWASP's ZAP_ are some dynamic security testing tools.

In this section, we explain the following techniques for dynamic analysis:

1. Taint analysis
2. Dynamic validation
3. Penetration testing
4. Behavioural analysis
5. Reverse Engineering
6. Fuzzing

### Taint analysis

Taint analysis is the dynamic version of Data Flow Analysis. In taint analysis, we track the values of variables that we want to *taint*, by maintaining a so-called *taint table*. For each tainted variable, we analyse how the value propagates throughout the codebase and affects other statements and variables. To enable tainting, ***code instrumentation*** is done by adding hooks to variables that are of interest. _Pin_ is an instrumentation tool from Intel, which allows taint analysis of binaries.

An example here is of the tool [Panorama](https://dl.acm.org/doi/abs/10.1145/1315245.1315261) that detects malicious software like _keyloggers_ (that log keystrokes in order to steal credentials) and _spyware_ (that stealthily collects and sends user data to $$3^{rd}$$ parties) using dynamic taint analysis. Panorama works on the intuition that benign software does not interfere with OS-specific information transfer, while information-stealing malware attempts to access the sensitive information being transferred. Similarly, malicious plugins collect and share user information with $$3^{rd}$$ parties while benign plugins do not send information out. These behaviours can be detected using the source/sink principles of taint analysis.  

### Dynamic validation

Dynamic validation does a functional testing of the SUT based on the system specifications. It basically checks for any deviations from the pre-defined specifications. **Model Checking** is a similar idea in which specifications are cross-checked with a model that is learnt from the SUT. Model checking is a broad field in the Software Verification domain.

[This work by Chen _et al._](http://seclab.cs.ucdavis.edu/papers/Hao-Chen-papers/ndss04.pdf) codifies security vulnerabilities as _safety properties_ that can be analysed using model checking. For example, they analyse processes that may contain _race conditions_ that an attacker may exploit to gain control over a system. In this regard, consider the following code snippet. Suppose that the process first checks the owner of `foo.txt`, and then reads `foo.txt`. An attacker may be able to introduce a race condition in between the two statements and alter the symbolic link of `foo.txt` such that it starts referring to `/etc/passwd` file. Hence, what the user reads as `foo.txt` is actually the `/etc/passwd` file that the attacker now has access to.


```java
Files.getOwner("foo.txt");
Files.readAllLines("foo.txt");
```

To check the existence of such scenarios, they codify it in a property that _stops a program from passing the same filename to two system calls on the same path_. Once codified in a model checker, they run it on various applications and report on deviations from this property.   

>> It is important to note that not all security properties can be codified into specifications. Additionally, such specifications need to be updated regularly to detect new vulnerabilities and to reduce false alarms.


### Penetration testing

Penetration (or Pen) testing is the most common type of security testing for organizations. It is sometimes also referred to as ***Ethical hacking***. What makes pen testing different from others is that it is done from the perspective of an attacker — [pen-testers attempt to breach the security of the SUT just as an adversary might](https://www.ncsc.gov.uk/guidance/penetration-testing). Since it is done from the perspective of the attacker, it is generally black-box, but depending on the assumed knowledge of the attacker, it may also be white-box.

Penetration testing checks the SUT in an end-to-end fashion, which means that it is done once the application is fully implemented, so it can only be done at the end of the SDLC. *MetaSploit* is an example of a powerful penetration testing framework. Most pen testing tools contain a ***Vulnerability Scanner*** module that either *runs existing exploits*, or allow the tester to *create an exploit*. They also contain ***Password Crackers*** that either *brute-force* passwords (i.e. try all possible combinations given some valid character set), or perform a *dictionary attack* (i.e. choose inputs from pre-existing password lists).

### Behavioural analysis

Given a software that may contain modules from unknown sources, behavioural analysis aims to gain insights about the software by generating behavioural logs and analysing them. This can be particularly helpful for finding abnormal behaviours (security problems, in particular) when neither the source code, nor the binary are accessible. The logs can be compared with known-normal behaviour in order to debug the SUT.

An example here is of JPacman that currently has support for two point calculator modules (`Scorer 1` and `Scorer 2`) that calculate the score in different ways. The goal is to find what the malicious module (`Scorer 2`) does. We have implemented an automated agent (called *fuzzer*) that runs various instances of JPacman to generate behavioural logs. At each iteration, it randomly picks a move (from the list of acceptable moves) until Pacman dies, and logs the values of interesting variables. The agent's code is given below.

```java
  /**
   * Basic fuzzer implementation.
   *
   * @param repetitionInfo repeated test information
   * @throws IOException when the log write created.
   */
  @RepeatedTest(RUNS)
  void fuzzerTest(RepetitionInfo repetitionInfo) throws IOException {
      Game game = launcher.getGame();
      Direction chosen = Direction.EAST;

      String logFileName = "log_" + repetitionInfo.getCurrentRepetition() + ".txt";
      File logFile = new File(logDirectory, logFileName);

      try (BufferedWriter logWriter = new BufferedWriter(new OutputStreamWriter(
          new FileOutputStream(logFile, true), StandardCharsets.UTF_8))) {

          logWriter.write(LOG_HEADER);

          try {
              game.start();

              while (game.isInProgress()) {
                  chosen = getRandomDirection();

                  log(logWriter, chosen);
                  game.getLevel().move(game.getPlayers().get(0), chosen);
              }
          } catch (RuntimeException e) {
              // Runtime exceptions should not stop the execution of the fuzzer
          } finally {
              log(logWriter, chosen);
              game.stop();
          }
      }
  }

```

Below you see an example of a log file resulting from one run of the fuzzer.

![behavioural log screenshot](img/security-testing/behav-log-screenshot.png)


In the figure below, the plots on the right show how the value of `score` variable changes over time. It is apparent that something goes wrong with `Scorer 2`, since the score is typically programmed to increase monotonically.

![JPacman and scorers](img/security-testing/jpacman-screenshot.png)


Behavioural logs are a good data source for forensic analysis of the SUT.

### Reverse Engineering (RE)

Reverse Engineering is a related concept where the goal is to reveal the internal structure of an application. We can consider RE as a system-wide process that converts a black-box application into a white-box. RE is, strictly speaking, not a testing technique, but it is useful when (i) converting legacy software into a modern one, or (ii) understanding a competitor's product.

A use case for the behavioural logs from the previous technique is to use them for automated reverse engineering that learns a model of the SUT. This model can then be used for, e.g. *Dynamic validation*, and/or to *guide path exploration* for better code coverage.

For example, [TABOR](https://dl.acm.org/doi/abs/10.1145/3196494.3196546) learns a model of a water treatment plant in order to detect attacks. They learn an automaton representing the _normal behaviour_ of the various sensors present in the plant. Anomalous incoming events that deviate from the normal model raise alerts.

### Fuzzing

Fuzzing has been used to uncover previously-unknown security bugs in several applications. _American Fuzzy Lop (AFL)_ is an efficient security fuzzer that has been used to find security vulnerabilities in command-line-oriented tools, like PuTTY, openSSH, and SQLite. [This video](https://www.youtube.com/watch?v=ibjkz7GTT3I) shows how to fuzz _ImageMagick_ using AFL to find security bugs.

Additionally, in 2015 a severe security vulnerability, by the name of [Stagefright](https://blog.zimperium.com/experts-found-a-unicorn-in-the-heart-of-android/), was discovered in Android smartphones that impacted 1 billion devices. It was present in the library for unpacking MMS messages, called `libstagefright`. A specially crafted MMS message could silently cause an overflow leading to remote code execution and privilege escalation. It was discovered by [fuzzing the `libstagefright` library using AFL](https://www.blackhat.com/docs/us-15/materials/us-15-Drake-Stagefright-Scary-Code-In-The-Heart-Of-Android.pdf) that ran \~3200 tests per second for about 3 weeks!

Finally, [Sage](https://dl.acm.org/doi/pdf/10.1145/2090147.2094081) is a white-box fuzzing tool that combines symbolic execution with fuzzing to find deeper security bugs. They report a use case of a critical security bug that black-box fuzzing was unable to find, while Sage discovered it in under 10 hours, despite having no prior knowledge of the file format.

## Performance evaluation of SAST and DAST

* Static analysis tools create a lot of FPs because they cannot see the run-time behaviour of the code. They also generate FNs if they don't have access to some code, e.g. code that is added at run-time.
* Dynamic analysis reduces both FPs and FNs — if an action is suspicious, an alarm will be raised. However, even in this case, we cannot ensure perfect testing.  
* Static analysis is generally more white-box than dynamic analysis, although there are interpretable dynamic testing methods, like symbolic execution.
* Static testing is more scalable in the sense that it is faster, while black-box dynamic testing is more generalizable.

## Chapter Summary

- Software testing checks correctness of the software, while security testing finds potential defects that may be exploited by an intruder.
- Even though Java is memory-safe, Java applications still have a large attack surface.
- Security testing is much more than penetration testing, and must be integrated at each step of the SDLC.
- Threat modelling can derive effective test cases.
- Perfect (security) testing is impossible.
- SAST checks the code for problems without running it, while DAST runs the code and monitors its behaviour to find problems.
- SAST is fast, but generates many false positives. DAST is operationally expensive, but generates insightful and high-quality results.
- Pattern matching finds limited but easy to find security problems.
- ASTs make the code structure analysis easy. CFGs and DFDs are better at finding security vulnerabilities.
- Behavioural logs are useful for forensic analysis of the SUT.
- Combining fuzzing with symbolic execution leads to finding optimal test cases that can maximize code coverage and find maximum security problems.

## Exercises

**Exercise 1.** Why is security testing hard?
1. To find a vulnerability, you must fully understand assembly code.
2. Pointers are hard to follow since they can point to any memory location.
3. Hacking is difficult and requires many years of specialization.
4. There exist a lot of strange inputs, and only few trigger a vulnerability.

**Exercise 2.** Give an example of a software bug that also qualifies as a security vulnerability, and also explain how.

**Exercise 3.** Static analysis can detect some classes of injection vulnerabilities more reliably than others. Which vulnerability is static analysis most likely to _prevent_?
1. Cross-Site Scripting
2. Cross-Site Request Forgery
3. Update attack
4. Format string injection

**Exercise 4.** What is the underlying defect that enables arbitrary code execution?
1. Buffer overflows
2. Deserializing bugs
3. Type confusion
4. All of the above

**Exercise 5.** In the following table, several testing objectives and techniques are given. Associate each _Objective_ with the most appropriate _Testing technique_. Note that no repetitions are allowed.

| Objective | Testing Technique | (Answer option) Testing Technique |
|----------------------------------------|------------------------|-----------------------------------|
| 3. Detect pre-defined patterns in code | A. Fuzzing | __ |
| 2. Branch reachability analysis | B. Regular expressions | __ |
| 1. Testing like an attacker  | C. Symbolic execution | __ |
| 4. Generate complex test cases | D. Penetration testing | __ |

**Exercise 6.** How can you use Tainting to detect spyware?

**Exercise 7.** Perform Reaching Definitions Analysis on the following piece of code. Which values of the variables does the analysis produce?
``` java
        int x = 0;
        int y = 1;
        while(y < 5) {
    	    y++;
        	if(y > 7)
        		x = 12;
        	else
        		x = 21;
    	}
        return x;
```
1. $$x = \{0\}; y = \{0,1,2,3,...\}$$
2. $$x = \{0,21\}; y = \{1,2,3,...\}$$
3. $$x = \{0,12,21\}; y = \{1,2,3,...\}$$
4. $$x = \{21\}; y = \{1,2,3,4\}$$

## References
* Bruschi, Danilo, Lorenzo Martignoni, and Mattia Monga. "Detecting self-mutating malware using control-flow graph matching." In International conference on detection of intrusions and malware, and vulnerability assessment, pp. 129-143. Springer, Berlin, Heidelberg, 2006.
* Chen, Hao, Drew Dean, and David A. Wagner. "Model Checking One Million Lines of C Code." In NDSS, vol. 4, pp. 171-185. 2004.
* Chen, Peng, and Hao Chen. "Angora: Efficient fuzzing by principled search." In 2018 IEEE Symposium on Security and Privacy (SP), pp. 711-725. IEEE, 2018.
* Godefroid, Patrice, Michael Y. Levin, and David Molnar. "SAGE: whitebox fuzzing for security testing." Queue 10, no. 1 (2012): 20-27.
* Gross, Florian, Gordon Fraser, and Andreas Zeller. "Search-based system testing: high coverage, no false alarms." In Proceedings of the 2012 International Symposium on Software Testing and Analysis, pp. 67-77. 2012.
* Lin, Qin, Sridha Adepu, Sicco Verwer, and Aditya Mathur. "TABOR: A graphical model-based approach for anomaly detection in industrial control systems." In Proceedings of the 2018 on Asia Conference on Computer and Communications Security, pp. 525-536. 2018.
* Sun, Xin, Yibing Zhongyang, Zhi Xin, Bing Mao, and Li Xie. "Detecting code reuse in android applications using component-based control flow graph." In IFIP international information security conference, pp. 142-155. Springer, Berlin, Heidelberg, 2014.
* Yin, Heng, Dawn Song, Manuel Egele, Christopher Kruegel, and Engin Kirda. "Panorama: capturing system-wide information flow for malware detection and analysis." In Proceedings of the 14th ACM conference on Computer and communications security, pp. 116-127. 2007.


# Static testing

Static testing analyses the code characteristics without executing the application. It can be considered as an automated code review. It checks the style and structure of the code, and can be used to _statically_ evaluate all possible code paths in the System Under Test (SUT).
Static analysis can quickly find _low-hanging fruit_ bugs that can be found in the source code, e.g. using deprecated functions. Static analysis tools are scalable and generally require less time to set up. _PMD_, _Checkstyle_, _Checkmarx_ are some common static analysis tools.

The classical approach underlying static analysis is checking the code for potential structural and/or stylistic rule violations. A code checker typically contains a parser and an acceptable rule set. We look at the following techniques for static analysis:

1. Pattern matching via *Regular expressions*
2. Syntax analysis via *Abstract Syntax Trees*


## Pattern matching

Pattern matching is a code checking approach that searches for pre-defined patterns in code. A common way to represent a pattern is via **regular expressions** or RegEx. Simply put, a regex engine reads input character-by-character and upon every character-match, progresses through the regular expression until no more characters remain. In case a match cannot be found, the regex engine backtracks and tries alternate paths to find a match. Depending on the logic, either a positive/negative reaction is returned indicating whether a match was found or not, or all matching inputs are returned.

An easy way to visualize a regular expression is via ***Finite State Automaton***. Each _node_ represents a state. We move from one state to the next by taking the _transition_ that matches the input symbol. Below you see a few examples of regular expressions and their corresponding finite state automata. The node with a black arrow is called the _starting state_. _Green states_ are accepting states. Any state other than accepting states is a rejecting state, e.g. _red_ and _gray states_ are rejecting states in the examples below. Note that while traversing the automaton, if the final state is not an accepting state, the string is considered rejected.

The automaton for the regular expression '**bug**' is shown below. `.` is a wildcard character that can match any possible character. An input string `bug` will transition from left to right, until we end up in the green state. However, the string `bag` will move from first state to the second state, and then to the red state. Since there is no transition out of this state, we will stay here until the input finishes. Similarly, the string `bu` will also be rejected since its final state is not a green state.



![FSM for bug](img/security-testing/regex1.PNG)


The regular expression '**.\*bug**' results in the following automaton. Again, `.` matches any possible character, and `*` denotes *0 or many times*. Hence, the regex accepts any string that ends in _bug_. The following strings will be accepted by this pattern: `bug`, `this is a bug`, `bad bug`, and `bugbug`. `bug!` will be rejected by this pattern. Note that this is a non-deterministic automata since there are two possible transitions for the symbol `b` in the first state.



![FSM for .*bug](img/security-testing/regex2.PNG)


The automaton for '**.\*bug.\***' is given below. It will accept any string that contains `b`, `u`, `g` consecutively, at least once. In this case, even `bug!` will be accepted.


![FSM for .*bug.*](img/security-testing/regex3.PNG)

While regular expressions are a fast and powerful pattern matching technique, their biggest limitation is that they do not take semantics into account, ending up with many matches that are not meaningful. For example, consider the following code snippet. Suppose that the regular expression, `\s*System.out.println\(.*\);`, searches for all print statements in the code to remove them before deployment. It will find three occurrences in the code snippet, which are all meaningless because the code is already disabled by a flag.

```java
boolean DEBUG = false;

if (DEBUG){
  System.out.println("Debug line 1");
  System.out.println("Debug line 2");
  System.out.println("Debug line 3");
}
```

## Syntax analysis

A more advanced code checking approach is syntax analysis. It works by deconstructing the input into a stream of characters, that are eventually turned into a Parse Tree. _Tokens_ are hierarchical data structures that are put together according to the code's logical structure.

![Parser pipeline](img/security-testing/lexer.png)


A **Parse Tree** is a concrete instantiation of the code, where each character is explicitly placed in the tree, whereas an **Abstract Syntax Tree (AST)** is an abstract version of the parse tree in which syntax-related characters, such as semi-colon and parentheses, are removed. An example of the AST of the code snippet above is given below.


![AST example](img/security-testing/ast-example.png)


A static analysis tool using syntax analysis takes as input (a) an AST, and (b) a rule-set, and raises an alarm in case a rule is violated.
For example, for a rule _allowing at most 3 methods_, and the following code snippet, the AST will be parsed and an error will be raised for violating the rule. Contrarily, a very complicated regular expression would be needed to handle the varying characteristics of the four available methods, potentially resulting in mistakes.


![AST rule enforcement](img/security-testing/ast-usecase1.png)


Abstract Syntax Trees are used by compilers to find semantic errors &mdash; compile-time errors associated to the _meaning_ of the program. ASTs can also be used for program verification, type-checking, and translating the program from one language to another.  

## Performance of static analysis

 Theoretically, static analysis produces _sound_ results, i.e. zero false negatives. This is because a static analysis tool has access to the whole codebase, and it can track all the possible execution paths a program might take. So, if there are _any_ vulnerabilities, the tool should be able to find them. However, this comes at the cost of _Completeness_. Because it tracks all possible execution paths without seeing how the application behaves in action, some of the results might never be reached in an actual execution scenario, resulting in false positives.

 >Note that in practice, static analysis tools _can produce unsound results_, e.g. if a piece of code is added at runtime, since the tool will fail to see the new code-piece. This is one reason why the results of static analysis cannot always be trusted, especially in a security context.  


>> _Soundness_ and _Completeness_ are defined more extensively in the Security testing chapter. 

## Exercises

**Exercise 1.** Regular expressions _CANNOT DO_ which of the following tasks?
1. Matching patterns
2. Detect semantics
3. Define wild cards
4. Detect coding mistakes

**Exercise 2.** Given that a static analysis tool has access to the entire codebase prior to execution, what is the quality of results that the tool will produce?
1. Sound and Complete
2. Sound but Incomplete
3. Unsound but Complete
4. Unsound and Incomplete

**Exercise 3.** Create an Abstract Syntax Tree for the following code snippet:
```java
(a + b) * (c - d)
```


## References

* Grune, D., Van Reeuwijk, K., Bal, H.E., Jacobs, C.J. and Langendoen, K., 2012. Modern compiler design. Springer Science & Business Media.
* Abstract Syntax Trees. https://en.wikipedia.org/wiki/Abstract_syntax_tree
* Semantic analysis. https://www.cs.usfca.edu/~galles/cs414S05/lecture/old2/lecture7.java.pdf
* Regular expressions in Java. https://www.tutorialspoint.com/java/java_regular_expressions.htm

# Fuzz testing

Fuzzing is a popular dynamic testing technique used for automatically generating complex test cases.
Fuzzers bombard the System Under Test (SUT) with randomly generated inputs in the hope to cause crashes.
A crash can either originate from *failing assertions*, *memory leaks*, or *improper error handling*.
Fuzzing has been successful in discovering [unknown bugs](https://lcamtuf.coredump.cx/afl/) in software.

>> Note that fuzzing cannot identify flaws that do not trigger a crash.

**Random fuzzing** is the most primitive type of fuzzing, where the SUT is considered as a completely black box, with no assumptions about the type and format of the input.
It can be used for exploratory purposes, but it takes a long time to generate any meaningful test cases.
In practice, most software takes some form of _structured input_ that is pre-specified, so we can exploit that knowledge to build more efficient fuzzers.


There are two ways of generating fuzzing test cases:

1. **Mutation-based Fuzzing** creates permutations from example inputs to be given as testing inputs to the SUT. These mutations can be anything ranging from *replacing characters* to *appending characters*. Since mutation-based fuzzing does not consider the specifications of the input format, the resulting mutants may not always be valid inputs. However, it still generates better test cases than the purely random case. _ZZuf_ is a popular mutation-based application fuzzer that uses random bit flipping. _American Fuzzy Lop (AFL)_ is a fast and efficient security fuzzer that uses genetic algorithms to increase code coverage for finding better test cases. Similarly, _Jazzer_ is a new Java fuzzer, which integrates into the OSS-Fuzz project managed by Google. Jazzer builds upon libFuzzer, which also uses genetic algorithms to improve coverage.

2. **Generation-based Fuzzing**, also known as *Protocol fuzzing*, takes the file format and protocol specification of the SUT into account when generating test cases. Generative fuzzers take a data model as input that specifies the input format, and the fuzzer generates test cases that only alter the values while conforming to the specified format. For example, for an application that takes `JPEG` files as input, a generative fuzzer would fuzz the image pixel values while keeping the `JPEG` file format intact. _PeachFuzzer_ is an example of a generative fuzzer.

Compared to mutative fuzzers, generative fuzzers are _less generalisable_ and _more difficult to set up_ because they require input format specifications.
However, they produce higher-quality test cases.


## Maximising code coverage

One of the challenges of effective software testing is to generate test cases that not only _maximise the code coverage, but do so in a way that tests for a wide range of possible values_.
Fuzzing helps achieve this goal by generating wildly diverse test cases.
For example, [this blog post by Regehr](https://blog.regehr.org/archives/896) describes the use of fuzzing in order to optimally test an ADT implementation.

Fuzzers can be used in a variety of ways to achieve high code coverage in a reasonable time:

1. Multiple tools
2. Telemetry as heuristics
3. Symbolic execution


### Multiple tools
A simple yet effective way to cover large parts of the codebase in a short time is to use multiple fuzzing tools.
Each fuzzer performs mutations in a different way, so they can be run together to cover different parts of the search space in parallel.
For example, using a combination of a mutative and generative fuzzer can help to generate diverse test cases while also ensuring valid inputs.

### Telemetry as heuristics
If the code structure is known (i.e., in a white-box setting), telemetry about code coverage can be used to halt fuzzing prematurely. While telemetry data does not directly help in generating valid test cases, it helps in selecting only those mutations that increase code coverage.
For example, for the `if` statement in the following code snippet, a heuristic based on ***branch coverage*** requires 3 test cases to fully cover it, while one based on ***statement coverage*** requires 2 test cases.
Using branch coverage ensures that all three branches are tested at least once. Upon the generation of such test cases, the fuzzer can stop.


```java
public String func(int a, int b){
    a = a + b;
    b = a - b;
    String str = "No";
    if (a > 2)
        str = "Yes!";
    else if (b < 100)
        str = "Maybe!";
    return str;
}
```

### Symbolic execution
We can limit the search-space covered by a fuzzer with the help of symbolic execution. We can specify bounds on variable values that ensure the coverage of a desired path, using so-called **symbolic variables**.
We assign symbolic values to these variables rather than explicitly enumerating each possible value.

We can then construct the formula of a **path predicate** that answers this question:
_Given the path constraints, is there any input that satisfies the path predicate?_.
We then only fuzz the values that satisfy these constraints.
A popular tool for symbolic execution is _Z3_.
It is a combinatorial solver that, when given path constraints, can find all combinations of values that satisfy the constraints.
The output of _Z3_ can be given as an input to a generative or mutative fuzzer to optimally test various code paths of the SUT.

The path predicate for the `else if` branch in the previous code snippet will be: $$((N+M \leq 2) \& (N < 100))$$. The procedure to derive it is as follows:

1. `a` and  `b` are converted into symbolic variables, such that their values are: $$a=N$$ and $$b=M$$.

1. The assignment statements change the values of `a` and `b` into $$a = N+M$$ and $$b = (N+M) - M = N$$.

1. The path constraint for the `if` branch is $$(N+M > 2)$$, so the constraint for the other branches will be its inverse: $$(N+M \leq 2)$$.

1. The path constraint for the `else if` branch is $$(N < 100)$$.

1. Hence, the final path predicate for the `else if` branch is the combination of the two: $$(N+M \leq 2) \& (N < 100)$$


Note that it is not always possible to determine the potential values of a variable because of the *halting problem* — answering whether a loop terminates with certainty is an undecidable problem. So, for a code snippet given below, Symbolic execution may give an imprecise answer.

```java
public void func(int a, boolean b){
    a = 2;
    while (b == true) {
        // some logic
    }
    a = 100;
}
```

>> For interested readers, we recommend the "fuzzing book": [https://www.fuzzingbook.org](https://www.fuzzingbook.org)!

## References

* Fuzzing for bug hunting. https://www.welcometothejungle.com/en/articles/btc-fuzzing-bugs
* Fuzzing Java in OSS-Fuzz. https://security.googleblog.com/2021/03/fuzzing-java-in-oss-fuzz.html



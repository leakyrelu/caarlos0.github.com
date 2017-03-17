---
layout: post
title: "Getting code coverage of a monolithic app in production"
---

Microservices is the new fancy way of doing applications, but, most
companies still have big, old monoliths in production. In rapid-evolving
softwares this big, it's common to have several lines of code which
are never executed in production, but, sometimes, it's hard to find them.

With that in mind, I decided to try to get the code coverage of production
code. Yeah, in production.

But first, I'll introduce you to our subject.

## ContaAzul, the monolith

The monolith in question here is [ContaAzul][]. It started as a Struts2
application and evolved to a Java EE application, but still has Struts2
code running in it. You can check [these slides][slides] by [@Marcos][]
to get an idea on how it evolved.

According to `cloc`, it has ~5k Java files, summing ~400k lines of code.
That's a lot of code, and I'm not even counting other kinds of source files
(like SQL, templates, etc).
In an attempt of remove useless Java code, I had the idea of running
the [JaCoCo][] agent in a production server for some time, and try to
generate the reports.

[@Marcos]: https://github.com/marcos
[slides]: https://pt.slideshare.net/marcoswp3x/tdc-2015-java-from-old-school-to-moder-art

## Wait, JaCoCo what?

[JaCoCo][] stands for "Java Code Coverage". It is usually used to
measure code covered by unit tests.

It is an agent that you pass to the `java` command in the
`javaagent` flag. It will then instrument your _bytecode_ and generate a binary
file. Having the source files and this binary report file, you can
generate HTML reports which are human readable.

In the build phase, Maven plugins usually do all this work for us. But I don't
want the test coverage, I want production coverage. So, I need to add
the JaCoCo agent to a production server to instrument production code.

## Instrumenting production code

I confess that the very first time I did all this by hand, mostly because
I wasn't sure it would work. After checking that it does work, I decided
to automate it.

We use Puppet to manage our servers configuration. So I created a class
called `jacoco` which would download and instrument our Wildfly
application server if a `coverage` tag was set on the target node:

```puppet
class contaazul_app::jacoco {
  include stdlib

  $jacoco_version='0.7.9'

  if tagged('coverage') {
    exec { 'download-jacoco':
      command => "/usr/bin/curl -L -o /storage/environment/jacoco-${jacoco_version}.zip https://repo1.maven.org/maven2/org/jacoco/jacoco/${jacoco_version}/jacoco-${jacoco_version}.zip",
      creates => "/storage/environment/jacoco-${jacoco_version}.zip",
    }~>
    exec { 'unzip-jacoco':
      command     => "/usr/bin/unzip /storage/environment/jacoco-${jacoco_version}.zip -d /storage/environment/jacoco",
      refreshonly => true,
      creates     => '/storage/environment/jacoco',
    }
    file_line { 'jacoco_java_opts':
      path => '/storage/environment/contaazul/bin/standalone.conf',
      line => 'JAVA_OPTS="$JAVA_OPTS -javaagent:/storage/environment/jacoco/lib/jacocoagent.jar=destfile=/storage/environment/contaazul/jacoco.exec,output=file,append=true,dumponexit=true"',
    }
  }
}
```

With that in place, I could tag the nodes I want coverage reports for:

```puppet
node api {
  tag('coverage')
}
```

But, running JaCoCo is only part of the problem. We also need to gather
the binary files and generate the HTML reports from them.

## Compiling the reports

The first step is to get the report from the server. The most
straightforward way is to use `scp`, and that was what I did.

Then, I downloaded and extracted JaCoCo to my `/tmp` folder:

```console
$ wget https://repo1.maven.org/maven2/org/jacoco/jacoco/0.7.9/jacoco-0.7.9.zip -O /tmp/jacoco.zip
$ unzip /tmp/jacoco.zip -d /tmp/jacoco
```

With all that in place, I wrote a very simple Ant task to generate the
HTML reports:

```xml
<project name="ContaAzul" xmlns:jacoco="antlib:org.jacoco.ant">
    <taskdef uri="antlib:org.jacoco.ant" resource="org/jacoco/ant/antlib.xml">
        <classpath path="/tmp/jacoco/lib/jacocoant.jar"/>
    </taskdef>
    <jacoco:report>
        <executiondata>
            <file file="jacoco.exec"/>
        </executiondata>
        <structure name="ContaAzul">
            <classfiles>
                <dirset dir="." includes="**/target/classes"/>
            </classfiles>
            <sourcefiles encoding="UTF-8">
                <dirset dir="." includes="**/src/main/java"/>
            </sourcefiles>
        </structure>
        <html destdir="report"/>
    </jacoco:report>
</project>
```

Them fire `ant` up and open the report in your web browser:

```console
$ ant
$ open report/index.html
```

You should see a report like this (this is a fake one):

![Fake example report ordering by less coverage](/public/images/coverage-report.png)

ProTip™: if the Ant task fails, try to run it with `-v` for a more verbose
output.

## To automate or not to automate

At this point I wanted to automate it all the way up and have daily
reports or something like that.

After giving it some thought, I decided to run it a for a few hours in
a chosen server when I need or someone asks for the reports.
The reasoning behind this decision was:

- JaCoCo may add some overhead;
- People won't look at these reports every day;
- Running it all the time would add the complexity of:
  - Merging reports;
  - Dealing with code that is always changing;
  - Ant failures due to class name conflicts (which may happen on big
multi-module projects).

I'm also aware that running it this way have its own setbacks:

- I (or someone else) have to manually generate and share the reports;
- Some code may be used only in some days or periods of the day (seasonality),
which may induce to humans removing code that is used.

Anyways, with all this code in place it's kind of easy to run it for a while
in a server and generate the reports later.

## Results

The team was able to open ~20 pull requests which removed ~5k lines of
useless code in very little time. I'm personally happy with this result and
looking forward to generate more reports and remove even more code.

How about you? What are your strategies to safely delete code?

[ContaAzul]: http://contaazul.com
[JaCoCo]: https://github.com/jacoco/jacoco

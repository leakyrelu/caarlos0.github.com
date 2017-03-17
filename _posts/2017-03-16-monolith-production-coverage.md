---
layout: post
title: "Getting code coverage of a monolithic app in production"
---

> intro?

Lot's of companies have monolithic applications running in production.
It's likely that most of those applications have pieces of code that are never executed.

The problem is that those pieces of code are hard to find.
Monoliths usually have a lot of not-very-well structured code.

With that in mind, I decided to try to get the code coverage of production code, in production. This is the story of how I get it.

## ContaAzul, the monolith

The monolith in question here is [ContaAzul][]. It started as a Struts2
application and evolved to a Java EE application, but still has Struts2
code running in it.

According to `cloc`, it has 5k Java files, summing 400k lines of code.
It also have some SQL, Maven, XML, some template languages. Counting them all,
we get 7k files and 900k lines of code.

In an attempt of remove useless code, I had the idea of running the [JaCoCo][]
agent in a production server for some time, and try to generate the reports.

## JaCoCo?

[JaCoCo][] stands for "Java Code Coverage", and it is usually used to
measure code covered by unit tests.

It is an agent that you pass to the `java` command in the
`javaagent` flag. It will then instrument your _bytecode_ and generate a binary
file. Having the source files and this binary report file, you can
generate HTML reports which are human readable.

That seems a lot of work, so, how can I automate it?

## Instrumenting production code

We use Puppet to manage our servers configuration. I created a class
called `jacoco` which would download and install instrument our Wildfly
application server if a `coverage` tag is set on the server:

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

With that in place, I could tag the nodes I want reports for:

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

With that file in hands, I wrote a very simple Ant task to generate the
HTML reports:

```xml
<project name="ContaAzul" xmlns:jacoco="antlib:org.jacoco.ant">
    <taskdef uri="antlib:org.jacoco.ant" resource="org/jacoco/ant/antlib.xml">
        <classpath path="/Users/carlos/Downloads/jacoco-0.7.9/lib/jacocoant.jar"/>
    </taskdef>
	<jacoco:report>
		<executiondata>
			<file file="jacoco.exec"/>
		</executiondata>
		<structure name="contaazul-app-business">
			<classfiles>
        <fileset dir="contaazul-app-business/target/classes"/>
			</classfiles>
			<sourcefiles encoding="UTF-8">
        <fileset dir="contaazul-app-business/src"/>
			</sourcefiles>
		</structure>
		<html destdir="contaazul-app-business/target/report"/>
	</jacoco:report>
</project>
```

It is worth saying that our monolith, as most monoliths (I assume), has
a lot of modules. `contaazul-app-business` is one of them, and I'm
generating its report in the sample task above. Each module would have
its own report


## To automate or not to automate

It is worth to mention that we don't intend to run this all the time, but
rather for a few hours in one server. The reasoning behind this decision is:

- JaCoCo may add some overhead;
- We won't look at these reports every day;
- Running it all the time would add the complexity of:
  - Merging reports;
  - Dealing code that is always changing;
  - Automate all these steps.

I'm also aware that running it this way have its own setbacks:

- I (or someone else) have to manually generate the reports;
- Some code may be used only in some days or periods of the day (seasonality).



[JaCoCo]: https://github.com/jacoco/jacoco

- pq
- o que é o jacoco
- java agents
- colocando pra rodar no jboss
- coletando info
- compilando a info coletada

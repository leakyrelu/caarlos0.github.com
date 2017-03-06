---
layout: post
title: "From Travis Enterprise to BuildKite"
---

At [ContaAzul][], we use the CI infrasctructure **a lot**. We open
several pull requests in several projects every day, and we block the merge
if the build don't pass. Our `master` branches are sacred, and we can't
afford too much waiting.

## Travis Enterprise

For the past year, we were using [Travis Enterprise](https://enterprise.travis-ci.com/).
While it was pretty good, it also had some problems (in my opinion, at least):

- No worker autoscale: We wrote some Ruby + Terraform scripts and put them in
a crontab to do autoscale for us, but it never worked really well, given us
random Build failures on Travis;
- It was really expensive for us. Dollars are very expensive here in Brazil.

So, I went out looking for cheaper alternatives. You'll never believe what
I've found!

[ContaAzul]: http://contaazul.com

## BuildKite

My friend [Caio](https://github.com/caiofbpa) told me they were using
[Buildkite][] at [MeusPedidos][] and that it was working really well.

So, my plan was to setup Buildkite for some repositories, at least, let both
Buildkite and Travis run for a few days, interview the rest of the team
to ask them what they think, and, finally, if the results were good, migrate
all repositories to Buildkite and terminate some machines.

Setting up the master and the workers was really easy. Buildkite provides
a tool called [Elastic CI Stack for AWS](https://github.com/buildkite/elastic-ci-stack-for-aws).
It basically a CloudFormation template with a AutoScaling Group of workers.

It works really great, letting us even use Spot Instances (which most of
the time are cheaper).

I spent some time to read all the of the code before using it (trust no one),
launched the cluster on our sandbox AWS account, and, boom, Buildkite
is working!

Then, I set Buildkite up on the top 3 most changed repositories and let it
run for a few days.

[Buildkite]: https://buildkite.com/
[MeusPedidos]: https://meuspedidos.com.br/

## Decision making

After a week, I was sold to Buildkite. But, I'm not the only user, so I asked
the team to classify both Travis and Buildkite from 0 to 10 and if what would
they feeling be if we migrate to Buildkite, and here are the results:

Classify, from 0 to 10, your satisfaction with Travis:

![Travis score distribution](/public/images/travis-scores.png)

Classify, from 0 to 10, your satisfaction with Buildkite:

![Buildkite score distribution](/public/images/buildkite-scores.png)

How would you feel if we fully migrate to Buildkite?

![Feeling about migrating to Buildkite](/public/images/buildkite-migration-feelings.png)

All the scores were good:

- Buildkite had higher satisfaction;
- No one voted -1 about fully migrating to Buildkite.

The only downside I was able to see was the migration process.

## The migration process

I had 50 projects to migrate to Buildkite. The migration process was:

1. Clone the repository;
1. Remove the `.travis.yml` file;
1. Create a new pipe on Buildkite;
1. Setup GitHub integration;
1. Create `.buildkite/pipeline.yml` and other required files;
1. Open a Pull Request;
1. Check if it worked.

7 steps x 50 repositories = A lot of work.

It would be really easier if:

- Buildkite automatically integrated with GitHub;
- Buildkite accepted `.travis.yml` as a fallback or had a conversion tool of
some sort.

But none of those were true, so I automated some steps of the process.

## The real work

First of all, I cloned all repositories with the help of [clone-org]:

```console
$ clone-org --org ContaAzul --destination /tmp/ca
```

Second, I created both Buildkite and GitHub API tokens to script the link
between them. [This script is now open-source][gh-wire].

After that, I some template files for Java Buildkite builds, since most of
our projects are in Java, I figured that this will do most of the work for me.

With all that in place, I wrote a "one-liner" and a helper function that
would do most of the work for me:

```bash
# helper function
setup_buildkite() {
  cd ~/Code/github-buildkite-wire &&
    ./setup-pipeline.sh ContaAzul "$(basename "$1")"
}
```

```bash
# the one liner
$ find . -depth 1 -type d | while read -r folder; do
  cd "$folder"
  test -f '.travis.yml' &&
    setup_buildkite "$folder" &&
    rm .travis.yml &&
    git checkout -b rm-travis &&
    git commit -am 'removed travis.yml' &&
    cp -r ~/Code/buildkite/template/* . &&
    git add -A &&
    git commit -m 'buildkite :wrench:' &&
    git pr
done
```

PS: The `git pr` alias refers to [this plugin](https://github.com/caarlos0/zsh-open-pr)

Since `git pr` open a tab in the default browser, my Chrome hang up for a little,
and then I just hit the green button several times to create the pulls.

Some of them failed, some worked, some needed more stuff. But, most of my
work was successfully automated and I was able of doing days of work in
~2 hours.

[clone-org]: http://github.com/caarlos0/clone-org
[gh-wire]: https://github.com/caarlos0/github-buildkite-wire

## Final thoughts

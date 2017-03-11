---
layout: post
title: "From Travis Enterprise to BuildKite"
---

At [ContaAzul][], we use the CI infrastructure **a lot**. We open
several pull requests in several projects every day, and we block the merge
until the build pass. We consider our `master` branches are sacred, and
we can't afford too much waiting to change them.

## Travis Enterprise

For the past year, we have been using [Travis Enterprise](https://enterprise.travis-ci.com/).
While it is pretty good, it also has some problems (in my opinion, at least):

- No worker autoscale: We kind of solve this by writing some
Ruby + Terraform scripts and putting them in a crontab to
do autoscale for us, but it never really worked well, giving us random
build failures on Travis;
- It was really expensive for us. Dollars are very expensive here in Brazil,
and Travis Enterprise costs a lot of them.

So, I went out looking for cheaper and hopefully better alternatives.

[ContaAzul]: http://contaazul.com

## BuildKite

My friend [Caio](https://github.com/caiofbpa) told me they were using
[Buildkite][] at [MeusPedidos][] and that it was working really well.

I decided to give it a try. My plan was:

1. Setup Buildkite for some repositories;
2. Let both Buildkite and Travis run for a few days;
3. Interview the team to ask them what they think;
4. If the results were good, migrate all repositories to Buildkite and
terminate some machines.

Setting up the master and the workers was really easy. Buildkite provides
an Open Source tool called
[Elastic CI Stack for AWS](https://github.com/buildkite/elastic-ci-stack-for-aws).
It is basically a CloudFormation template with an AutoScaling Group of workers.

It works really great, letting us even use Spot Instances (which most of
the time are cheaper).

I spent some time reading all its code before using it (trust no one),
launched the cluster on our sandbox AWS account, and, boom, Buildkite
is working!

Then, I set Buildkite up on the top 3 most changed repositories and let it
run for a few days.

[Buildkite]: https://buildkite.com/
[MeusPedidos]: https://meuspedidos.com.br/

## Decision making

After a week, I was sold to Buildkite. But, I'm not the only user, so I asked
the team to classify both Travis and Buildkite from 0 to 10 and what would
they feel about migrating to Buildkite. Here are the results:

Classify, from 0 to 10, your satisfaction with Travis:

![Travis score distribution](/public/images/travis-scores.png)

Classify, from 0 to 10, your satisfaction with Buildkite:

![Buildkite score distribution](/public/images/buildkite-scores.png)

How would you feel if we fully migrate to Buildkite?

![Feeling about migrating to Buildkite](/public/images/buildkite-migration-feelings.png)

All the scores were good:

- Buildkite had higher satisfaction;
- No one voted against migrating to Buildkite.

The only downside I was able to see was the migration process.

## The migration process

I had 50 projects to migrate to Buildkite. The migration process was:

1. Clone the repository;
2. Remove the `.travis.yml` file;
3. Create a new pipe on Buildkite;
4. Setup GitHub integration;
5. Create `.buildkite/pipeline.yml` and other required files;
6. Open a Pull Request;
7. Check if it worked.

7 steps x 50 repositories = A lot of work.

It would be easier if:

- Buildkite had better GitHub integration;
- Buildkite accepted `.travis.yml` as a fallback or had a conversion tool of
some sort.

But none of those were true, so I automated some steps of the process.

## Automating the work

First of all, I cloned all repositories with the help of [clone-org][]:

```console
$ clone-org --org ContaAzul --destination /tmp/ca
```

Second, I created both Buildkite and GitHub API tokens to script the link
between them. Of course I made the script [open-source][gh-wire].

After that, I created some template files for Java Buildkite builds, since
most of our projects are in Java, I figured that this will do most of the
work for me. This included creating a Docker image with Java 8, Java 7,
Maven and our Nexus repository configuration.

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

PS: The `git pr` alias calls [this plugin](https://github.com/caarlos0/zsh-open-pr)

Since `git pr` open a tab in the default browser, my Chrome hang up for a
little, and then I just hit the green button several times to create the pulls.

Some of them failed, some worked, some needed more stuff. But, most of my
work was successfully automated and I was able of doing days of work in
about 2 hours.

[clone-org]: http://github.com/caarlos0/clone-org
[gh-wire]: https://github.com/caarlos0/github-buildkite-wire

## Final thoughts

We are really happy with the results! Our previous CI stack was not working
very well with an elastic amount of workers, which caused several builds
to ramdonly fail during the day.

Buildkite native elastic stack works great, the web interface feels faster
and cleaner and I never had tickets about random job failures!


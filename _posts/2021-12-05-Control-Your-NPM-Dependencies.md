---
title: Control your npm dependencies
date: 2021-12-05 09:00:00 +0530
categories: [JavaScript]
tags: []     # TAG names should always be lowercase
math: true
mermaid: true
toc: true
---

## Background

Initializing a new react application using create-react-app installs 1900 packages, while the package.json only
defines 23 dev dependencies. A fresh angular app uses the same number of direct dependencies and installs 1029
packages. This is the minimum number of packages where many developers start their projects. The only way forward
is more dependencies.

As per GitHub’s State of the Octoverse 2020 [security report](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwjI5Mvg5Lr0AhWfH7cAHTZ7A1YQFnoECA8QAQ&url=https%3A%2F%2Foctoverse.github.com%2Fstatic%2Fgithub-octoverse-2020-security-report.pdf&usg=AOvVaw1UB7Ipca--qRS8EkGI6X8K), the mean JavaScript repository has 10 direct dependencies
and 683 transitive dependencies (dependencies of dependencies). PHP comes second with 9 direct dependencies and 70
transitive dependencies. The npm ecosystem can be accurately defined by this popular xkcd,

![A project some random person in Nebraska has been thanklessly maintaining since 2003](https://miro.medium.com/max/770/0*CSAEGeuhVWNJBw-g.png)
_Source: [https://xkcd.com/2347/](https://xkcd.com/2347/)_

The harsh truth is that maintaining open source packages is unrewarding hard work. No one can be held accountable
to fix vulnerabilities or critical issues in OS packages. In 2016, the left-pad npm package was deleted from npm by
its maintainer, causing thousands of projects to break worldwide. Such incidents and security vulnerabilities become
frequent with an increasing number of dependencies. This is not a problem that can be deferred to the future.

## Why are there so many transitive dependencies?

Let’s look at some npm packages before going through the reasons,

* [isarray](https://www.npmjs.com/package/isarray) has 53 million weekly downloads and 812 dependent packages. This is
    the only function exported by the library,

    ```js
    module.exports = Array.isArray || function (arr) {
        return {}.toString.call(arr) === '[object Array]';
    };
    ```

* [is-number](https://www.npmjs.com/package/isnumber) has 55 million weekly downloads and 833 dependent packages.
    It exports a [7 line function](https://github.com/jonschlinkert/is-number/blob/master/index.js).

* [is-windows](https://www.npmjs.com/package/is-windows) with 18 million weekly downloads and 615 dependent packages
  exports only the following function,

  ```js
  return process && (process.platform === 'win32' || /^(msys|cygwin)$/.test(process.env.OSTYPE));
  ```
* [is-promise](https://www.npmjs.com/package/is-promise) has 10 million weekly downloads and 818 dependent packages.
  Again, this exports a [single line function](https://github.com/then/is-promise/blob/master/index.js).

* [is-even](https://www.npmjs.com/package/is-even) has 160k weekly downloads and itself depends on
  [is-odd](https://www.npmjs.com/package/is-odd), which has 430k weekly downloads. Both of these packages are single
  line functions. At one point, babel was using the is-odd package.

*The numbers are taken as of 4th Dec, 2021.*

These examples are only a demonstration of the extent of the problem. There are probably some useful single-line
function packages. Similarly, there could be packages with a good number of lines that do not necessarily do anything
justifiably useful.

That is how we ended up here. **A ton of tiny packages.**

This is not a problem found in other languages. PHP is miles away with only 70 mean transitive dependencies. One of the
main reasons for this is the **lack of a good standard library** in JavaScript. Most of these little packages are
generally substitutes for basic utility functions. Java has extensive in-built libraries that give performant and
correct utility functions. So do C++ and PHP. JavaScript did not have a good standard library, which ended up driving
developers to depend on external libraries for trivial tasks. In the language’s defense, utility functions have been
continuously improved with every ECMAScript version.

## How does this affect us?

We live in a privileged time where a 2GB node_modules directory is acceptable. Memory is no longer a constraint,
but as shown in the xkcd it only takes one stone to bring your project to its feet.

### Packages can disappear

While we cherish Open Source for its benefits, we should also be wary of its downsides. One fine day in 2016, the
maintainer of a widely used package, left-pad, unpublished all of their packages from npm. They had valid reasons to
do so, but this left the world scrambling as this tiny 7-line package was a dependency for some major packages like
React. Although npm has long since disabled unpublishing popular packages with no prior notice, there are no
guarantees for obscure packages.

### Abandoned packages

Most package owners create their packages out of necessity. After a point, they no longer see any benefit in
maintaining their packages. They usually have to deal with a day job even if they decide to actively maintain it.
Such packages either thrive on external contributors or die out slowly, leading to possibly unhandled vulnerabilities.
A package can end up abandoned within a year, given the velocity of the JavaScript ecosystem.

### Inadvertent errors

Most package users do not pin their dependency versions, which enables automatic upgrade of minor versions whenever
available. While in most cases this works fine, there have been cases where this caused issues. In Apr 2020, a minor
release was deployed for the package [is-promise](https://www.npmjs.com/package/is-promise), which had
[unintended](https://medium.com/@forbeslindesay/is-promise-post-mortem-cab807f18dcc) bugs causing most of its dependents
to break. Even though this was fixed within 3 hours, it should serve as a cautionary tale.

### Malicious actors

Package maintainers are not always well-intentioned. The owner of the popular package, ua-parser-js, transferred their
package ownership to another user who used their privileges to publish malware using a postinstall script. npm took
down these malicious packages within a few hours, but the damage was done.

Reducing the number of dependencies used helps reduce the surface area for such issues to arise.

## How can we solve this?

### Stop using dependencies for simple tasks

Write your own methods for minor tasks such as checking the type of a variable or manipulating a string, instead
of depending on a package. Prefer implementing the feature if it takes less than 10 minutes to do so. If this is not
an option or if the resulting code is hard to maintain, **prefer using packages that bundle common utilities**, such as
[lodash](https://lodash.com/) or [underscore.js](https://underscorejs.org/). You should use a trigonometry dependency
instead of one for cosine.

### Prefer packages with less number of transitive dependencies

Most of the above-mentioned problems cannot be fixed if they are transitive dependencies. A broken transitive dependency
would mean that the direct dependency needs to be updated first before the direct dependency can be updated. If possible,
choose a package with no dependencies. Otherwise, choose the ones with the least number of dependencies. Prefer packages
with a higher number of dependents as this can guarantee any vulnerabilities to be fixed soon.

If the dependencies are justified and the risks are considered, feel free to use any number of them!

## Conclusion

This is a deeply rooted problem that is not easily solvable even though I’ve listed “solutions”. Most dependencies in
node_modules come as transitive dependencies, which cannot be avoided or controlled. The best we can do is raise
awareness of the dangers of including packages recklessly.

*Please remember that all of these are opinions and ideas of a single person. So, feel free to correct me!*

Follow me on [Twitter](https://twitter.com/saihemanth9019) :)

## Sources

* [https://www.reddit.com/r/programming/](https://www.reddit.com/r/programming/)
* [https://blog.npmjs.org/post/141577284765/kik-left-pad-and-npm](https://blog.npmjs.org/post/141577284765/kik-left-pad-and-npm)
* [https://blog.npmjs.org/post/141905368000/changes-to-npms-unpublish-policy](https://blog.npmjs.org/post/141905368000/changes-to-npms-unpublish-policy)
* [https://github.com/then/is-promise/issues/13](https://github.com/then/is-promise/issues/13)
* [https://github.com/faisalman/ua-parser-js/issues/536](https://github.com/faisalman/ua-parser-js/issues/536)
* [https://javascript.plainenglish.io/is-promise-post-mortem-cab807f18dcc](https://javascript.plainenglish.io/is-promise-post-mortem-cab807f18dcc)
* [https://www.npmjs.com/](https://www.npmjs.com/)

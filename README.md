This repository contains the docs for Exist's API, which live at [developer.exist.io](http://developer.exist.io). 
They're built on Slate, so I've left the Slate readme contents here so you have instructions for building your own copy.


Slate
========

[![Build Status](https://travis-ci.org/tripit/slate.svg?branch=master)](https://travis-ci.org/tripit/slate) [![Dependency Status](https://gemnasium.com/tripit/slate.png)](https://gemnasium.com/tripit/slate)

Slate helps you create beautiful API documentation. Think of it as an intelligent, responsive documentation template for your API.

Getting Started with Slate
------------------------------

### Prerequisites

You're going to need:

 - **Linux or OS X** — Windows may work, but is unsupported.
 - **Ruby, version 1.9.3 or newer**
 - **Bundler** — If Ruby is already installed, but the `bundle` command doesn't work, just run `gem install bundler` in a terminal.

### Getting Set Up

 1. Fork this repository on Github.
 2. Clone *your forked repository* (not our original one) to your hard drive with `git clone https://github.com/YOURUSERNAME/slate.git`
 3. `cd slate`
 4. Install all dependencies: `bundle install`
 5. Start the test server: `bundle exec middleman server`

Or use the included Dockerfile! (must install Docker first)

```shell
docker build -t slate .
docker run -d -p 4567:4567 slate
```

You can now see the docs at <http://localhost:4567>. Whoa! That was fast!

*Note: if you're using the Docker setup on OSX, the docs will be
availalable at the output of `boot2docker ip` instead of `localhost:4567`.*

Now that Slate is all set up your machine, you'll probably want to learn more about [editing Slate markdown](https://github.com/tripit/slate/wiki/Markdown-Syntax), or [how to publish your docs](https://github.com/tripit/slate/wiki/Deploying-Slate).


Need Help? Found a bug?
--------------------

Just [submit a issue](https://github.com/tripit/slate/issues) to the Slate Github if you need any help. And, of course, feel free to submit pull requests with bug fixes or changes.


Contributors
--------------------

Slate was built by [Robert Lord](http://lord.io) while at [TripIt](http://tripit.com).

Thanks to the following people who have submitted major pull requests:

- [@chrissrogers](https://github.com/chrissrogers)
- [@bootstraponline](https://github.com/bootstraponline)
- [@realityking](https://github.com/realityking)

Also, thanks to [Sauce Labs](http://saucelabs.com) for helping sponsor the project.

Special Thanks
--------------------
- [Middleman](https://github.com/middleman/middleman)
- [jquery.tocify.js](https://github.com/gfranko/jquery.tocify.js)
- [middleman-syntax](https://github.com/middleman/middleman-syntax)
- [middleman-gh-pages](https://github.com/neo/middleman-gh-pages)
- [Font Awesome](http://fortawesome.github.io/Font-Awesome/)

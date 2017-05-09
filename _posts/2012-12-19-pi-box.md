title:        Dropbox on Pi
categories:   arm pi tools
date:         2012-12-19 00:54

There's currently no build of the Dropbox sync client that runs on ARM devices; while there is source available, it's not suitable for automatic, unattended sync on a Raspberry Pi that may not have any windowing system installed.

Dropbox does make various SDKs available though, which enables a ruby or python script to be used instead of the official client.

<!-- more -->

# a ready made solution..

[dbox] is a great ruby client for Dropbox that can locally create, clone, push, pull, sync, and move, a Dropbox folder hierarchy.  It can be used as an API directly from another ruby script, or with the included command line tool.

[^dev]: A developer who's quite involved with the ruby incarnation of the Dropbox SDK has a nice tool that does almost exactly what I want.  There's even an example script to do *exactly* what I want! ;)

> [dbox]  
> Dropbox integration made easy. This robust client gives you control over what, where, and when you sync with Dropbox.

[dbox]: https://github.com/kenpratt/dbox
[example script]: https://github.com/kenpratt/dbox/blob/master/sample_polling_script.rb

dbox syncs one file at a time, so it won't be as quick as the official client when syncing large amounts of files.  For most uses this will only be an issue when initially cloning a large folder.

## installation

Before installing dbox, make sure you have any needed packages.

```sh
	sudo apt-get install ruby ruby-dev libsqlite3-dev
	sudo gem install dbox
```

Then follow the rest of the instructions from the [dbox] page, to configure your keys and clone your Dropbox folder.

To run dbox periodically, download[^dl] the [example script] and make it executable, then replace the example keys and path settings with your own.

[^dl]: the script is also included in the installed gem:  `cp /var/lib/gems/1.9.1/gems/dbox-0.7.2/sample_polling_script.rb dboxsync.rb`

<!--	cp /var/lib/gems/1.9.1/gems/dbox-0.7.2/sample_polling_script.rb dboxsync.rb -->

```sh
	wget https://raw.github.com/kenpratt/dbox/master/sample_polling_script.rb
	mv ./sample_polling_script.rb dboxsync.rb
	chmod +x dboxsync.rb
	nano dboxsync.rb
	./dboxsync.rb
```

All output is written to a log file, so it's normal not to see any output while running.  If you want to see some basic info while testing, change the script so `LOGFILE = STDOUT`.  To show the full syncing progress and other debug info, set the level of the [logger class] by changing the script so it reads `LOGGER.level = Logger::DEBUG`.

[logger class]: http://www.ruby-doc.org/stdlib-1.9.3/libdoc/logger/rdoc/Logger.html


To have the script run on startup, add the following entry to `/etc/rc.local`.

```sh
	sudo nano /etc/rc.local

	# add the following
	if ! pgrep -f dboxsync; then
	  sudo -u xbian nice -n 10 /home/xbian/dboxsync.rb &
	fi
```

There's more proper ways of doing this, but this seemed the simplest.


## todo

* increase polling delay, but add detection of local changes to initiate push/pull


## notes

While cron or similar could be used to run dbox periodically, each separate invocation of dbox from the command line incurs a startup delay of a few seconds, during which time the Pi's CPU is maxed out loading ruby and the code needed to run dbox.  This isn't ideal and would likely cause issues if the Pi is used as a media player.

The sample_polling_script accesses dbox directly though it's ruby API, and doesn't exit during each polling interval.  Therefore, code remains cached in memory[^mem] and the delay occurs only once upon startup; this is a great advantage over running the command line version periodically.

[^mem]: memory usage starts at ~8MB after startup, and evens out to ~12MB after many polling loops; polling takes a second or two, during which CPU usage is around 5%, peaking at ~12%.  Contrast with the CLI version which uses a similar amount of memory, but takes several seconds to complete each sync, and includes the overhead of almost total CPU usage for half that time.


# alternatives

Here's some more options I considered, in order of decreasing usefulness.


## one way sync

Create a simple ruby script to sync files one way - _from Dropbox_ - using the Dropbox SDK.  It's also possible to sync both ways in ruby, but [dbox] has already done a great job implementing that.

Ruby SDK [Tutorial](https://www.dropbox.com/developers/start/setup#ruby), [Documentation](https://www.dropbox.com/static/developers/dropbox-ruby-sdk-1.5.1-docs/index.html).

eg. See the [delta method](https://www.dropbox.com/static/developers/dropbox-ruby-sdk-1.5.1-docs/DropboxClient.html#method-i-delta) to keep local files in sync without writing too much code.

There's also the cli_example.rb script included in the SDK, which provides a simple shell interface similar to FTP but does not have any syncing logic included.

[^ex2]: https://github.com/ACMatUCF/FlashSync


## linux daemon

Dropbox does have a download for a daemon, but it's x86/_64 only; see the [Linux Download](https://www.dropbox.com/install?os=lnx) page for more info.

> The Dropbox daemon works fine on all 32-bit and 64-bit Linux servers. To install, run the following command in your Linux terminal.


## build from source

The Dropbox source appears to be just a plugin for the nautilus file browser (and doesn't include the above daemon?).  This is only useful when using the desktop.

Download the [source archive](https://www.dropbox.com/download?dl=packages/nautilus-dropbox-1.4.0.tar.bz2) and build as usual.  More details can be found in [this help topic](https://www.dropbox.com/help/247).


## use something else

I also use github, sftp, scp, and rsync for file transfer, version history and keeping various things in sync; there are many ways to achieve automatic folder syncing.  However, Dropbox fulfills a role that can't easily be duplicated if you use many Dropbox enabled mobile apps, and do not wish the Pi to rely upon a second computer to sync through.
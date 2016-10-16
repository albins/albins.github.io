---
layout: post
title: Scheduling Periodic Cleaning in macOS
comments: true
---
On most of my machines, I treat the default Downloads folder as a staging area: it's where things arrive before being filed away wherever they belong. Most files, however, are only useful for a very brief period of time and belong nowhere really. Therefore, the Downloads folders on my systems tend to accumulate all sorts of meaningless gunk, as sorting and _taking out the digital trash_ is precisely the sort of menial task we built computers to _avoid having to do_. Or, they did before I automated the process of taking out the trash. Here's how.


Begin by setting up an Automator workflow to do the actual cleaning. You might be tempted to choose a workflow along the lines of "if file creation date is not within...", but that will -- for some reason -- exclude files created today. The proper way of doing it is to set up a NOT-AND criterion: if a file is not created today and not created within the last 60 days, move it to Trash as seen below.

![Screenshot of an Automator workflow for moving files in Downloads to Trash](/resources/find-and-delete-finder-items.png)

The next step is to periodically schedule the Automator workflow using Launchd. If your computer is always on at a given time, you could use Cron, but Launchd has more advanced trigger options, and will also make sure to reschedule your task, should your computer have been off or sleeping at the time when it should have run. In this instance, this is not very important, as the script runs every hour, but if you would -- say -- clean your Downloads folder every first monday of the month, it suddenly becomes more important. In the script below, the workflow is called when it is first loaded (e.g. on login etc) as well as periodically every hour (on the 0:th minute), which may or may not be excessive for your use case (it probably is for mine).

Change the path to your Automator workflow file below (mine is in `~/Documents` and is called `clean-downloads.workflow`). It may be a good idea to avoid spaces in the file name. Save the Launchd configuration file in `~/Library/LaunchAgents/com.orgname.scriptname.plist` (as you can see below, i used `org.albin` and `cleandownloads` for organisation name and script/agent name respectively).

Once you are done, you may or may not need to load the script using `launchctl load <path-to-script>`, e.g. `launchctl load ~/Library/LaunchAgents/org.albin.cleandownloads.plist`.

If you want to do more in-depth editing of the Launchd script, I'd recommend using [LaunchControl](http://www.soma-zone.com/LaunchControl/). See also [this StackOverflow thread on creating Launchd tasks and where to place them](http://stackoverflow.com/questions/132955/how-do-i-set-a-task-to-run-every-so-often). It also contains other ways of scheduling periodic tasks under macOS.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>org.albin.cleandownloads</string>
	<key>ProgramArguments</key>
	<array>
		<string>automator</string>
		<string>/Users/albin/Documents/clean-downloads.workflow</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
	<key>StartCalendarInterval</key>
	<dict>
		<key>Minute</key>
		<integer>0</integer>
	</dict>
</dict>
</plist>
```

Please note that this script (for security reasons) doesn't actually _delete_ anything, it just moves them to the Trash. But it pairs excellently with macOS Sierra's feature for automatically purging Trash of old files!

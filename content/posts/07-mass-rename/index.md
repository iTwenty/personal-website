---
title: "Adventures in mass renaming files"
date: 2023-05-25T22:09:58+05:30
draft: false
tags: [cli]
authors: [itwenty]
---

Back in my college days, I was a bit of a data hoarder. I had built up a sizable collection of movies. Not necessarily for watching, but because someone somewhere on the Internet said something good about the movie. Recently, I decided to put this movie collection to some good use. Or at least better use than sitting idly on a hard drive gathering dust in a closet, waiting for entropy to toggle random bits and corrupt the files. Plan was to move the whole collection to the cloud and stream to my devices using [Jellyfin](https://jellyfin.org/).

Those familiar with Jellyfin - or other media servers like Plex, Emby etc - will know that the files need to be organized in a certain way for Jellyfin to be able to fetch correct metadata. My movie files were already organized, but not in the way Jellyfin wanted.

My movies were organized this way -

```
Highway
└── Highway (720p) (2014).mkv
Kingdom of Heaven
├── Kingdom of Heaven (1080p) (2005) (Director's Cut).mp4
└── Kingdom of Heaven (1080p) (2005) (Director's Cut).srt
```

The first three fields - movie title, resolution and release year - are present for all movies. Extras like (Director's Cut) are optional fields that may or may not be present. [Jellyfin docs](https://jellyfin.org/docs/general/server/media/movies/) recommend movie title be immediately followed by release year. In my case, I guess the resolution field between title and year was causing Jellyfin to treat resolution as part of movie title, which was obviously not correct.

The goal was simple - rename all files from current naming scheme to this one -

```
Highway
└── Highway (2014) (720p).mkv
Kingdom of Heaven
├── Kingdom of Heaven (2005) (1080p) (Director's Cut).mp4
└── Kingdom of Heaven (2005) (1080p) (Director's Cut).srt
```

 How to achieve this goal for over 400 files? Not so simple. I instinctively knew this task will need dabbling with regexes and capture groups. My initial idea was to whip up a quick python script. But this seemed like the sort of task that someone would have solved before me. So I set out to find a good readymade tool for mass renaming files with regex support.

### Attempt #1 - Finder
Not many people know it, but macOS's Finder has a pretty nifty mass rename tool. Just select a bunch of files and try renaming them to be greeted with this dialog -

{{< image src="finder_rename.png" alt="Finder mass rename" position="center" style="border-radius: 8px;" >}}

Finder's mass rename works pretty well for simple stuff like adding, removing or replacing fixed characters. But it has no support for regex, so re-arranging parts of filename is not possible. On to the next alternative...

### Attempt #2 - Various GUI tools
A quick google search along the lines of "macOS bulk file rename" yields a ton of GUI tools built for this purpose. Of all the options, these two looked promising -

- [NameChanger](https://mrrsoftware.com/namechanger/)
- [A Better Finder Rename (ABFR)](https://www.publicspace.net/ABetterFinderRename/index.html)

NameChanger is free to use but hasn't been updated in a while. I decided to give it a go anyway. Unfortunately, it kept crashing randomly whenever I tried to edit the regex.

ABFR is very promising. It has a ton of options for renaming, shows real-time previews of renames and is updated regularly. It costs 30$ but offers a free trial version that can rename 10 files. It worked very well in renaming the 10 files I gave it, but I found it a tad too costly for what is basically a one-time task for me. I made a mental note to get back to ABFR in case I don't find anything better and moved on to the next alternative...

### Attempt #3 - Embrace the command line
It's strange how searching for "macOS bulk file rename" shows only GUI tools when there's a plethora of CLI-based tools available as well. One such CLI program that looked promising was [rnr](https://github.com/ismaelgv/rnr). It ticked all the boxes I needed -

 - Has support for regex and capture groups
 - Dry run mode so I can verify the renames before the fact.
 - Ability to undo the renames in case I mess up even after dry run!

Now that I had seemingly found the perfect tool for the job, next task was conjuring a regex to match the original filenames. [regex101](https://regex101.com/) is super helpful here, as is rnr's dry run mode. Here's the regex I came up with -


```
 (.*) (\(720p\)|\(1080p\)) (\([0-9]+\))
```

And here's the full command I used to rename -

```
rnr -rfD  '(.*) (\(HD\)|\(1080p\)) (\([0-9]+\))' '${1} ${3} ${2}' ./*

-r Recursive mode since actual movie files are inside sub-directories.
-f Perform the actual rename. rnr is dry run by default. So you need to pass this flag to actually make the changes.
-D Match and rename directories as well, along with files.
```

The regex captures three groups from each matched filename - title, resolution and year. The replacement string `${1} ${3} ${2}` simply swaps the position of resolution and year in the matched name. The final `./*` tells rnr to include all directories recursively from the current directory.

### Conclusion
After nearly 2 hours of research, trial and error, it was good to see rnr proceed to rename 400+ files in less than a minute with no issues. Writing my own script for this would have taken longer. Adding support for features like dry run and recursive mode - far far longer. So I guess the moral of the story is - Look for existing tools for the task at hand rather than trying to reinvent the wheel!

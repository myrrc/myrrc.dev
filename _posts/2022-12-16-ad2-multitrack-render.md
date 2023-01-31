---
layout: post
title: "Rendering Addictive Drums 2 track to multitracks: a Reaper script"
tags: music vst reaper
---

Sometimes I use XLN Audio Addictive Drums 2 (AD2) as a drum synthesizer for my projects. However, by default, this
plugin outputs just a stereo mix. For mixing purposes, one usually needs drum multitracks, 6 to 20 wav files.
I got tired of excessive mouse clicking and decided to automate the task with a
[Reaper script](https://github.com/myrrc/reaper_scripts/blob/master/Rendering/ad2mixrender.lua).

> This article is way too big for a 100-line plugin description. Think of it as an introduction to Reaper scripting.

## Initial task

If you send your track to a mixing/mastering engineer, they would usually expect the following drum parts:

- Kick (mono, a mix of in/out microphones or two separate channels),
- Snare (mono, a mix of top/bottom microphones or two separate channels),
- Hihat (mono),
- Tom-toms (either a single stereo channel or 3-4 mono tracks for hi, med, lo, and floor toms),
- Overhead (stereo channel capturing cymbals and drum kit high-frequency reverb),
- and Room (stereo channel capturing low-frequency reverb).

You may add some submix buses (like AD2 does), mike hihat with two or more microphones, add external
sounds like cowbells and percussion, but you need to include these 6 channels at least.
I wanted to transform my MIDI with AD2 into such channels.

AD2 is a VSTi (basically, a plugin that produces sounds) exposing 18 mono output channels:
stereo master, kick, snare, hihat, 4 toms, 3 additional sound slots (flexi), stereo overhead, stereo room, and
stereo submix bus.
Inside of the plugin, there is a 13-channel mixer. Each channel can be routed to the internal master (which in turn routes
to channels 1/2 of plugin output) and/or to external outputs.

<a id="build-routing"></a>
Solution [proposed](https://xlnaudio-assets.s3.amazonaws.com/documents/separate-outputs.pdf)
by XLN Audio is to click through every channel in the internal mixer, routing it to plugin outputs, and then operate on
the latter.
For example, Reaper has a [handy tool](https://forum.cockos.com/showpost.php?p=247858&postcount=3)
that automatically builds channels for every output channel in VSTi.
The tool can be invoked in two ways:

1. `Insert virtual instrument`, or, if you already have the plugin inserted,
2. `FX > Options > Build multichannel routing for output of selected FX`.

However, that still involves a lot of unnecessary mouse-clicking work, so I decided to automate it.

## API

There are multiple ways of automating work in Reaper. You can write scripts
([ReaScripts](http://www.cockos.com/reaper/sdk/reascript/reascript.php)) in C-like
[EEL2](https://www.cockos.com/EEL2/) language, use
built-in Lua API, or Python (official `RPR` or more user-friendly
[`reapy`](https://python-reapy.readthedocs.io)).
Python is not bundled with Reaper and I don't have it on my PC, EEL2 looks quite scary and lacks important functions
like file manipulation, so I decided to stick to Lua.

API documentation can be obtained from
[the official website](https://www.reaper.fm/sdk/reascript/reascripthelp.html#l), but it's fairly outdated
and does not include widely-used extensions like [SWS](https://www.sws-extension.org/), so you can generate your own
docs from `Help > Reascript Documentation` (a page will open in your web browser).

And then comes the fun part. To be honest, ReaScript API is something you wouldn't get from any other DAW, but it's
still poorly documented. First of all, there are no code examples. You may obtain some from the
[Cockos wiki](https://wiki.cockos.com/wiki/index.php/REAPER_API_Functions), however, most pages are just generated
from the C header file and [don't help at all](https://wiki.cockos.com/wiki/index.php/RPR_InsertMedia).
If you know C-like languages, you may guess what a function does from its signature, nevertheless,
some names are hard to guess.

How to get a selected track? `GetSelectedTrack`. Ok, how to get FX on it? `TrackFX_GetInstrument` if your plugin is a
synth, in any other case you need another function. How to get FX's name? `TrackFX_GetFXName`. How to get FX's path
(as it appears under the "Path" header in Reaper `ProjectBay > FX`)? `BR_TrackFX_GetFXModuleName`.

Such small inconsistencies are fine, but if we add

- a lot of magic constants (current project = 0, rendering entire project in time bounds = 1,
  rendering selected tracks in projects = 3),
- bitwise magic (by default sends are stereo, to create a mono send, you need `send_channel | 1024`),
- index vs pointer arguments (some functions operate on `MediaTrack` index, some take a `MediaTrack` object), and
- legacy API designs (two different functions to set a numeric or a string value with a single get\set method
  with `bool is_set` as a parameter that determines whether it's a get or a set),

writing a script may turn out a complex task.

> Disclaimer: Reaper API is extremely cool. Still, you have to know there are some caveats.

## Solution

1. Determine the selected track with AD2 on it, do a sanity check (plugin name)
2. Invoke `Build multichannel routing ...`
3. Select newly created tracks
4. Render them to some user-specified folder
5. Delete selected tracks.

Quite a simple task, right? Wrong guess. But let's start from the beginning.
The simplest possible Lua script follows:

{% highlight lua %}
function main() reaper.ShowConsoleMsg("Hello\n") end

reaper.PreventUIRefresh(1)
reaper.Undo_BeginBlock()
main()
reaper.Undo_EndBlock("Our script", -1)
reaper.UpdateArrange()
reaper.PreventUIRefresh(-1)
{% endhighlight %}

Prevent/Update functions are just boilerplate to prevent UI nasal daemons; undo blocks come in handy when you want to
revert the operation at once (our operation is non-revertable, we'll [talk about it later](#undo-and-redo));
main function does the job.

In fact, using `ShowConsoleMsg` is tedious, so I wrote a slightly more complex version that acts like `printf`:

{% highlight lua %}
function msg(...) reaper.ShowConsoleMsg(string.format("%s\n", string.format(...))) end
{% endhighlight%}

### Selection and sanity checks

For simplicity, we may require a user to select the [only] track with AD2. This has two benefits: we get track id
immediately and can use this information for rendering (there is a [special mode](#render-selected)).

{% highlight lua %}
if reaper.CountSelectedTracks(CUR_PROJ) ~= 1 then return msg("err") end
drums_track = reaper.GetSelectedTrack(CUR_PROJ, 0) -- CURR_PROJ = 0
{% endhighlight %}

Then we obtain the first virtual instrument (target VSTi) and check whether it's really AD2 (for simplicity
multiple VSTis on track are not supported):

{% highlight lua %}
fx_idx = reaper.TrackFX_GetInstrument(drums_track)
if fx_idx == -1 then return msg("err") end

ok, fx_name = reaper.BR_TrackFX_GetFXModuleName(drums_track, fx_idx)
if not ok then return msg("err") end
if not fx_name:lower():find("addictive drums 2") then return msg("err") end
{% endhighlight %}

### Where to store multitracks

We also have to create a directory to store rendered files. Reaper has two options for that:

- `RENDER_FILE`, which is a render *directory* (obtained through call to `GetSetProjectInfo_String`), and
- `RECORD_PATH`, which is a recording directory (obtained through the former or `GetProjectPathEx`).

> "How is a render *file* setting called" you might ask? `RENDER_PATTERN`, and that makes sense as
 one may write a wildcard for rendering multiple files at once.

> `Ex` postfix states this function is a successor to deprecated `GetProjectPath`. Reaper sometimes uses
`2` (`GetProjectTimeSignature[2]`) for the same purpose.

Despite the first directory looks like a better choice, it's not that obvious. Render directory is usually a place
for submix/mix renders, something you take and upload somewhere, i.e. a _result_ (at least that's my interpretation
of its purpose).
The recording directory, on the contrary, is a place for _source files_ (where, for example, files recorded in Reaper are
placed). The drum mix looks more like a source file. However, I thought that a separate directory would be a better choice.
So, for example, my record path is `.\src`, and the path for target drum submixes (`render_folder`)
is a relative `..\src_mixing`.

{% highlight lua %}
record_path = reaper.GetProjectPathEx() .. "\\" .. render_folder
reaper.RecursiveCreateDirectory(record_path, 0)
{% endhighlight %}

In ordinary Lua I would rather write something like

{% highlight lua %}
require "path"
record_path = path.combine(reaper.GetProjectPathEx(), render_folder)
{% endhighlight %}

However, modules in Reaper Lua are not supported in general. There are
[some workarounds](https://forums.cockos.com/showthread.php?t=266220), but compiling a module and importing a
whole library for two usages (combine and normalize) doesn't look like a good trade-off.

Filesystem manipulation is partially supported in Lua, so we have to use Reaper-provided functions.

### Routing

Bad news: the [aforementioned option](#build-routing) can't be called from Lua so we have to build
multichannel output on our own. The algorithm is quite straightforward, but do we really want it?

By default, AD2 internal mixer channels output only to the internal master, and all other outputs remain silent.
The button that changes internal routing mode
[isn't an automation item](https://assets.xlnaudio.com/documents/addictive-drums-manual.pdf#page=46),
so it can't be changed outside of AD2 UI (its state is still saved in [presets](#improvements)).

{% spoiler I still want to do that %}
{% highlight lua %}
-- Assume you know plugin output index (chan_out_idx)
-- and channel mono/stereo status (is_mono) for channels to route (discussed further)
chan_idx = 0
reaper.InsertTrackAtIndex(chan_idx, false) --don't need envelopes on new track
track = reaper.GetTrack(CUR_PROJ, chan_idx)
track_send_idx = reaper.CreateTrackSend(DRUMS_MASTER, track)
if track < 0 then return msg("err") end
if is_mono then chan_out_idx |= 1024 end
ok = reaper.SetTrackSendInfo_Value(DRUMS_MASTER, 0, track_send_idx, "I_SRCCHAN", chan_out_idx)
if not ok then return msg("err") end
{% endhighlight %}
{% endspoiler %}

The only reason for writing this relatively complex code would be utilizing Reaper's parallel render.
If we select "render only selected tracks" in the render dialog and multiple tracks are selected, rendering will
use all available cores to process tracks simultaneously.
Unfortunately, we don't have automation items, therefore we fall back to rendering one part at a time, so
we don't need other tracks except for the one we have.

If we look at AD2 internal mixer, we [can see](https://assets.xlnaudio.com/pages/addictive_drums/new/GUI-edit-2x.jpg)
a "Solo" button (S) for each channel. This button (plugin `param` in Reaper
terminology) is an automation item so we can use it.

We can easily dump all AD2 params, their names, and their min/max values:

{% highlight lua %}
for i = 0, reaper.TrackFX_GetNumParams(TRACK, FX_ID) do
    current, min, max = reaper.TrackFX_GetParam(TRACK, FX_ID, i)
    name = reaper.TrackFX_GetParamName(TRACK, FX_ID, i)
    msg("%d %s %d %d", i, name, min, max)
end
{% endhighlight %}

{% spoiler Internal mixer solo channel params list %}
```
kick = 236, snare = 240, hihat = 244, hi_tom = 248,
med_tom = 252, lo_tom = 256 , floor_tom = 260,
flexi_1 = 264, flexi_2 = 268, flexi_3 = 272,
overhead = 277, room = 282, bus = 287
```
{% endspoiler %}

> Of course, it would be more convenient to address params by their names (e.g. "Kick Solo").
> Reaper has `TrackFX_GetParamFromIdent`,
> but AD2 identifiers are just parameter indices as strings, like `"2"`.

Then all we have to do is solo some channel, do the rendering, and un-solo it back:

{% highlight lua %}
-- boolean toggles are just floats
ok = reaper.TrackFX_SetParam(drums_track, fx_idx, solo_param_idx, 1.0)
if not ok then return msg("err") end
render(...)
ok = reaper.TrackFX_SetParam(drums_track, fx_idx, solo_param_idx, 0.0)
if not ok then return msg("err") end
{% endhighlight %}

### Rendering

As for rendering, Reaper API provides ~~surprisingly meager~~ zero functionality. There are no built-in functions.
Users wishing to render anything may only invoke provided rendering actions:

1. `File: Render project to disk...`, id `40015`.
2. `File: Render project, using the most recent render settings`, id `41824`.
3. `File: Render project, using the most recent render settings, auto-close render dialog`, id `42230`.
4. `File: Render project, using the most recent render settings, with a new target file name...`, id `41855`.

For the action list you may press `?` in Reaper, right-click on the action, and `Copy selected action command ID`
(don't use [Cockos website](https://wiki.cockos.com/wiki/index.php/Action_List_Reference), it's outdated).
Chances are high you will get different IDs.

> There is an alternate Render API (well, a full-blown Reaper API, actually) made by
> [Ultraschall](https://mespotin.uber.space/Ultraschall/US_Api_Introduction_and_Concepts.html#Introduction_001_Api)
> but it's a *HUGE* library I didn't dare to use.

All actions mentioned above use _most recent render settings_. Such settings define the output directory, file (or file
pattern), format, and more. The easiest solution is to back up some settings' keys, override them, render, and restore.
Copy-pasting each value setup and restoration would be boring, so I wrote an `exchange`-like function:

{% highlight lua %}
function xchg(key, value)
    local k, v = key, value -- to prevent variable overwrite in closures
    if type(v) == "string" then
        local ok, old_v = reaper.GetSetProjectInfo_String(CURR_PROJ, k, "", false)
        if not ok then return nil end
        reaper.GetSetProjectInfo_String(CURR_PROJ, k, v, true)
        return function() reaper.GetSetProjectInfo_String(CURR_PROJ, k, old_v, true) end
    else
        local old_v = reaper.GetSetProjectInfo(CURR_PROJ, k, 0, false)
        reaper.GetSetProjectInfo(CURR_PROJ, k, v, true)
        return function() reaper.GetSetProjectInfo(CURR_PROJ, k, old_v, true) end
    end
end
{% endhighlight %}

1. Checking value type is essential as Reaper has different functions for numeric and string values.
2. We can't just use the `GetSet...` function because it would set the new value and return it (instead of the old one;
  bizarre API).
3. Returning the rollback function is necessary as Lua has no `defer` functionality, and I couldn't get
  `reaper.defer` to work.

So usage follows:

{% highlight lua %}
r = xchg("foo", bar)
do_evil_stuff()
r() -- Ugly but better than plain copy-paste.
{% endhighlight %}

Here are the render settings we need to change:

| Name | Target value | Explanation |
|-|-|-|
| `RENDER_SETTINGS` | 3 | <a id="render-selected"></a> Render selected tracks only |
| `RENDER_BOUNDSFLAG` | 1 | Render entire project, start time to end time |
| `RENDER_CHANNELS` | 1 or 2 | Render mono or stereo |
| `RENDER_FILE` | ... | Target render directory |
| `RENDER_PATTERN` | ... | Target render file name |
| `RENDER_FORMAT` | `evaw` | Render .wav files |

The rest of the render function is just calling an action:

{% highlight lua %}
-- And we also need to delete the file we are rendering to.
-- Reaper would prompt whether we really want to overwrite it otherwise
os.remove(render_dir .. "\\" .. render_file .. ".wav")

RENDER_WITH_AUTO_CLOSE_ID = 42230
reaper.Main_OnCommandEx(RENDER_WITH_AUTO_CLOSE_ID, 0, CURR_PROJ)
{% endhighlight %}

The script finally started working, so I thought about packaging and improvements.

### Undo and redo

Undoing a MIDI modification or a track creation is quite trivial, but what about rendering a track?
I decided to match Reaper API: the track can't be "unrendered". We can delete the rendered file, but we can't restore
its previous version. We can rename the file and store the new version, but what about subsequent renders?
It's truly Pandora's box, so rendered files are not touched. All previous state is restored.

### Packaging

The best way to manage scripts in Reaper is to use [ReaPack](https://reapack.com/).
1. Create a Github repository,
2. Add your plugin into one of the supported subfolders ("Rendering" in my case),
3. clone [`reapack-index`](https://github.com/cfillion/reapack-index),
4. invoke it on the repository,
5. ~~install 150MB of dependencies because this script requires pandoc~~,

and you're done. This script is also available in ReaPack if you add my repository
`https://raw.githubusercontent.com/myrrc/reaper_scripts/master/index.xml`.

## Improvements

As I mentioned in the beginning, a mixing engineer would typically expect you to send two kick and two snare tracks.
AD2 has a "Beater/Front" slider for kick and a "Top/Bottom" slider for the snare which control a blend of two mike
channels (-100% is Beater only, 100% is Front only, 0% is 50/50% mix). However, this slider isn't an automation item
either.

While playing with AD2 Presets, I saw that the slider position was saved with preset, so I opened an `.adpreset` file
created for that purpose. I found a Lua table with keys handling slider positions:
`AD2Sampler -> [SNAR; KICK] -> MicBalance`, an ordinary float. There was no way I could automate preset loading and
saving in AD2, but if I could load some custom Lua map into AD2...

In that case, I would also avoid rendering empty items (if the item was empty, it surely wouldn't make a sound).

But this surely is a topic for another article.

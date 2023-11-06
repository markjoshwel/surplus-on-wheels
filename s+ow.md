# surplus on wheels

surplus on wheels (s+ow) is a pure shell script to get your location using
`termux-location`, process it through [surplus](https://github.com/markjoshwel/surplus),
and send it to messaging service or wherever using [bridges](#bridges).

surplus was made to emulate sending your location through the iOS Shortcuts app, and
surplus on wheels complements it by running surplus automatically using a cron job.  
(but using it manually also works!)

- [installing](#installing)
  - [as a standalone script](#as-a-standalone-script)
  - [as a cron job](#as-a-cron-job)
  - [using installation scripts](#using-installation-scripts)
- [usage](#usage)
  - [environment variables](#environment-variables)
  - [faking locations](#faking-locations)
- [bridges](#bridges)
  - [bring your own bridge](#bring-your-own-bridge)
- [licence](#licence)

## installing

> [!IMPORTANT]  
> s+ow is a Termux-first script, and will not work anywhere else unless you have
> a utility that emulates [termux-location](https://wiki.termux.com/wiki/termux-location)
> on `$PATH` alongside bridges that supports your platform.

there are two notable ways to install s+ow:

1. [as a standalone script](#as-a-standalone-script)
2. or, [as a cron job](#as-a-cron-job).

there is also an [installation script](#using-installation-scripts) for quickly getting
started from a _fresh_ termux installation.

### as a standalone script

1. firstly install python and termux-api if you haven't already:

   ```text
   pkg install python termux-api
   ```

   also install the accompanying the Termux:API app from [F-Froid](https://f-droid.org/en/packages/com.termux.api/).

2. install pipx

   ```text
   python3 -m pip install --user pipx
   ```

3. install surplus:

   ```text
   pip install https://github.com/markjoshwel/surplus/releases/latest/download/surplus-latest-py3-none-any.whl
   ```

4. install surplus on wheels:

   ```text
   mkdir -p ~/.local/bin/
   curl https://raw.githubusercontent.com/markjoshwel/surplus-on-wheels/main/s+ow > ~/.local/bin/s+ow
   chmod +x ~/.local/bin/s+ow
   ```

if `~/.local/bin` is not in your `$PATH`, add the following to your shell's rc file:

```shell
export PATH="$HOME/.local/bin:$PATH"
```

et voilà! s+ow is now setup. to actually send the message to a messaging platform,
[install an appropriate bridge](#bridges).

### as a cron job

> [!IMPORTANT]  
> these instructions rely on following the [previous instructions](#as-a-standalone-script).

1. install necessary packages to run cron jobs:

   ```text
   pkg install cronie termux-services
   ```

2. restart termux and start the cron service:

   ```text
   sv-enable cron
   ```

3. set up the cron job:

   > [!IMPORTANT]  
   > minimally fill in the `SPOW_TARGETS` variable before running s+ow.  
   > [(see usage for more info)](#usage)

   run the following command:

   ```text
   crontab -e
   ```

   and add the following text:

   ```text
   59 * * * *      SPOW_TARGETS="" SPOW_CRON=y ~/.local/bin/s+ow
   ```

   this will run s+ow every hour, a minute before the hour.

   modify the variables as per your needs.
   see [usage](#usage) for more information.

et voilà! s+ow will now send a message every hour. feel free to experiment with the cron
job to your liking. see [crontab.guru](https://crontab.guru/) if you’re new to cron jobs.

if you haven’t already, [install an appropriate bridge](#bridges) to actually send a
message to a messaging platform.

### using installation scripts

> [!WARNING]  
> these scripts assume you're starting from a fresh base installation of Termux.
> if you have already cron jobs, then manually carry out the instructiions in
> [as a cron job](#as-a-cron-job).

> [!IMPORTANT]  
> if not installed already, install [Termux:API](https://f-droid.org/en/packages/com.termux.api/)
> from F-Droid.

1. setup s+ow:

   ```text
   curl https://raw.githubusercontent.com/markjoshwel/surplus-on-wheels/main/termux-s+ow-setup | sh
   ```

2. restart termux!

3. setup cron job:

   ```text
   curl https://raw.githubusercontent.com/markjoshwel/surplus-on-wheels/main/termux-s+ow-setup-cron | sh
   ```

   the script will run `crontab -e`, and you can then edit the variables as per your
   needs. minimally fill in the `SPOW_TARGETS` variable before running s+ow.  
   see [usage](#usage) for more information.

et voilà! s+ow is now setup. to actually send the message to a messaging platform,
[install an appropriate bridge](#bridges).

## usage

### environment variables

s+ow uses three environment variables, two of which are optional:

1. `SPOW_TARGETS`  
   a single line of comma-deliminated chat IDs with bridge prefixes.

   ```text
   wa:000000000000000000@g.us,tg:-0000000000000000000,...
   ```

   in the example above, the WhatsApp chat ID is `wa:`-prefixed as recognised by the
   [spow-whatsapp-bridge](https://github.com/markjoshwel/spow-whatsapp-bridge), and the
   Telegram chat ID is `tg:`-prefixed as recognised by the
   [spow-telegram-bridge](https://github.com/markjoshwel/spow-telegram-bridge).

2. `SPOW_CRON` (optional)  
   set as non-empty to declare that s+ow is being run as a cron job.  
   if running as a cron job, start s+ow one minute earlier than intended to account for
   the time it takes to run `termux-location` and `surplus`.  
   s+ow will delay itself appropriately.

   setting it to `n` will also be treated as empty.

3. `LOCATION_PRIORITISE_NETWORK` (optional)  
   set as non-empty to declare that s+ow can just use network location instead of GPS
   if GPS is taking too long.  
   you should only turn this on if punctuality means that much to you, or you’re in a
   country with cell towers close by or everywhere, like Singapore.

   setting it to `n` will also be treated as empty.

the JIDs can be obtained by sending a message to the user/group, while running
`s+ow mdtest`, and examining the output for your message. JIDs are email address-like
strings.

### faking locations

> sometimes you gotta do what you gotta do

you can fake your s+ow messages by either:

1. setting a dummy `last` file in s+ow cache

   `$HOME/.cache/s+ow/last` is used as the fallback response when a part of s+ow (either
   `termux-location` or `surplus` errors out). you can set this file to whatever you want
   and just turn off location on your device.

2. setting a `fake` file in s+ow cache

   > [!WARNING]  
   > s+ow uses the `read` command to read the file. as such, it is possible for s+ow to
   > prematurely stop reading the file if the file does not contain a trailing newline.

   you can also write text to `$HOME/.cache/s+ow/fake` to fake upcoming messages. the file
   is delimited by empty lines. as such, arrange the file like so:

   ```text
   The Clementi Mall
   3155 Commonwealth Avenue West
   Westpeak Terrace
   129588
   Southwest, Singapore

   Westgate
   3 Gateway Drive
   Jurong East
   608532
   Southwest, Singapore

   ...

   ```

   on every run of s+ow, the first group of lines will be consumed, and the file will be
   updated with the remaining lines. if the file is empty, it will be deleted.

## bridges

there are two “official” bridges for s+ow:

- [spow-whatsapp-bridge](https://github.com/markjoshwel/spow-whatsapp-bridge)
- [spow-telegram-bridge](https://github.com/markjoshwel/spow-telegram-bridge)

bridges can be located anywhere, as long as they are reachable by the shell s+ow is
running in.

s+ow will run the bridges through definitions in in `$HOME/.s+ow-bridges`.
each line of `$HOME/.s+ow-bridges` is evaluated as a shell command, and is piped
`SPOW_TARGETS`.

### bring your own bridge

custom bridges are relatively easy as they are:

1. an executable or script

2. that reads `SPOW_TARGETS` (see [usage](#usage)) from stdin

   - bridges should account for the possibility of comma and space (`, ` instead of just
     `,`) delimited targets, and strip each target of preceding and trailing whitespace.

   - bridges should recognise a platform based on a prefix  
     (e.g. `wa:` for WhatsApp, `tg:` for Telegram, etc.)

   - bridges do not need to account for the possibility of multiple lines sent to stdin.

3. reads `SPOW_MESSAGE` (`~/.cache/spow/message`) for the message content

notes:

1. stderr and stdout are redirected to s+ow’s error and output logs respectively.
2. any errors encountered by the bridge should always result in a non-zero return.  
   error logs will show the exact error code, so feel free to use other numbers than 1.
3. persistent data such as credentials and session data storage are to be handled by the
   bridge itself.  
   consider storing them in `$HOME/.local/share/<bridge-name>/`, or similar.

## licence

surplus on wheels is free and unencumbered software released into the public domain.
for more information, please refer to [UNLICENCE](/UNLICENCE) or <http://unlicense.org/>.

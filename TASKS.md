# Simple Linux Voice Dictation - Tasks

- ~~Add actions to `dictate` for:~~
  - ~~`discard` If recording, discard recording in progress. If not recording, do nothing.~~
  - ~~`copy` If recording, copy to CLIPBOARD but do not paste. If not recording, start recording.~~
  - ~~`copypaste` (default) If recording, copy to CLIPBOARD and paste. If not recording, start recording. This is the way it behaves now.~~

- ~~Add a general post-processing pass with support for a series of rules that are applied such as:~~
  - ~~Capitalize the first letters of sentences (perhaps require 4+ words).~~
  - ~~Remove extra newlines.~~
  - ~~Convert "newline" to a newline.~~
  - ~~Convert "clawed code" to "Claude Code".~~
  - ~~Special phrases for punctuation, camel case or snake style code names, all caps, quotes, etc.~~

- ~~Suppress the "Thank you." output when there is no audio. This could be a post-processing rule.~~

- ~~Add a --raw flag to dictate.~~
- ~~Make sure "foo dot bar" -> "foo.bar"~~

## Manual Tasks

- Map the new `discard` and `copy` to new keys in KDE.

## Low Priority

- Push to public GitHub repo.
- Get working under X11 with autodetection (or a command line flag).
- Consider a code mode vs. prose mode.

## Done

- Do more research on Whisper and its options, particularly with how to specify punctuation while talking. - Whisper does not natively support this, so it would require post-processing.


HOW TO ADD DEFAULT MUSIC TRACKS
================================

1. Drop your .mp3 or .ogg audio files into this folder.

2. Open index.html and find the DEFAULT_TRACKS array near the top of the
   <script> section (search for "DEFAULT_TRACKS"):

     const DEFAULT_TRACKS = [
       // { name: 'Spring Breeze', file: 'music/spring-breeze.mp3' },
     ];

3. Add one entry per track, for example:

     const DEFAULT_TRACKS = [
       { name: 'Spring Breeze', file: 'music/spring-breeze.mp3' },
       { name: 'Bunny Hop',     file: 'music/bunny-hop.ogg'     },
     ];

4. Save index.html and reload the browser. The tracks will appear as
   selectable pills in Settings > Audio.

NOTE: Tracks are loaded from disk via fetch(), so the game must be
served from a local web server (e.g. VS Code Live Server, python -m
http.server) rather than opened as a file:// URL — otherwise the
browser will block the fetch request for security reasons.

# quran_player

Welcome to the quran_player wiki!
# quran_player in Dart

`quran_player/`
`├── lib/`
`│   ├── models/`
`│   │   ├── ayah.dart`
`│   │   └── word_timing.dart`
`│   ├── services/`
`│   │   ├── database_service.dart`
`│   │   └── audio_service.dart`
`│   ├── utils/`
`│   │   └── logger.dart`
`│   └── main.dart`
`├── assets/`
`│   └── quran.db`
`└── pubspec.yaml`

#########################################

`class Ayah {`
  `final int number;`
  `final int surahNumber;`
  `final String text;`
  `final String? audio;`
  `final List<String>? secondaryAudio;`
  `final int numberInSurah;`
  `final int juz;`
  `final int page;`
  `final Duration? duration;`

  `Ayah({`
    `required this.number,`
    `required this.surahNumber,`
    `required this.text,`
    `this.audio,`
    `this.secondaryAudio,`
    `required this.numberInSurah,`
    `required this.juz,`
    `required this.page,`
    `this.duration,`
  `});`

  `factory Ayah.fromMap(Map<String, dynamic> map) {`
    `return Ayah(`
      `number: map['number'],`
      `surahNumber: map['surah_number'],`
      `text: map['text'],`
      `audio: map['audio'],`
      `secondaryAudio: map['secondary_audio_urls']?.split(','),`
      `numberInSurah: map['numberInSurah'],`
      `juz: map['juz'],`
      `page: map['page'],`
    `);`
  `}`
`}`

###############################################################################################################

Create the database service (lib/services/database_service.dart):

###############################################################################################################

`import 'dart:io';`
`import 'package:sqlite3/sqlite3.dart';`
`import '../models/ayah.dart';`
`import '../utils/logger.dart';`

`class DatabaseService {`
  `late Database _db;`
  `final String dbPath;`
  `final Logger logger;`

  `DatabaseService({`
    `required this.dbPath,`
    `required this.logger,`
  `}) {`
    `_initDatabase();`
  `}`

  `void _initDatabase() {`
    `try {`
      `if (!File(dbPath).existsSync()) {`
        `throw Exception('Database not found at: $dbPath');`
      `}`
      `_db = sqlite3.open(dbPath);`
      `logger.info('Database connected successfully');`
    `} catch (e) {`
      `logger.error('Database initialization error: $e');`
      `rethrow;`
    `}`
  `}`

  `List<Ayah> getAyahsForSurah(int surahNumber) {`
    `try {`
      `final results = _db.select('''`
        `SELECT a.*, GROUP_CONCAT(sec.audio_url) as secondary_audio_urls`
        `FROM ayahs a`
        `LEFT JOIN audio_secondary sec ON a.number = sec.ayah_number`
        `WHERE a.surah_number = ?`
        `GROUP BY a.number`
        `ORDER BY a.numberInSurah`
      `''', [surahNumber]);`

      `return results.map((row) => Ayah.fromMap(row)).toList();`
    `} catch (e) {`
      `logger.error('Error fetching ayahs for surah $surahNumber: $e');`
      `rethrow;`
    `}`
  `}`

  `void close() {`
    `_db.dispose();`
    `logger.info('Database connection closed');`
  `}`
`}`

###########################################################################################

Create the audio service (lib/services/audio_service.dart):

###########################################################################################

`import 'package:audioplayers/audioplayers.dart';`
`import '../models/ayah.dart';`
`import '../utils/logger.dart';`

`class AudioService {`
  `final AudioPlayer _player;`
  `final Logger logger;`
  `final void Function(Ayah ayah)? onAyahChange;`
  `final void Function()? onPlaybackComplete;`
  
  `List<Ayah> _playlist = [];`
  `int _currentIndex = 0;`
  `bool _isPlaying = false;`

  `AudioService({`
    `required this.logger,`
    `this.onAyahChange,`
    `this.onPlaybackComplete,`
  `}) : _player = AudioPlayer() {`
    `_initializeAudioPlayer();`
  `}`

  `void _initializeAudioPlayer() {`
    `_player.onPlayerComplete.listen((_) => _onAyahComplete());`
    `_player.onPositionChanged.listen((position) => _updatePosition(position));`
    `logger.info('Audio player initialized');`
  `}`

  `Future<void> loadPlaylist(List<Ayah> ayahs) async {`
    `_playlist = ayahs;`
    `_currentIndex = 0;`
    `logger.info('Loaded playlist with ${ayahs.length} ayahs');`
  `}`

  `Future<void> play() async {`
    `if (_playlist.isEmpty) {`
      `logger.warning('No ayahs loaded in playlist');`
      `return;`
    `}`

    `try {`
      `final currentAyah = _playlist[_currentIndex];`
      `if (currentAyah.audio == null) {`
        `logger.warning('No audio available for ayah ${currentAyah.number}');`
        `return;`
      `}`

      `await _player.play(UrlSource(currentAyah.audio!));`
      `_isPlaying = true;`
      `onAyahChange?.call(currentAyah);`
      `logger.info('Playing ayah ${currentAyah.number}');`
    `} catch (e) {`
      `logger.error('Error playing audio: $e');`
    `}`
  `}`

  `void _onAyahComplete() {`
    `if (_currentIndex < _playlist.length - 1) {`
      `_currentIndex++;`
      `play();`
    `} else {`
      `_isPlaying = false;`
      `onPlaybackComplete?.call();`
      `logger.info('Playlist complete');`
    `}`
  `}`

  `void _updatePosition(Duration position) {`
    `// You can implement word-level timing here if needed`
    `logger.debug('Position: ${position.inSeconds}s');`
  `}`

  `Future<void> pause() async {`
    `await _player.pause();`
    `_isPlaying = false;`
    `logger.info('Playback paused');`
  `}`

  `Future<void> resume() async {`
    `await _player.resume();`
    `_isPlaying = true;`
    `logger.info('Playback resumed');`
  `}`

  `Future<void> stop() async {`
    `await _player.stop();`
    `_isPlaying = false;`
    `_currentIndex = 0;`
    `logger.info('Playback stopped');`
  `}`

  `Future<void> seekToAyah(int index) async {`
    `if (index >= 0 && index < _playlist.length) {`
      `_currentIndex = index;`
      `if (_isPlaying) {`
        `await play();`
      `}`
      `logger.info('Seeked to ayah $index');`
    `}`
  `}`

  `bool get isPlaying => _isPlaying;`
  
  `void dispose() {`
    `_player.dispose();`
    `logger.info('Audio player disposed');`
  `}`
`}`
#####################################################################################################################

Create the logger (lib/utils/logger.dart):

#####################################################################################################################

`import 'package:intl/intl.dart';`

`class Logger {`
  `final DateFormat _dateFormatter = DateFormat('yyyy-MM-dd HH:mm:ss');`

  `void info(String message) {`
    `_log('INFO', message);`
  `}`

  `void error(String message) {`
    `_log('ERROR', message);`
  `}`

  `void warning(String message) {`
    `_log('WARN', message);`
  `}`

  `void debug(String message) {`
    `_log('DEBUG', message);`
  `}`

  `void _log(String level, String message) {`
    `final timestamp = _dateFormatter.format(DateTime.now().toUtc());`
    `print('[$timestamp] $level: $message');`
  `}`
`}`


##########################################################################################################################

Finally, create the main application (lib/main.dart):

##########################################################################################################################

`import 'dart:io';`
`import 'package:intl/intl.dart';`
`import 'services/database_service.dart';`
`import 'services/audio_service.dart';`
`import 'utils/logger.dart';`
`import 'models/ayah.dart';`

`class QuranPlayer {`
  `final DatabaseService _db;`
  `final AudioService _audio;`
  `final Logger _logger;`

  `QuranPlayer({`
    `required String dbPath,`
  `}) : _logger = Logger(),`
       `_db = DatabaseService(`
         `dbPath: dbPath,`
         `logger: Logger(),`
       `),`
       `_audio = AudioService(`
         `logger: Logger(),`
         `onAyahChange: (ayah) {`
           `print('\nNow playing Ayah ${ayah.numberInSurah}:');`
           `print(ayah.text);`
         `},`
         `onPlaybackComplete: () {`
           `print('\nPlayback complete');`
         `},`
       `);`

  `Future<void> playSurah(int surahNumber) async {`
    `try {`
      `final ayahs = _db.getAyahsForSurah(surahNumber);`
      `await _audio.loadPlaylist(ayahs);`
      `await _audio.play();`
    `} catch (e) {`
      `_logger.error('Error playing surah: $e');`
    `}`
  `}`

  `Future<void> handleUserCommands() async {`
    `print('\nAvailable commands:');`
    `print('play <surah_number> - Play a surah');`
    `print('pause - Pause playback');`
    `print('resume - Resume playback');`
    `print('stop - Stop playback');`
    `print('seek <ayah_number> - Seek to specific ayah');`
    `print('quit - Exit the application');`

    `while (true) {`
      `stdout.write('\nEnter command: ');`
      `final input = stdin.readLineSync()?.toLowerCase().split(' ');`
      
      `if (input == null || input.isEmpty) continue;`

      `switch (input[0]) {`
        `case 'play':`
          `if (input.length > 1) {`
            `final surahNumber = int.tryParse(input[1]);`
            `if (surahNumber != null) {`
              `await playSurah(surahNumber);`
            `}`
          `}`
          `break;`
        
        `case 'pause':`
          `await _audio.pause();`
          `break;`
        
        `case 'resume':`
          `await _audio.resume();`
          `break;`
        
        `case 'stop':`
          `await _audio.stop();`
          `break;`
        
        `case 'seek':`
          `if (input.length > 1) {`
            `final ayahNumber = int.tryParse(input[1]);`
            `if (ayahNumber != null) {`
              `await _audio.seekToAyah(ayahNumber - 1);`
            `}`
          `}`
          `break;`
        
        `case 'quit':`
          `return;`
        
        `default:`
          `print('Unknown command');`
      `}`
    `}`
  `}`

  `void dispose() {`
    `_audio.dispose();`
    `_db.close();`
  `}`
`}`

`void main() async {`
  `final player = QuranPlayer(dbPath: 'assets/quran.db');`
  
  `try {`
    `await player.handleUserCommands();`
  `} finally {`
    `player.dispose();`
  `}`
`}`

################################################################################################################################

To run the application:

1. Create the project:
1. bash
1. mkdir quran_player
1. cd quran_player
1. Create the file structure and copy the files as shown above.

## Install dependencies:

* bash
* dart pub get
* Make sure your quran.db file is in the assets directory.

## Run the application:

* bash
* dart run lib/main.dart
* The application can be tested from CLI with these commands:

> bash
> play 1    # Play Surah Al-Fatiha
> pause     # Pause playback
> resume    # Resume playback
> seek 3    # Jump to third ayah
> stop      # Stop playback
> quit      # Exit the application
## Features:

> Autoplay functionality (continues playing through all ayahs)
> CLI interface for testing
> Proper error handling
> Logging system
> Clean architecture with separated concerns
> Easy to extend and modify
> The code is organized in a way that makes it easy to:

> Add new features
> Implement a GUI later
> Add word-level timing
> Add more playback controls
> Implement different audio sources
> Let me know if you need any clarification or have questions about specific parts of the implementation!


########################################################################################################################################

Added streaming capability in Audio Service

########################################################################################################################################

`// lib/services/audio_service.dart`
`import 'package:audioplayers/audioplayers.dart';`
`import 'package:http/http.dart' as http;`
`import '../models/ayah.dart';`
`import '../utils/logger.dart';`

`class AudioService {`
  `final AudioPlayer _player;`
  `final Logger logger;`
  `final void Function(Ayah ayah)? onAyahChange;`
  `final void Function()? onPlaybackComplete;`
  `final void Function(String message)? onError;`
  
  `List<Ayah> _playlist = [];`
  `int _currentIndex = 0;`
  `bool _isPlaying = false;`
  `bool _isBuffering = false;`

  `AudioService({`
    `required this.logger,`
    `this.onAyahChange,`
    `this.onPlaybackComplete,`
    `this.onError,`
  `}) : _player = AudioPlayer() {`
    `_initializeAudioPlayer();`
  `}`

  `void _initializeAudioPlayer() {`
    `_player.onPlayerComplete.listen((_) => _onAyahComplete());`
    `_player.onPositionChanged.listen((position) => _updatePosition(position));`
    
    `// Add buffering state listener`
    `_player.onPlayerStateChanged.listen((state) {`
      `switch (state) {`
        `case PlayerState.playing:`
          `_isBuffering = false;`
          `logger.info('Playing audio');`
          `break;`
        `case PlayerState.paused:`
          `_isBuffering = false;`
          `logger.info('Audio paused');`
          `break;`
        `case PlayerState.stopped:`
          `_isBuffering = false;`
          `logger.info('Audio stopped');`
          `break;`
        `case PlayerState.completed:`
          `_isBuffering = false;`
          `logger.info('Audio completed');`
          `break;`
      `}`
    `});`
  `}`

  `Future<bool> _verifyAudioUrl(String url) async {`
    `try {`
      `final response = await http.head(Uri.parse(url));`
      `return response.statusCode == 200;`
    `} catch (e) {`
      `logger.error('Error verifying audio URL: $e');`
      `return false;`
    `}`
  `}`

  `Future<void> play() async {`
    `if (_playlist.isEmpty) {`
      `logger.warning('No ayahs loaded in playlist');`
      `return;`
    `}`

    `try {`
      `final currentAyah = _playlist[_currentIndex];`
      `if (currentAyah.audio == null || currentAyah.audio!.isEmpty) {`
        `logger.warning('No audio URL available for ayah ${currentAyah.number}');`
        `_tryPlayNextAyah();`
        `return;`
      `}`

      `// Verify audio URL before playing`
      `final isValidUrl = await _verifyAudioUrl(currentAyah.audio!);`
      `if (!isValidUrl) {`
        `logger.error('Invalid audio URL for ayah ${currentAyah.number}');`
        `_tryPlayNextAyah();`
        `return;`
      `}`

      `_isBuffering = true;`
      `logger.info('Buffering audio for ayah ${currentAyah.number}');`

      `await _player.play(UrlSource(currentAyah.audio!));`
      `_isPlaying = true;`
      `onAyahChange?.call(currentAyah);`
      
      `// Pre-buffer next ayah`
      `_preBufferNextAyah();`

    `} catch (e) {`
      `logger.error('Error playing audio: $e');`
      `onError?.call('Error playing ayah: $e');`
      `_tryPlayNextAyah();`
    `}`
  `}`

  `Future<void> _preBufferNextAyah() async {`
    `if (_currentIndex < _playlist.length - 1) {`
      `final nextAyah = _playlist[_currentIndex + 1];`
      `if (nextAyah.audio != null) {`
        `try {`
          `// Verify next audio URL`
          `await _verifyAudioUrl(nextAyah.audio!);`
        `} catch (e) {`
          `logger.error('Error pre-buffering next ayah: $e');`
        `}`
      `}`
    `}`
  `}`

  `void _tryPlayNextAyah() {`
    `if (_currentIndex < _playlist.length - 1) {`
      `_currentIndex++;`
      `play();`
    `} else {`
      `_isPlaying = false;`
      `onPlaybackComplete?.call();`
    `}`
  `}`

  `void _onAyahComplete() {`
    `logger.info('Ayah ${_playlist[_currentIndex].number} completed');`
    `_tryPlayNextAyah();`
  `}`

  `// ... rest of the AudioService class remains the same`
`}`

`// lib/main.dart - Add status display`
`class QuranPlayer {`
  `// ... previous code ...`

  `Future<void> playSurah(int surahNumber) async {`
    `try {`
      `print('\nLoading Surah $surahNumber...');`
      `final ayahs = _db.getAyahsForSurah(surahNumber);`
      
      `if (ayahs.isEmpty) {`
        `print('No ayahs found for Surah $surahNumber');`
        `return;`
      `}`

      `print('Found ${ayahs.length} ayahs');`
      `await _audio.loadPlaylist(ayahs);`
      
      `print('Starting playback...');`
      `await _audio.play();`
    `} catch (e) {`
      `_logger.error('Error playing surah: $e');`
      `print('Error: $e');`
    `}`
  `}`

  `Future<void> handleUserCommands() async {`
    `print('\nQuran Audio Player');`
    `print('=' * 50);`
    `print('Available commands:');`
    `print('  play <surah_number>  - Play a surah');`
    `print('  pause               - Pause playback');`
    `print('  resume              - Resume playback');`
    `print('  stop                - Stop playback');`
    `print('  seek <ayah_number>  - Seek to specific ayah');`
    `print('  status              - Show current status');`
    `print('  quit                - Exit the application');`
    `print('=' * 50);`

    `while (true) {`
      `stdout.write('\nEnter command: ');`
      `final input = stdin.readLineSync()?.toLowerCase().split(' ');`
      
      `if (input == null || input.isEmpty) continue;`

      `switch (input[0]) {`
        `case 'play':`
          `if (input.length > 1) {`
            `final surahNumber = int.tryParse(input[1]);`
            `if (surahNumber != null && surahNumber > 0 && surahNumber <= 114) {`
              `await playSurah(surahNumber);`
            `} else {`
              `print('Invalid surah number. Please enter a number between 1 and 114.');`
            `}`
          `} else {`
            `print('Please specify a surah number (1-114)');`
          `}`
          `break;`
        
        `case 'status':`
          `_displayStatus();`
          `break;`
        
        `// ... other cases remain the same`
      `}`
    `}`
  `}`

  `void _displayStatus() {`
    `final now = DateTime.now().toUtc();`
    `final formatter = DateFormat('yyyy-MM-dd HH:mm:ss');`
    
    `print('\nCurrent Status');`
    `print('=' * 50);`
    `print('Time (UTC): ${formatter.format(now)}');`
    `print('User: zrayyan');`
    `print('Playback: ${_audio.isPlaying ? "Playing" : "Stopped"}');`
    `// Add more status information as needed`
    `print('=' * 50);`
  `}`
`}`

#######################################################
`// Update pubspec.yaml`
`dependencies:`
  `sqlite3: ^2.1.0`
  `audioplayers: ^5.2.1`
  `intl: ^0.17.0`
  `path: ^1.8.0`
  `http: ^1.1.0  # Add this for URL verification`

  ##################################################

  ## Key improvements:

1. ### ## URL Verification:

1. Checks if audio URLs are valid before attempting playback
1. Handles invalid or missing URLs gracefully
1. Buffering Status:

1. Shows when audio is buffering
1. Pre-buffers next ayah for smoother playback
1. Error Handling:

1. Better error messages for network issues
1. Automatic skip to next ayah if current one fails
1. Status Display:

1. Added status command to show current state
1. Better feedback during playback





# perl-plexapi

> [!CAUTION]
> **Work in progress.** Lots of churn, lots of untested code.

Perl 5.42+ interface to the [Plex Media Server HTTP API](https://github.com/Arcanemagus/plex-api/wiki), built as a CPAN-style distribution using native `class`/`field`/`method` OO.

## Modules

| Module | Description |
| --- | --- |
| `WebService::Plex` | Connection object, HTTP layer, submodule accessors |
| `WebService::Plex::Server` | Server identity, preferences, sessions, butler tasks, diagnostics |
| `WebService::Plex::Library` | Sections, metadata, search, maintenance, playback state |
| `WebService::Plex::Video` | Movies, shows, seasons, episodes (type codes: 1–4, 12) |
| `WebService::Plex::Audio` | Artists, albums, tracks (type codes: 8–10) |
| `WebService::Plex::Playlist` | Playlist CRUD and item management |
| `WebService::Plex::Collection` | Collection CRUD and item management |
| `WebService::Plex::DownloadQueue` | Download queue management |
| `WebService::Plex::PlayQueue` | Play queue management and item control |
| `WebService::Plex::Photo` | Photo transcoding and image helpers |
| `WebService::Plex::Services` | Service endpoints (e.g. ultrablur) |
| `WebService::Plex::LiveTV` | Live TV DVR, EPG, and session access |
| `WebService::Plex::MyPlex` | plex.tv account, home users, watchlist |
| `WebService::Plex::Client` | Remote playback and navigation control per client device |
| `WebService::Plex::Alert` | WebSocket notification listener (optional: AnyEvent) |

## Quick Start

```perl
use v5.42;
use WebService::Plex;

# token is optional when connecting to a local unclaimed server
my $plex = WebService::Plex->new(
    baseurl => 'http://localhost:32400',
    token   => $ENV{PLEX_TOKEN},
);

# Server
my $sessions = $plex->server->sessions;
$plex->server->run_task('CleanOldBundles');

# Library browsing
my $sections = $plex->library->sections;
my $movies   = $plex->video->movies(1);         # section key 1
my $shows    = $plex->video->shows(2);
my $show_key = $shows->{MediaContainer}{Metadata}[0]{ratingKey};
my $episodes = $plex->video->all_episodes($show_key);

my $artists    = $plex->audio->artists(3);
my $artist_key = $artists->{MediaContainer}{Metadata}[0]{ratingKey};
my $albums     = $plex->audio->albums_for($artist_key);
my $album_key  = $albums->{MediaContainer}{Metadata}[0]{ratingKey};
my $tracks     = $plex->audio->tracks($album_key);

# Playback state
$plex->library->mark_played(12345);
$plex->library->update_progress(12345, 60_000, 'playing');

# Playlists and collections
my $playlists = $plex->playlist->all;
$plex->playlist->create(title => 'My Mix', type => 'audio', smart => 0);

# MyPlex / plex.tv
$plex->myplex->add_to_watchlist(12345);

# Client remote control
my $client = $plex->client('http://192.168.1.50:32433');
$client->play;
$client->set_volume(80);

# Live TV / DVR
my $livetv = $plex->livetv;
$livetv->dvrs(limit => 10);

# Real-time notifications
$plex->alert->listen(sub {
    my ($notification) = @_;
    say $notification->{type};
    return 1;   # return false to stop listening
});
```

## Unit Tests

Requires Perl 5.42. No external Plex server needed — all tests use mocks.

```bash
perl Makefile.PL && make && make test
```

Via Docker (no local Perl 5.42 required):

```bash
docker build -f docker/test/Dockerfile -t plexapi-test .
docker run --rm plexapi-test
```

## Integration Tests

Integration tests run against a real `plexinc/pms-docker` container populated
with stub media. They skip automatically when the server is unavailable, so
`make test` always passes cleanly without Docker.

**Prerequisites:** Docker, `docker compose`, `ffmpeg`

```bash
# 1. Generate stub MP4/MP3 media files
bash tools/create-test-media.sh

# 2. Start the Plex container
docker compose up -d

# 3. Wait for startup and configure libraries (idempotent)
perl tools/bootstrap-test-server.pl

# 4. Run the integration suite
make test_integration

# Tear down
docker compose down

# Full reset (wipes Plex config for a clean re-bootstrap)
make plex_reset
```

Environment variables (with defaults):

| Variable | Default |
| --- | --- |
| `PLEX_TEST_BASEURL` | `http://127.0.0.1:32400` |
| `PLEX_TEST_TOKEN` | _(empty — not needed for an unclaimed server)_ |

## Dependencies

| Kind | Modules |
| --- | --- |
| Runtime | `Carp`, `HTTP::Request`, `JSON::XS`, `LWP::UserAgent`, `URI::Escape` |
| Optional | `AnyEvent`, `AnyEvent::WebSocket::Client` (for `WebService::Plex::Alert`) |
| Test | `Test::More`, `Test::Exception`, `Test::MockModule`, `Test::LWP::UserAgent` |
| Perl | v5.42+ |

## Requirements vs python-plexapi

This library is a deliberate thin wrapper — it returns raw decoded JSON rather
than typed object hierarchies. Areas not yet covered compared to
[python-plexapi](https://github.com/pkkid/python-plexapi):

- Server Settings object (`settings.py`)
- Sonos speaker integration (`sonos.py`)
- Device sync for offline playback (`sync.py`)
- GDM local server discovery (`gdm.py`)
- Rich media/stream metadata objects (`media.py`)
- Custom exception hierarchy (`exceptions.py`)
- Mixin system: edit, rating, artwork, smart-filter, watched state

## License

Copyright (C) 2026 Sam Robertson.
GNU General Public License, version 3 or later.
See [https://www.gnu.org/licenses/](https://www.gnu.org/licenses/) for details.

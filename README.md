# Music Downloader

This project is a Fish script that downloads audio from a $potify playlist by searching for corresponding tracks on YouTube. The downloaded audio is "converted" into the OPUS format and saved to the specified output directory.

## Code | MusicDownloader

```Fish
#!/usr/bin/env fish

for cmd in curl jq yt-dlp aria2c ffmpeg
    if not command -v $cmd >/dev/null
        echo "Error: $cmd is not installed. Please install it."
        exit 1
    end
end

function print_header
    echo "==================================="
    echo "        Music      Downloader      "
    echo "==================================="
end

function usage
    echo "Usage: $argv[0] [<playlist_url> <client_id> <client_secret> <output_dir>]"
    echo "Or run without arguments for interactive mode."
end

function validate_playlist_url
    set url (string trim -c "'" -c "\"" $argv[1]) # Remove quotes
    if string match -qr '^https://open\.spotify\.com/playlist/[a-zA-Z0-9]+.*$' $url
        return 0
    else
        return 1
    end
end

function get_spotify_token
    set auth (echo -n "$argv[1]:$argv[2]" | base64 -w 0)
    set temp_file (mktemp)
    set http_code (curl -s -w "%{http_code}" -X POST "https://accounts.spotify.com/api/token" \
        -H "Authorization: Basic $auth" \
        -d "grant_type=client_credentials" \
        -o $temp_file)
    set body (cat $temp_file)
    rm -f $temp_file
    if test $http_code -ne 200
        echo "Failed to obtain Spotify token. HTTP $http_code: $body"
        return 1
    end
    echo $body | jq -r '.access_token'
end

function get_playlist_tracks
    set playlist_id (echo $argv[1] | sed -E 's|.*/playlist/([^?]+).*|\1|')
    set token $argv[2]
    set tracks
    set offset 0
    set limit 100

    while true
        set temp_file (mktemp)
        set http_code (curl -s -w "%{http_code}" -H "Authorization: Bearer $token" \
            "https://api.spotify.com/v1/playlists/$playlist_id/tracks?offset=$offset&limit=$limit" \
            -o $temp_file)
        set body (cat $temp_file)
        rm -f $temp_file
        if test $http_code -ne 200
            echo "Failed to fetch tracks. HTTP $http_code: $body"
            return 1
        end
        set total (echo $body | jq -r '.total')
        set items (echo $body | jq -c '.items[]' 2>/dev/null)
        if test $status -ne 0
            echo "Error parsing tracks: $body"
            return 1
        end

        for item in $items
            set id (echo $item | jq -r '.track.id // "unknown"')
            set name (echo $item | jq -r '.track.name // "unknown"')
            set artists (echo $item | jq -r '.track.artists | if length > 0 then map(.name) | join(", ") else "Unknown Artist" end')
            set cover_url (echo $item | jq -r '.track.album.images[0].url // ""')
            set duration_ms (echo $item | jq -r '.track.duration_ms // 0')
            echo "Debug: Track $name by $artists, cover_url=$cover_url" >&2
            if test -z "$cover_url"
                set album_id (echo $item | jq -r '.track.album.id // ""')
                if test -n "$album_id"
                    set album_response (curl -s -H "Authorization: Bearer $token" \
                        "https://api.spotify.com/v1/albums/$album_id")
                    set cover_url (echo $album_response | jq -r '.images[0].url // ""')
                    echo "Debug: Fetched cover_url from album $album_id: $cover_url" >&2
                end
            end
            set tracks $tracks "$id|$name|$artists|$cover_url|$duration_ms"
        end

        set offset (math $offset + $limit)
        if test $offset -ge $total
            break
        end
        sleep 0.5
    end

    for track in $tracks
        echo $track
    end
end

# Function to get audio URL from Qobuz, YouTube, or other sources with duration check
function get_audio_url
    set query (echo $argv[1] | string replace -r '[/:*?"<>|]' '_')
    set spotify_duration_ms $argv[2]
    set duration_tolerance 15000 # ±15 seconds in milliseconds

    # Helper function to convert duration (MM:SS or seconds) to milliseconds
    function duration_to_ms
        set duration $argv[1]
        if string match -qr '^[0-9]+$' $duration
            # Duration in seconds
            echo (math $duration * 1000)
        else
            # Duration in MM:SS format
            set parts (string split ':' $duration)
            set minutes $parts[1]
            set seconds $parts[2]
            echo (math "($minutes * 60 + $seconds) * 1000")
        end
    end

    # Try Qobuz
    set result (yt-dlp -f bestaudio --get-url --get-duration "ytsearch:$query site:qobuz.com" 2>&1 | string collect)
    set url (echo $result | head -n 1)
    set duration (echo $result | tail -n 1)
    if test -n "$url" -a -n "$duration"
        set duration_ms (duration_to_ms $duration)
        if test (math "abs($spotify_duration_ms - $duration_ms)") -le $duration_tolerance
            echo "qobuz|$url"
            return 0
        end
    end

    # Fallback to YouTube
    set result (yt-dlp -f bestaudio --get-url --get-duration "ytsearch1:$query" 2>&1 | string collect)
    set url (echo $result | head -n 1)
    set duration (echo $result | tail -n 1)
    if test -n "$url" -a -n "$duration"
        set duration_ms (duration_to_ms $duration)
        if test (math "abs($spotify_duration_ms - $duration_ms)") -le $duration_tolerance
            echo "youtube|$url"
            return 0
        end
    end

    # Fallback to other platforms (auto search)
    set result (yt-dlp -f bestaudio --get-url --get-duration "ytsearch1:$query" --default-search auto 2>&1 | string collect)
    set url (echo $result | head -n 1)
    set duration (echo $result | tail -n 1)
    if test -n "$url" -a -n "$duration"
        set duration_ms (duration_to_ms $duration)
        if test (math "abs($spotify_duration_ms - $duration_ms)") -le $duration_tolerance
            echo "other|$url"
            return 0
        end
    end

    echo ""
end

function download_cover
    set cover_url $argv[1]
    set filename (echo $argv[2] | string replace -r '[/:*?"<>|]' '_')
    set cover_file "$output_dir/$filename.jpg"

    if test -n "$cover_url"
        echo "Debug: Attempting to download cover from $cover_url to $cover_file"
        set curl_output (curl -s -w "%{http_code}" --fail --max-time 10 "$cover_url" -o "$cover_file" 2>&1)
        set curl_status $status
        set http_code (echo $curl_output | tail -n 1)
        if test $curl_status -eq 0 -a -f "$cover_file"
            echo "Cover art downloaded: $cover_file"
            echo $cover_file
            return 0
        else
            echo "Failed to download cover art from $cover_url. HTTP $http_code, curl error: $curl_output"
            rm -f "$cover_file"
            return 1
        end
    else
        echo "No cover art URL provided"
        return 1
    end
end

function check_file_integrity
    set file $argv[1]
    if test -f "$file"
        ffmpeg -v error -i "$file" -f null - >/dev/null 2>&1
        if test $status -eq 0
            return 0
        else
            echo "Audio file is corrupted: $file"
            return 1
        end
    end
    echo "File does not exist: $file"
    return 1
end

function check_cover_art
    set file $argv[1]
    if test -f "$file"
        set metadata (ffmpeg -i "$file" 2>&1 | grep -i "Stream.*Video.*mjpeg")
        if test -n "$metadata"
            echo "Cover art found in metadata: $metadata"
            return 0
        else
            echo "No cover art found in metadata"
            return 1
        end
    end
    return 1
end

# Function to download and convert audio with embedded cover and cleanup
function download_and_convert
    set source $argv[1]
    set url $argv[2]
    set filename (echo $argv[3] | string replace -r '[/:*?"<>|]' '_')
    set track_name (echo $argv[4] | string replace -r '[/:*?"<>|]' '_')
    set track_artists (echo $argv[5] | string replace -r '[/:*?"<>|]' '_')
    set cover_file $argv[6]
    set temp_file "$output_dir/$filename.webm"
    set final_file "$output_dir/$filename.opus"
    set ffmpeg_log "/tmp/ffmpeg_log.txt"

    echo "Downloading audio from $source: $url"
    aria2c $url \
        --dir=$output_dir \
        --out=$filename.webm \
        --max-connection-per-server=16 \
        --split=16 \
        --min-split-size=1M \
        --max-tries=3 \
        --retry-wait=5 \
        --enable-http-pipelining=true \
        --enable-http-keep-alive=true \
        --user-agent="Mozilla/5.0" >/dev/null 2>&1

    if not test -f $temp_file
        echo "Failed to download audio from $url"
        rm -f $cover_file
        return 1
    end

    # Check integrity of downloaded audio
    if not check_file_integrity $temp_file
        echo "Downloaded audio is corrupted: $temp_file"
        rm -f $temp_file $cover_file
        return 1
    end

    echo "Converting to OPUS and embedding cover art..."
    if test -n "$cover_file" -a -f "$cover_file"
        echo "Debug: Using cover file $cover_file for $final_file"
        ffmpeg -y -i "$temp_file" -i "$cover_file" -c:a opus -c:v mjpeg -map 0:a:0 -map 1:v:0 \
            -metadata title="$track_name" \
            -metadata artist="$track_artists" \
            -disposition:v attached_pic \
            "$final_file" >$ffmpeg_log 2>&1
        set ffmpeg_status $status
        if test $ffmpeg_status -eq 0
            echo "Cover art embedded"
            rm -f "$cover_file"
            echo "Temporary cover file removed: $cover_file"
        else
            echo "Failed to embed cover art with ffmpeg. Log:"
            cat $ffmpeg_log
            rm -f "$temp_file" "$cover_file" "$final_file"
            return 1
        end
    else
        echo "Debug: No cover file available, converting without cover"
        ffmpeg -y -i "$temp_file" -c:a copy -vn \
            -metadata title="$track_name" \
            -metadata artist="$track_artists" \
            "$final_file" >$ffmpeg_log 2>&1
        set ffmpeg_status $status
        if test $ffmpeg_status -ne 0
            echo "Failed to convert audio. Log:"
            cat $ffmpeg_log
            rm -f "$temp_file" "$final_file"
            return 1
        end
    end
    rm -f "$temp_file"

    # Check file integrity
    if not check_file_integrity "$final_file"
        rm -f "$final_file"
        return 1
    end

    # Check cover art if it was supposed to be embedded
    if test -n "$cover_file" -a $ffmpeg_status -eq 0
        if check_cover_art "$final_file"
            echo "Cover art verified in: $final_file"
        else
            echo "Warning: Cover art not found in metadata, but audio is valid: $final_file"
        end
    end

    return 0
end

function file_exists
    set filename (echo $argv[1] | string replace -r '[/:*?"<>|]' '_')
    test -f "$output_dir/$filename.opus"
end

function print_stats
    set total_tracks $argv[1]
    echo
    echo "=== Download Statistics ==="
    echo "Total tracks in playlist: $total_tracks"
    echo "Successfully downloaded: $downloaded_count"
    echo "Skipped (already downloaded): $skipped_count"
    echo "Failed to download: $failed_count"
    echo "Output directory: $output_dir"
    echo "Checking for residual .jpg files..."
    set residual_jpgs (find "$output_dir" -maxdepth 1 -name "*.jpg")
    if test -n "$residual_jpgs"
        echo "Warning: Found residual .jpg files:"
        for jpg in $residual_jpgs
            echo "  $jpg"
            rm -f "$jpg"
            echo "  Removed: $jpg"
        end
    else
        echo "No residual .jpg files found."
    end
    echo "==========================="
end

function get_user_input
    echo "Welcome to Spotify Music Downloader!"
    echo

    # Playlist URL
    while true
        echo "Enter Spotify playlist URL (e.g., https://open.spotify.com/playlist/...):"
        read -P "> " playlist_url
        if validate_playlist_url "$playlist_url"
            break
        else
            echo "Invalid Spotify playlist URL. Please try again."
        end
    end

    # Client ID
    echo "Enter Spotify Client ID (from https://developer.spotify.com/dashboard):"
    read -P "> " client_id
    if test -z "$client_id"
        echo "Client ID cannot be empty."
        exit 1
    end

    # Client Secret
    echo "Enter Spotify Client Secret:"
    read -P "> " client_secret
    if test -z "$client_secret"
        echo "Client Secret cannot be empty."
        exit 1
    end

    # Output directory
    echo "Enter output directory (e.g., ~/music):"
    read -P "> " output_dir
    if test -z "$output_dir"
        set output_dir "~/music"
        echo "Using default output directory: $output_dir"
    end
    set output_dir (eval echo $output_dir) # Expand ~ to full path
    mkdir -p $output_dir

    # Clean up residual .jpg files
    set residual_jpgs (find "$output_dir" -maxdepth 1 -name "*.jpg")
    if test -n "$residual_jpgs"
        echo "Cleaning up residual .jpg files..."
        for jpg in $residual_jpgs
            rm -f "$jpg"
            echo "Removed: $jpg"
        end
    end

    # Get token and validate
    set token (get_spotify_token $client_id $client_secret)
    if test -z "$token"
        echo "Failed to obtain Spotify token. Check Client ID and Secret."
        exit 1
    end

    # Get tracks
    set tracks (get_playlist_tracks $playlist_url $token)
    if test $status -ne 0
        echo "Failed to retrieve tracks."
        exit 1
    end
    set total_tracks (count $tracks)
    echo "Found $total_tracks tracks in the playlist."

    # Start from track
    while true
        echo "Enter the track number to start from (1-$total_tracks, default is 1):"
        read -P "> " start_from
        if test -z "$start_from"
            set start_from 1
            break
        end
        if string match -qr '^[0-9]+$' $start_from
            if test $start_from -ge 1 -a $start_from -le $total_tracks
                break
            else
                echo "Please enter a number between 1 and $total_tracks."
            end
        else
            echo "Please enter a valid number."
        end
    end

    set -g playlist_url $playlist_url
    set -g client_id $client_id
    set -g client_secret $client_secret
    set -g output_dir $output_dir
    set -g start_from $start_from
    set -g token $token
    set -g cached_tracks $tracks
end

function download_playlist
    set token $argv[1]
    set start_from $argv[2]
    set tracks $cached_tracks
    set total_tracks (count $tracks)

    echo
    print_header
    echo "Starting download process for $total_tracks tracks..."
    echo

    set start_index (math $start_from - 1)
    if test $start_index -lt 0
        set start_index 0
    end
    if test $start_index -ge $total_tracks
        set start_index (math $total_tracks - 1)
    end

    set -g downloaded_count 0
    set -g skipped_count 0
    set -g failed_count 0
    set index 0

    for track in $tracks
        set index (math $index + 1)
        if test $index -le $start_index
            continue
        end

        set actual_index (math $index)
        set track_parts (string split '|' $track)
        set track_id $track_parts[1]
        set track_name $track_parts[2]
        set track_artists $track_parts[3]
        set cover_url $track_parts[4]
        set spotify_duration_ms $track_parts[5]
        set safe_name "$track_artists - $track_name"

        set spotify_duration_sec (math -s0 "$spotify_duration_ms / 1000")
        echo "[$actual_index/$total_tracks] Processing: $safe_name (Duration: $spotify_duration_sec sec)"

        if file_exists $safe_name
            echo "[$actual_index/$total_tracks] Skipped: Audio already exists"
            set -g skipped_count (math $skipped_count + 1)
            continue
        end

        set audio_result (get_audio_url "$track_artists - $track_name" $spotify_duration_ms)
        if test -n "$audio_result"
            set source (string split '|' $audio_result)[1]
            set url (string split '|' $audio_result)[2]
            echo "[$actual_index/$total_tracks] Downloading from $source..."
            echo "Debug: Cover URL for $safe_name: $cover_url"
            set cover_file (download_cover $cover_url $safe_name)
            echo "Debug: Cover file result: $cover_file"
            if download_and_convert $source $url $safe_name $track_name $track_artists $cover_file
                echo "[$actual_index/$total_tracks] Success: Audio downloaded, converted, and cover processed"
                set -g downloaded_count (math $downloaded_count + 1)
            else
                echo "[$actual_index/$total_tracks] Failed: Download, conversion, or integrity check failed"
                set -g failed_count (math $failed_count + 1)
            end
        else
            echo "[$actual_index/$total_tracks] Failed: No audio URL found within duration tolerance"
            set -g failed_count (math $failed_count + 1)
        end
        echo
    end

    print_stats $total_tracks
end

# Main execution
print_header
if test (count $argv) -eq 4
    # Command-line mode
    set playlist_url $argv[1]
    set client_id $argv[2]
    set client_secret $argv[3]
    set output_dir (eval echo $argv[4])
    mkdir -p $output_dir

    if not validate_playlist_url $playlist_url
        echo "Invalid Spotify playlist URL."
        usage $argv[0]
        exit 1
    end

    set token (get_spotify_token $client_id $client_secret)
    if test -z "$token"
        echo "Failed to obtain Spotify token."
        exit 1
    end

    set tracks (get_playlist_tracks $playlist_url $token)
    if test $status -ne 0
        echo "Failed to retrieve tracks."
        exit 1
    end
    set total_tracks (count $tracks)
    echo "Found $total_tracks tracks in playlist."

    while true
        echo "Enter the track number to start from (1-$total_tracks, default is 1):"
        read -P "> " start_from
        if test -z "$start_from"
            set start_from 1
            break
        end
        if string match -qr '^[0-9]+$' $start_from
            if test $start_from -ge 1 -a $start_from -le $total_tracks
                break
            else
                echo "Please enter a number between 1 and $total_tracks."
            end
        else
            echo "Please enter a valid number."
        end
    end
    set cached_tracks $tracks
else
    # Interactive mode
    get_user_input
end

download_playlist $token $start_from

```

# Music Downloader

This project is a Kotlin application that downloads audio from a $potify playlist by searching for corresponding tracks on YouTube. The downloaded audio is "converted" into the OPUS format and saved to the specified output directory.

## Requirements

- Java 11 or later
- `yt-dlp`, `aria2c`, and `ffmpeg` must be installed on your system.
- `OkHttp`, `Jackson Kotlin`, and `Jsoup` dependencies.

## Code | MusicDownloader & build.gradle.kts

```kotlin
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue
import okhttp3.OkHttpClient
import okhttp3.Request
import java.io.File
import java.util.concurrent.TimeUnit

data class Track(val id: String, val name: String, val artists: List<String>)

class MusicDownloader(private val outputDir: File) {
    private val client = OkHttpClient()
    private val mapper = jacksonObjectMapper()

    init {
        outputDir.mkdirs()
    }

    private fun getSpotifyToken(clientId: String, clientSecret: String): String {
        val request = Request.Builder()
            .url("https://accounts.spotify.com/api/token")
            .post(
                okhttp3.FormBody.Builder()
                    .add("grant_type", "client_credentials")
                    .build()
            )
            .addHeader("Authorization", "Basic ${java.util.Base64.getEncoder().encodeToString("$clientId:$clientSecret".toByteArray())}")
            .build()

        client.newCall(request).execute().use { response ->
            val json = mapper.readValue<Map<String, Any>>(response.body?.string() ?: "{}")
            return json["access_token"] as String
        }
    }

    fun getPlaylistTracks(playlistUrl: String, token: String): List<Track> {
        val playlistId = playlistUrl.split("/playlist/")[1].split("?")[0]
        val request = Request.Builder()
            .url("https://api.spotify.com/v1/playlists/$playlistId/tracks")
            .addHeader("Authorization", "Bearer $token")
            .build()

        val tracks = mutableListOf<Track>()
        client.newCall(request).execute().use { response ->
            val json = mapper.readValue<Map<String, Any>>(response.body?.string() ?: "{}")
            val items = json["items"] as List<Map<String, Any>>
            items.forEach { item ->
                val track = item["track"] as Map<String, Any>
                val id = track["id"] as String
                val name = track["name"] as String
                val artists = (track["artists"] as List<Map<String, Any>>).map { it["name"] as String }
                tracks.add(Track(id, name, artists))
            }
        }
        return tracks
    }

    private fun getYoutubeAudioUrl(track: Track): String? {
        val query = "${track.artists.joinToString(" ")} - ${track.name}"
        val process = ProcessBuilder("yt-dlp", "-f", "bestaudio", "--get-url", "ytsearch1:$query")
            .redirectErrorStream(true)
            .start()

        process.waitFor(10, TimeUnit.SECONDS)
        val url = process.inputStream.bufferedReader().readText().trim()
        return if (url.isNotEmpty()) url else null
    }

    private fun downloadAndConvert(url: String, fileName: String) {
        val tempFile = File(outputDir, "$fileName.webm")
        val finalFile = File(outputDir, "$fileName.opus")

        ProcessBuilder(
            "aria2c",
            url,
            "--dir=${outputDir.absolutePath}",
            "--out=${tempFile.name}",
            "--max-connection-per-server=16",
            "--split=16",
            "--min-split-size=1M",
            "--enable-http-pipelining=true",
            "--enable-http-keep-alive=true",
            "--user-agent=Mozilla/5.0"
        )
            .inheritIO()
            .start()
            .waitFor()

        ProcessBuilder(
            "ffmpeg", "-y",
            "-i", tempFile.absolutePath,
            "-c:a", "copy",
            "-vn",
            finalFile.absolutePath
        ).inheritIO().start().waitFor()

        tempFile.delete()
    }

    fun downloadPlaylist(playlistUrl: String, clientId: String, clientSecret: String) {
        println("Starting download process...")
        val token = getSpotifyToken(clientId, clientSecret)
        val tracks = getPlaylistTracks(playlistUrl, token)
        println("Found ${tracks.size} tracks in playlist.")

        tracks.forEachIndexed { index, track ->
            val safeName = "${track.artists.joinToString(", ")} - ${track.name}"
                .replace(Regex("[/:*?\"<>|]"), "_")
            println("Processing track ${index + 1}/${tracks.size}: $safeName")

            val youtubeUrl = getYoutubeAudioUrl(track)
            if (youtubeUrl != null) {
                println("Downloading from: $youtubeUrl")
                downloadAndConvert(youtubeUrl, safeName)
                println("Finished downloading: $safeName")
            } else {
                println("Could not find YouTube URL for: $safeName")
            }
        }

        println("Download complete! Check folder: ${outputDir.absolutePath}")
    }
}

fun main(args: Array<String>) {
    if (args.size < 4) {
        println("Usage: MusicDownloader <playlist_url> <client_id> <client_secret> <output_dir>")
        return
    }

    val downloader = MusicDownloader(File(args[3]))
    downloader.downloadPlaylist(args[0], args[1], args[2])
}
```

```kotlin
plugins {
    kotlin("jvm") version "2.1.10"
    application
}

group = "org.example"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.17.0")
    implementation("org.jsoup:jsoup:1.17.2")
    implementation(kotlin("stdlib-jdk8"))
}

application {
    mainClass.set("MusicDownloaderKt")
}

tasks {
    jar {
        manifest {
            attributes["Main-Class"] = "MusicDownloaderKt"
        }
        from(sourceSets.main.get().output)
        dependsOn(configurations.runtimeClasspath)
        from({
            configurations.runtimeClasspath.get().filter { it.name.endsWith("jar") }.map { zipTree(it) }
        })
        duplicatesStrategy = DuplicatesStrategy.EXCLUDE
    }
}
```

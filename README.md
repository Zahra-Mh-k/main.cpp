#include <string>
#include <vector>
#include <unordered_map>
#include <fstream>
#include <iostream>
#include <sstream>
#include <algorithm>
#include <functional>
using namespace std;

const int FAVORITE_GENRE_BONUS = 5;

struct Song
{
    string name;
    int id;
    int year;
    string genre;
    string artist;
};
struct User
{
    string username;
    unordered_map<int, int> music_history;
    string favorite_genre;
};
struct Playlist
{
    string name;
    string creator;
    vector<int> song_ids;
};
struct Database
{
    unordered_map<int, Song> songs;
    unordered_map<string, User> users;
    unordered_map<string, Playlist> playlists; //?
};

vector<string> split(const string &str, char delimiter)
{
    vector<string> tokens;
    string token;
    istringstream tokenStream(str);
    while (getline(tokenStream, token, delimiter))
    {
        tokens.push_back(token);
    }
    return tokens;
}
bool ascii_compare(const string &first_word, const string &second_word)
{
    size_t i = 0;
    while (i < first_word.size() && i < second_word.size())
    {
        unsigned char caracter_first = static_cast<unsigned char>(first_word[i]);
        unsigned char caracter_second = static_cast<unsigned char>(second_word[i]);
        if (caracter_first < caracter_second)
            return true;
        if (caracter_first > caracter_second)
            return false;
        ++i;
    }
    return first_word.size() < second_word.size();
}
Song parse_song(const string &line)
{
    vector<string> parts = split(line, ',');
    Song song;
    song.name = parts[0];
    song.id = stoi(parts[1]);
    song.year = stoi(parts[2]);
    song.genre = parts[3];
    song.artist = parts[4];
    return song;
}
User parse_user(const string &line)
{
    vector<string> parts = split(line, ',');
    User user;
    user.username = parts[0];
    string history = parts[1];
    user.favorite_genre = parts[2];
    vector<string> entries = split(history, ';');
    for (const string &e : entries)
    {
        if (e.empty())
            continue;
        vector<string> kv = split(e, ':');
        int song_id = stoi(kv[0]);
        int count = stoi(kv[1]);
        user.music_history[song_id] = count;
    }
    return user;
}
bool read_songs(const string &file_path, Database &database)
{
    ifstream file(file_path);
    if (!file.is_open())
    {
        return false;
    }
    string line;
    if (!getline(file, line))
    {
        return false;
    }
    while (getline(file, line))
    {
        if (line.empty())
            continue;
        vector<string> parts = split(line, ',');
        if (parts.size() < 5)
            continue;
        Song song = parse_song(line);

        database.songs[song.id] = song;
    }
    file.close();
    return true;
}

bool read_users(const string &file_path, Database &database)
{
    ifstream file(file_path);
    if (!file.is_open())
    {
        return false;
    }
    string line;
    if (!getline(file, line))
    {
        return false;
    }
    while (getline(file, line))
    {
        if (line.empty())
            continue;
        vector<string> fields = split(line, ',');
        if (fields.size() < 3)
            continue;
        User user = parse_user(line);
        database.users[user.username] = user;
    }
    file.close();
    return true;
}

long long compute_score(const Database &database, const User &user, const Song &song)
{
    long long artist_score = 0, year_score = 0, genre_score = 0;
    for (const auto &history_entry : user.music_history)
    {
        int id = history_entry.first;
        int count = history_entry.second;
        auto it = database.songs.find(id);
        if (it == database.songs.end())
            continue;
        const Song &historical_song = it->second;
        if (historical_song.artist == song.artist)
            artist_score += count;
        if (historical_song.year == song.year)
            year_score += count;
        if (historical_song.genre == song.genre)
            genre_score += count;
    }
    long long score = artist_score + year_score + genre_score;
    if (song.genre == user.favorite_genre)
        score += FAVORITE_GENRE_BONUS;
    return score;
}
struct ScoredSong
{
    long long score;
    Song song;
};
bool compare_scored_songs(const ScoredSong &a, const ScoredSong &b)
{
    if (a.score != b.score)
        return a.score > b.score;
    return ascii_compare(a.song.name, b.song.name);
}
void print_song(const Song &song)
{
    cout << song.name << " " << song.artist << " " << song.year << " " << song.genre << "\n";
}
vector<ScoredSong> find_and_score_candidates(const Database &database, const User &user)
{
    vector<ScoredSong> candidates;
    for (const auto &pair : database.songs)
    {
        const Song &song = pair.second;
        auto it_history = user.music_history.find(song.id);
        if (it_history != user.music_history.end() && it_history->second > 0)
            continue;

        long long score = compute_score(database, user, song);
        candidates.push_back({score, song});
    }
    sort(candidates.begin(), candidates.end(), compare_scored_songs);
    return candidates;
}
void print_playlist(const Playlist &playlist, const Database &database)
{
    cout << playlist.name << ":\n";
    for (int song_id : playlist.song_ids)
    {
        const Song &song = database.songs.at(song_id);
        print_song(song);
    }
}
//------------------------------------------------------------------------
void listen_song(Database &database, const vector<string> &tokens)
{
    if (tokens.size() != 3)
        return;
    const string &username = tokens[1];
    int song_id;
    song_id = stoi(tokens[2]);
    auto user_it = database.users.find(username);
    if (user_it == database.users.end())
        return;
    auto song_it = database.songs.find(song_id);
    if (song_it == database.songs.end())
        return;
    user_it->second.music_history[song_id] += 1;
    cout << username << " listened to " << song_it->second.name << "!\n";
}
void create_playlist(Database &database, const vector<string> &tokens)
{
    if (tokens.size() != 4)
        return;
    const string &playlist_name = tokens[1];
    const string &creator = tokens[2];
    int count = stoi(tokens[3]);
    auto user_it = database.users.find(creator);
    if (user_it == database.users.end())
        return;
    User &user = user_it->second;
    vector<ScoredSong> candidates = find_and_score_candidates(database, user);
    Playlist playlist;
    playlist.name = playlist_name;
    playlist.creator = creator;
    int take = min(count, static_cast<int>(candidates.size()));
    for (int i = 0; i < take; ++i)
        playlist.song_ids.push_back(candidates[i].song.id);

    string key = creator + "|" + playlist_name;
    database.playlists[key] = playlist;
    print_playlist(playlist, database);
}
void listen_playlist(Database &database, const vector<string> &tokens)
{
    if (tokens.size() != 4)
        return;
    const string &username = tokens[1];
    const string &playlist_name = tokens[2];
    const string &creator = tokens[3];

    string key = creator + "|" + playlist_name;
    auto playlist_it = database.playlists.find(key);
    if (playlist_it == database.playlists.end())
        return;
    auto user_it = database.users.find(username);
    if (user_it == database.users.end())
        return;
    for (int song_id : playlist_it->second.song_ids)
    {
        user_it->second.music_history[song_id] += 1;
    }
    cout << username << " listened to the " << playlist_name << " playlist!\n";
}
void recommend_song(Database &database, const vector<string> &tokens)
{
    if (tokens.size() != 2)
        return;
    const string &username = tokens[1];
    auto user_it = database.users.find(username);
    if (user_it == database.users.end())
        return;
    const User &user = user_it->second;
    vector<ScoredSong> candidates = find_and_score_candidates(database, user);
    cout << "Recommended song for " << username << ":\n";
    if (!candidates.empty())
    {
        print_song(candidates[0].song);
    }
}

const unordered_map<string, function<void(Database &, const vector<string> &)>> COMMANDS_MAP = {
    {"listen_song", listen_song},
    {"create_playlist", create_playlist},
    {"listen_playlist", listen_playlist},
    {"recommend_song", recommend_song}};

int main(int argc, char *argv[])
{
    if (argc != 3)
    {
        return 1;
    }
    string songs_file = argv[1];
    string users_file = argv[2];
    Database database;
    if (!read_songs(songs_file, database) || !read_users(users_file, database))
    {
        return 1;
    }
    string line;
    while (getline(cin, line))
    {
        if (line.empty())
            continue;
        vector<string> tokens = split(line, ' ');
        if (tokens.empty())
            continue;
        auto it = COMMANDS_MAP.find(tokens[0]);
        if (it != COMMANDS_MAP.end())
        {
            it->second(database, tokens);
        }
        else
        {
            continue;
        }
    }

    return 0;
}

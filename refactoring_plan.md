# Protobuf Schema Refactoring Plan

**Overall Goals:**

1.  **Standardize Package Names:** Use a consistent naming convention, like `movieque.common`, `movieque.movie`, `movieque.tvshow`, etc.
2.  **Correct Import Paths:** Ensure all `import` statements use correct relative paths from the workspace root (e.g., `"common/genre.proto"` instead of `"proto-types/common/genre.proto"`).
3.  **Logical Field Ordering:** Reorder fields within each message prioritizing `id`, `name`/`title`, key identifiers, descriptive fields, stats, and finally repeated/complex types.
4.  **Address Optionality:** Leverage proto3's default optionality for scalar types. Use `google.protobuf.Timestamp` for dates that can be null. Avoid unnecessary wrappers like `StringValue` unless distinguishing between null and empty is critical.
5.  **Resolve Redundancy & Improve Structure:**
    *   Define canonical `Episode` and `Season` messages.
    *   Refactor references in `tv-show.proto` and `season.proto` to use or import these canonical messages appropriately.
    *   Introduce specific messages where needed (e.g., for `belongs_to_collection` in `movie.proto`).
    *   Ensure consistency between similar messages like `common/cast.proto` and `common/crew.proto`.
6.  **Type Correction:** Adjust field types where the current definition doesn't match the likely TMDB structure (e.g., `belongs_to_collection`, `next_episode_to_air`).

**Detailed Plan - File by File:**

**1. Common Types (`common/`)**

*   **General:** Apply consistent field ordering (`id`, `name` first where applicable) and update package names to `movieque.common.*`.
*   `common/cast.proto`:
    *   Package: `movieque.common.cast`
    *   Order: `id`, `name`, `original_name`, `character`, `credit_id`, `order`, `adult`, `gender`, `known_for_department`, `popularity`, `profile_path`.
*   `common/crew.proto`:
    *   Package: `movieque.common.crew`
    *   Order: `id`, `name`, `original_name`, `job`, `department`, `credit_id`, `adult`, `gender`, `known_for_department`, `popularity`, `profile_path`.
*   `common/company.proto`:
    *   Package: `movieque.common.company`
    *   Order: `id`, `name`, `logo_path`, `origin_country`. (`logo_path` can be empty).
*   `common/country.proto`:
    *   Package: `movieque.common.country`
    *   Order: `iso_3166_1`, `name`.
*   `common/genre.proto`:
    *   Package: `movieque.common.genre`
    *   Order: `id`, `name`.
*   `common/language.proto`:
    *   Package: `movieque.common.language`
    *   Order: `iso_639_1`, `name`, `english_name`.
*   `common/network.proto`:
    *   Package: `movieque.common.network`
    *   Order: `id`, `name`, `logo_path`, `origin_country`, `headquarters`, `homepage`. (`logo_path`, `homepage`, `headquarters` can be empty).

**2. Person (`person.proto`)**

*   Package: `movieque.person`
*   Imports: Correct paths for `timestamp.proto`, `wrappers.proto`.
*   Order: `id`, `name`, `imdb_id`, `adult`, `gender`, `birthday`, `deathday`, `place_of_birth`, `known_for_department`, `biography`, `popularity`, `profile_path`, `homepage`, `also_known_as`.
*   Types: Keep `Timestamp` for `birthday`, `deathday`. Change `homepage` from `StringValue` to standard `string`.

**3. Episode (`episode.proto`)**

*   Package: `movieque.episode`
*   Imports: Correct paths for `timestamp.proto`, `common/crew.proto`, `common/cast.proto`. Remove incorrect `common/credit.proto` import.
*   Order: `id`, `name`, `episode_number`, `season_number`, `air_date`, `overview`, `production_code`, `runtime`, `still_path`, `vote_average`, `vote_count`, `crew`, `guest_stars`.
*   Types: Keep `Timestamp` for `air_date`. Use corrected import paths for `Crew` and `Cast`.

**4. Season (`season.proto`)**

*   Package: `movieque.season`
*   Imports: Correct paths for `timestamp.proto`, `episode.proto`. Remove unused `common/crew.proto`, `common/cast.proto`.
*   Order: `id`, `name`, `season_number`, `air_date`, `overview`, `poster_path`, `vote_average`, `_id` (keep but give high number like 99?), `episodes`.
*   Types: Keep `Timestamp` for `air_date`. Change `repeated media.season.EpisodeReference episodes` to `repeated movieque.episode.Episode episodes`.
*   Remove `EpisodeReference` message definition from this file.

**5. Movie (`movie.proto`)**

*   Package: `movieque.movie`
*   Imports: Correct paths for `common/*.proto`, `timestamp.proto`.
*   Define `CollectionInfo` message:
    ```protobuf
    message CollectionInfo {
        uint32 id = 1;
        string name = 2;
        string poster_path = 3;
        string backdrop_path = 4;
    }
    ```
*   Order: `id`, `title`, `original_title`, `imdb_id`, `status`, `release_date`, `adult`, `video`, `overview`, `tagline`, `homepage`, `poster_path`, `backdrop_path`, `original_language`, `budget`, `revenue`, `runtime`, `popularity`, `vote_average`, `vote_count`, `genres`, `production_companies`, `production_countries`, `spoken_languages`, `belongs_to_collection`.
*   Types: Keep `Timestamp` for `release_date`. Change `belongs_to_collection` from `string` to `CollectionInfo`. Mark `belongs_to_collection` as `optional` explicitly.

**6. TV Show (`tv-show.proto`)**

*   Package: `movieque.tvshow`
*   Imports: Correct paths for `common/*.proto`, `timestamp.proto`, `wrappers.proto`, `episode.proto`, `season.proto`.
*   **Rename References:**
    *   Rename `EpisodeReference` to `TvShowEpisodeReference`.
    *   Rename `SeasonReference` to `TvShowSeasonReference`.
*   **Refactor References:**
    *   Keep `TvShowEpisodeReference` fields as they are (summary).
    *   Keep `TvShowSeasonReference` fields as they are (summary).
    *   Keep `Creator` message, ensure field order (`id`, `name`, `credit_id`, `gender`, `profile_path`).
*   Order (TvShow message): `id`, `name`, `original_name`, `status`, `type`, `first_air_date`, `last_air_date`, `in_production`, `adult`, `overview`, `tagline`, `homepage`, `poster_path`, `backdrop_path`, `original_language`, `number_of_seasons`, `number_of_episodes`, `episode_run_time`, `popularity`, `vote_average`, `vote_count`, `origin_country`, `languages`, `created_by`, `networks`, `production_companies`, `production_countries`, `genres`, `spoken_languages`, `seasons`, `last_episode_to_air`, `next_episode_to_air`.
*   Types:
    *   Keep `Timestamp` for dates.
    *   Change `next_episode_to_air` from `google.protobuf.StringValue` to `TvShowEpisodeReference`. Mark as `optional`.
    *   Use `TvShowSeasonReference` for the `seasons` field.
    *   Use `TvShowEpisodeReference` for `last_episode_to_air`.

**Mermaid Diagram:**

```mermaid
graph TD
    subgraph Common
        C_Genre[movieque.common.Genre]
        C_Company[movieque.common.Company]
        C_Country[movieque.common.Country]
        C_Lang[movieque.common.Language]
        C_Network[movieque.common.Network]
        C_Cast[movieque.common.Cast]
        C_Crew[movieque.common.Crew]
    end

    subgraph Person
        P_Person[movieque.person.Person]
    end

    subgraph Movie
        M_Movie[movieque.movie.Movie]
        M_Coll[movieque.movie.CollectionInfo]
        M_Movie --> M_Coll
        M_Movie --> C_Genre
        M_Movie --> C_Company
        M_Movie --> C_Country
        M_Movie --> C_Lang
    end

    subgraph TV
        TV_Show[movieque.tvshow.TvShow]
        TV_Creator[movieque.tvshow.Creator]
        TV_EpRef[movieque.tvshow.TvShowEpisodeReference]
        TV_SeaRef[movieque.tvshow.TvShowSeasonReference]
        TV_Show --> TV_Creator
        TV_Show --> TV_EpRef
        TV_Show --> TV_SeaRef
        TV_Show --> C_Genre
        TV_Show --> C_Company
        TV_Show --> C_Country
        TV_Show --> C_Lang
        TV_Show --> C_Network
    end

    subgraph Season
        S_Season[movieque.season.Season]
    end

    subgraph Episode
        E_Episode[movieque.episode.Episode]
        E_Episode --> C_Cast
        E_Episode --> C_Crew
    end

    S_Season --> E_Episode
    TV_Show -.-> S_Season;  // Via TvShowSeasonReference import
    TV_Show -.-> E_Episode; // Via TvShowEpisodeReference import
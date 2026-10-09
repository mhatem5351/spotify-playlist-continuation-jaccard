# Spotify playlist continuation with Jaccard weighting

Experiments on automatic playlist continuation for the
[Spotify Million Playlist Dataset Challenge](https://www.aicrowd.com/challenges/spotify-million-playlist-dataset-challenge):
given a playlist title and some seed tracks, recommend 500 more tracks.

## Attribution

Most of this code is not mine. It is based on the thesis code of **Sotiris Podaras**
([@sotiris93](https://github.com/sotiris93)), written for his postgraduate thesis on the same challenge. His later,
cleaned-up version is published at
[sotiris93/Automatic_Song_Playlist_Creation-Completion--Postgraduate_Thesis](https://github.com/sotiris93/Automatic_Song_Playlist_Creation-Completion--Postgraduate_Thesis).
`DIPLOMATIKI 2B_Jaccard.ipynb` ("diplomatiki" is Greek for thesis) is an edited copy of the thesis notebook
`DIPLOMATIKI2Β.ipynb`, and the helper scripts in `codes/` come from the same project.

Parts of that base code (the helper scripts with `/home/irlab/...` paths and the FastText + Surprise SVD pipeline)
look like they were adapted from an earlier RecSys Challenge 2018 implementation. I could not identify its original
author. `verify submission.ipynb` is the challenge's `verify_submission.py` format checker, run in a notebook.

I used this repository to study and run his approach while learning recommender systems; the method and the code are
his, and I don't claim either. If you are one of the original authors and want the credit worded differently, please
open an issue.

## How it works

1. `codes/gererate data file.py` writes every MPD playlist title (lower-cased) to `data.txt`. The notebook trains a
   FastText model on these titles (100 epochs).
2. `codes/personal_stats.py` counts how often each track, artist and album appears in the MPD (`*_stats.json`).
   `codes/playlist_file_map.py` maps each normalized title to the MPD slice file that contains it
   (`track_filename.json`).
3. For each challenge playlist with a title, the notebook takes the 20 nearest titles from FastText, loads the MPD
   playlists with those titles and scores their tracks.
4. The scores form a playlist × track rating matrix. A Surprise SVD model is trained on it (k-fold, up to 10 folds)
   and predicts a score for each candidate track for the challenge playlist. The candidates are ranked by that
   score after removing seed tracks (see Known issues).
5. If fewer than 500 tracks are left, the list is filled with the most frequent MPD tracks. Each row
   (`pid` + 500 track URIs) is appended to `final_submission.csv`.

Playlists without a title skip step 3 (that call is commented out), so their list is almost entirely popular tracks.

## Scoring

The notebook contains two scoring variants. The earlier one, still there as commented-out lines, gives every track
of a similar playlist a popularity weight:

```
score = (artist share + album share + track share) / (1 + position in playlist)
```

where each share is the item's count divided by the total count from `personal_stats.py`. The active variant uses
a weighted Jaccard similarity between a similar playlist and the reference playlist:

```
score = (1 + 0.5 * J(tracks) + 0.25 * J(artists) + 0.25 * J(albums)) / (1 + J(title))
```

`J` is the Jaccard similarity of the two sets of track, artist or album URIs. `J(title)` compares the two normalized
titles as single items, so it is 1 when the titles are identical and 0 otherwise. In `create_weighted_playlists` the
reference is the MPD playlist(s) that carry the challenge playlist's title.

The thesis code linked above uses the same weighted-Jaccard score (same weights), but applies it directly to rank
all MPD playlists, without FastText or SVD.

## Data

The Spotify Million Playlist Dataset (1,000,000 playlists created by Spotify users between 2010 and 2017) and the
10,000-playlist challenge set were released by Spotify for the
[ACM RecSys Challenge 2018](https://recsys.acm.org/recsys18/challenge/) and are now hosted as an open challenge on
AIcrowd. Neither is included here. Download them from the
[AIcrowd dataset page](https://www.aicrowd.com/challenges/spotify-million-playlist-dataset-challenge/dataset_files)
after accepting the terms.

## How to run

There is no requirements file. The notebook was run on Python 3.9 with numpy, pandas, scikit-learn, gensim,
scikit-surprise, tqdm and ipywidgets.

1. Put the MPD slice files in `spotify_million_playlist_dataset/data/` and `challenge_set.json` in
   `spotify_million_playlist_dataset_challenge/`, next to the notebook.
2. Edit the hard-coded paths in the three scripts in `codes/` and run them to create `data.txt`,
   `artist_stats.json`, `album_stats.json`, `tracks_stats.json` and `track_filename.json`. Put these files in an
   `arxeia/` folder (Greek for "files") next to the notebook. `playlist_file_map.py` is a fragment: it expects
   `os`, `json`, `path`, `filenames`, `mapping` and `normalize_name` to be defined (as in the notebook's first
   cells), and on Python 3 its output file must be opened with `'w'` instead of `'wb'`.
3. Run `DIPLOMATIKI 2B_Jaccard.ipynb` from top to bottom. It appends rows to `final_submission.csv`; add the
   `team_info` header line the challenge expects yourself.
4. Run `verify submission.ipynb` to check the file (edit the two paths in its last lines first).

The notebook uses Windows-style relative paths (`arxeia\data.txt`); change the separators on macOS or Linux.

## Results

There is no evaluation here: the challenge metrics (R-precision, NDCG, recommended-songs clicks) were never computed.

- The recorded run in the notebook stopped after 541 of the 10,000 challenge playlists. For the 540 finished
  playlists the model produced 12 to 152 candidate tracks (median 36), so most of each 500-track list came from the
  popular-track fallback.
- The per-fold RMSE printed by Surprise (median 0.018) only shows how well SVD fits the scores it was trained on.
  It does not measure recommendation quality.
- `verify submission.ipynb` reports a submission file with 2,000 of the 10,000 playlists, so no complete submission
  was produced.

## Known issues

- In the Jaccard version the score line sits after the loop over a playlist's tracks, so each matching playlist adds
  only its last track to the rating matrix. This keeps the candidate lists short.
- The main loop keeps only the last seed track in `seed_tracks`, so the other seed tracks are not filtered out of the
  recommendations.
- The main loop scores each challenge playlist against the tracks of the "Throwbacks" test playlist (set before the
  loop), not against reference playlists of its own.
- Paths are hard-coded, and the scripts in `codes/` still contain absolute paths from the machines they were
  written on.

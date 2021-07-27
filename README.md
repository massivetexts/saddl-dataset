# saddl-dataset

This page describes the access, use, and data structure of the Similarities and Duplicates in Digital Libraries (SaDDL) project.

The SaDDL project used content-based methods to learn duplicate work and partial work relationships

## Before you download: Easy access alternative

You can browse the relationship inferences, as well as book recommendations and work clusters, online at https://saddl.du.edu.

E.g. https://saddl.du.edu/relationships/htid/hvd.hn3jbk

A JSON endpoint for the dataset is also available; e.g. https://saddl.du.edu/relationships/htid/hvd.hn3jbk.json

## Downloading the Dataset

The dataset is hosted on Google Compute Engine. To Download, install [gsutil](https://cloud.google.com/storage/docs/gsutil_install) and run the following command:

```gsutil cp -r gs://saddl-data/dataset/0-0-1-prerelease /path/on/your/computer```

### Current version

The current dataset is the _v.0.0.1-prelease_. The data format may be updated during pre-release.

### Using the Data

The dataset is comprised of multiple tables, presented in [Apache Parquet](https://parquet.apache.org/) format. Parquet is a fast columnar storage format that is supported by many data analysis tools, such as Pandas and Dask in Python. For SQL access to parquet files, we recommend DuckDB, a portable database tool that can import Parquet as well as natively perform SQL queries over it. A DuckDB import script is included in the dataset export.

The tables included in the dataset are:

#### `clean_predictions`

The `clean_predictions` table has the softmax predictions from the SaDDL relationship classifier. Each row specifies a target/candidate pair of books, and probabilities for membership in a set of 10 classes:

- `swsm`: Same-work same-manifestation, or the likelihood that the target/candidate pair looks identical.
- `swde`: Same-work different-expression, which suggests different printings, editions, versions, etc. of the same work. (e.g. different copies of *Hamlet*).
- `wp_dv`: Different volume. This class suggests a different volume from the same set; e.g., if the target is volume 1 of *Pride and Prejudice* and the candidate is volume 2.
- `partof`/`contains`: Is the target contained by, or does it contain, the candidate book; e.g. in an anthology of works versus a printing of just the given text.
    - Tip: Sometimes you see a volume of an anthology printing just one story, in which case the expected relationship is `swde`. This can be a good way to tease out what's actually in, say, v.14 of *The Works of Charles Dickens*.
- `overlaps`: Do the two books overlaps, in that a part of the target is also a part of the candidate? This class was trained from fully artificial data, as cataloguing metadata typically does not provide enough information to find and train this relationship on real-world data.
- `diff` classes (`author`, `simdiff`, `grsim`, `randdiff`): These classes all suggest that the target and candidate are not the same work. We recommend adding them together.
- `relatedness`: A metric used for book recommendation.

The `clean_predictions` table actually takes a mean from the classifier where the classifer saw both books reciprocally - that is, it evaluated both book1 to book2 *and* book2 to book1. This makes the dataset more succinct: as of July 2021, `clean_predictions` has `17925478` judgments where the original `predictions` table held `158449603`.

```SELECT * FROM clean_predictions USING SAMPLE 1```

|    | target             | candidate    |        swsm |     swde |      wp_dv |     partof |    contains |    OVERLAPS |      author |     simdiff |       grsim |    randdiff |   relatedness |   count |
|---:|:-------------------|:-------------|------------:|---------:|-----------:|-----------:|------------:|------------:|------------:|------------:|------------:|------------:|--------------:|--------:|
|  0 | mdp.39015072111084 | uc1.b3637302 | 7.76776e-05 | 0.992764 | 0.00715149 | 7.2751e-07 | 3.81831e-07 | 1.96945e-07 | 3.33364e-06 | 1.76866e-06 | 2.71564e-07 | 1.16576e-11 |      0.140004 |       2 |

#### `meta`

This is a slightly cleaned dataset of metadata derived from the [Hathifiles](https://www.hathitrust.org/hathifiles). One additional field added is `page_count`.

```SELECT * FROM meta USING SAMPLE 1```
|    | htid                | access   | rights   |   ht_bib_key |   description | source   |   source_bib_num |   oclc_num |   isbn |   issn |   lccn | title                                                              | imprint       | rights_reason_code   | rights_timestamp    |   us_gov_doc_flag |   rights_date_used | pub_place   | lang   | bib_fmt   | collection_code   | content_provider_code   | responsible_entity_code   | digitization_agent_code   | access_profile_code   | author                   |   page_count |
|---:|:--------------------|:---------|:---------|-------------:|--------------:|:---------|-----------------:|-----------:|-------:|-------:|-------:|:-------------------------------------------------------------------|:--------------|:---------------------|:--------------------|------------------:|-------------------:|:------------|:-------|:----------|:------------------|:------------------------|:--------------------------|:--------------------------|:----------------------|:-------------------------|-------------:|
|  0 | umn.31951002077812h | allow    | pd       |     11988159 |           nan | UMN      |        000889539 |   37236219 |    nan |    nan |    nan | Slings and arrows, and other tales / by Hugh Conway (F. J. Fargus) | [s.n.], 1885. | bib                  | 2013-04-18 19:25:55 |                 0 |               1885 | nyu         | eng    | BK        | UMN               | umn                     | umn                       | google                    | google                | Conway, Hugh, 1847-1885. |          388 |

#### `clusters`

The `clusters` table puts each book into a group of exact duplicates (*manifestations*) or a larger group with all the iterations of that book's content (*works*).

```SELECT * FROM clusters USING SAMPLE 1```

|    | htid               |   work_id |   man_id |
|---:|:-------------------|----------:|---------:|
|  0 | mdp.39015040731187 |    747590 |   820031 |


#### `work_predictions` and `manifestation_predictions`

These tables are functionally the same as the `clean_predictions` table, but providing relationships between *works* or *manifestations*. The `target` and `candidate` columns refer to work or manifestations ids, as resolved to individual volumes in the `clusters` table.

Each target<->candidate row is an average of *all* inferences between books in the target cluster and books in the candidate cluster.

#### `work_stats` and `manifestation_stats`

Details about work and manifestation clusters. Each record represents a single work or manifestation. Fields include:

- `label_count`: Number of members in that group.
- `gov_count`, `gov_prop`, `serial_count`, `serial_prop`: Counts and proportions of serials and government records in a work or manifestation. SaDDL focused on books but still processed serials and gov't docs. We recommend ignoring works where `gov_prop > .5 or serial_prop>.5`.
- `best_centroid`, `best_centroid_pd`: A recommendation for the preferred copy of a work or manifestation. The `_pd` recommendation is for the preferred book that is *full-text accessible* at the [HathiTrust Digital Library](https://www.hathitrust.org/) - this is useful for the clusters of duplicate works where only some copies are confirmed to be in the public domain. An alternate method for recommending the preferred copy is `best_median` and `best_median_pd`, which takes the median point of a group of books in a linear vector space. An additional PageRank-based method has not been crunched for the full dataset but can be extracted from this dataset with [provided code](https://github.com/massivetexts/compare-tools/blob/master/scripts/BestCopySelection.ipynb).

```SELECT * FROM work_stats WHERE label_count > 10 LIMIT 1```

|    |   work_id |   label_count |   gov_count |   serial_count |   gov_prop |   serial_prop | include   | best_centroid   | best_centroid_pd   | best_median   | best_median_pd   |
|---:|----------:|--------------:|------------:|---------------:|-----------:|--------------:|:----------|:----------------|:-------------------|:--------------|:-----------------|
|  0 |     27730 |            12 |           0 |              0 |          0 |             0 | True      | uc1.b3114093    | uc1.b3114093       | uc1.b3114093  | uc1.b3114093     |

# AffiniGuider

AffiniGuider is an in-memory filtered approximate nearest-neighbor index. It builds a vanilla graph, then uses attribute correlation to expand the query filter at search time. The implementation is based on the legacy C++ code in the [`cpp_main`](https://github.com/microsoft/DiskANN/tree/cpp_main) branch of [microsoft/DiskANN](https://github.com/microsoft/DiskANN), under the MIT license in `LICENSE.txt`.

Two ways to rank related attributes are supported:

- `0`: one-hop co-occurrence on the graph (topology)
- `1`: distance between attribute centroids

At query time, `--expand_labels_k` adds the top correlated attributes to the query filter. The search itself remains a filtered graph search.

## Build

```bash
sudo apt install make cmake g++ libaio-dev libgoogle-perftools-dev libboost-all-dev libmkl-full-dev
mkdir build && cd build && cmake -DCMAKE_BUILD_TYPE=Release .. && make -j
```

Binaries used below:

- `build/apps/build_memory_index`
- `build/apps/test_or`
- `build/apps/utils/compute_groundtruth_for_multi_filters`

## Data format

- Vectors are DiskANN `.bin` files: `uint32` count, `uint32` dimension, then row-major vectors.
- `label_file` is text. Line `i` lists the attributes of base point `i`, separated by commas.
- `query_filters_file` is text. Line `i` lists the attributes of query `i`, separated by commas.

## Build an index

`--post_build_label_processing` builds an ordinary graph first, then attaches labels and computes attribute correlation. It does not run filtered graph construction.

Topology correlation:

```bash
build/apps/build_memory_index \
  --data_type float --dist_fn l2 \
  --data_path base.bin \
  --index_path_prefix index/affiniguider \
  --label_file labels.txt \
  --label_type uint \
  --max_degree 64 --Lbuild 100 --alpha 1.2 \
  --post_build_label_processing \
  --label_correlation_method 0 \
  --expand_labels_k 4
```

Centroid correlation is the same command with `--label_correlation_method 1`.

`--expand_labels_k` stored at build time is the default expansion width. Search can override it.

## Search

```bash
build/apps/test_or \
  --data_type float --dist_fn l2 \
  --index_path_prefix index/affiniguider \
  --query_file query.bin \
  --query_filters_file query_labels.txt \
  --gt_file gt.bin \
  --recall_at 10 \
  --search_list 10 20 40 80 \
  --expand_labels_k 4 \
  --result_path results/run
```

Useful search flags:

- `--expand_labels_k`: how many correlated attributes to add. `0` searches with the original query attributes only.
- `--disable_probe`: do not adjust the expansion width with the selectivity probe.
- `--fixed_expand_k`: still run the probe, but keep the expanded attribute count fixed.
- `--label_type uint`: attribute id type. Use the same value as at build time.

The result log reports QPS, distance comparisons, and Recall@K for each search-list size.

## Ground truth

```bash
build/apps/utils/compute_groundtruth_for_multi_filters \
  --data_type float --dist_fn l2 \
  --base_file base.bin \
  --query_file query.bin \
  --label_file labels.txt \
  --query_filters_file query_labels.txt \
  --gt_file gt.bin \
  -K 10
```

Pass the same `--gt_file` to `test_or` to print recall.
